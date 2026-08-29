# securarr

A security-hardened Docker Compose media stack — Plex, the \*arr apps,
download clients behind a VPN, and Seerr for requests — built to live in
git: secrets encrypted with SOPS, images pinned, and an optional
Flatcar/Butane layer that turns `git push` into a deploy.

What "hardened" means here, concretely:

- **No port forward to anything but Plex.** Every other app is reached
  through Caddy on the LAN, or through a Cloudflare tunnel with Access
  policies in front of it remotely. A source-address guard in the
  Caddyfile backstops the router: even if a forward appeared, no
  hostname would answer it.
- **Torrent traffic cannot leak.** qBittorrent runs inside the VPN
  container's network namespace — if the VPN dies, it has no route out
  at all.
- **Plex is assumed compromised eventually.** It is the one
  internet-facing service, so it gets its own stack (ideally its own
  host and VLAN) and a read-only view of the library. A total compromise
  cannot encrypt or delete your media.
- **Every container drops all capabilities** and adds back only what it
  demonstrably needs, with `no-new-privileges` across the board.
- **Secrets never touch git in plaintext.** SOPS + age, split per trust
  domain — the VPN key reaches exactly one container.

## Layout

```
compose.yaml            the arr host — 13 services
compose.plex.yaml       the Plex host — 1 service
services/
  proxy.yaml            caddy + cloudflared — TLS, routing, ZTNA
  download.yaml         gluetun, qbittorrent, sabnzbd, unpackerr
  arr.yaml              prowlarr, sonarr, radarr, bazarr, recyclarr
  media.yaml            seerr, tautulli
  plex.yaml             plex, deployed alone in its own stack
hosts/                  hardware profiles (GPU wiring), chosen in .env
caddy/Caddyfile         one block per app, routed by subdomain
recyclarr/recyclarr.yml TRaSH quality profiles — the one app config in git
host/butane-*.yaml      optional: Flatcar host provisioning (Ignition source)
.env                    non-secret config — committed, edit to taste
secrets/*.env.example   templates for the four secret files
secrets/*.enc.env       SOPS-encrypted secrets — committed (yours, not ours)
scripts/secrets         encrypt / decrypt / edit helper
docs/                   secrets workflow, Flatcar provisioning
.github/workflows/      CI — validate, scan (see below)
renovate.json5          update policy — grouped PRs, no automerge
trivy-secret.yaml       custom leak-scan rules for this stack's secrets
.trivyignore.yaml       risk-acceptance ledger for the image scan
```

The service split follows the pipeline rather than file size, and
everything under `services/` is deliberately hardware-agnostic — GPU
wiring lives only in `hosts/`, chosen by one line in `.env`, so the
stack can move between machines untouched.

## Prerequisites

- A Docker host (any Linux with Docker + Compose v2; the Flatcar layer
  is optional)
- A domain on [Cloudflare](https://www.cloudflare.com/) (free plan is
  fine) — for TLS via DNS-01, the tunnel, and Access
- A WireGuard-capable VPN subscription for torrent traffic
  (anything [gluetun supports](https://github.com/qdm12/gluetun-wiki))
- `sops` and `age` on your workstation (`brew install sops age`)

## Quick start

```bash
git clone https://github.com/YOURUSER/securarr.git && cd securarr
```

**1. Configure.** Edit `.env`: your domain, timezone, media paths. Leave
`BIND_ADDRESS=127.0.0.1` — apps are meant to be reached through the
proxy.

**2. Set up secrets** (full walkthrough in
[docs/secrets.md](docs/secrets.md)):

```bash
age-keygen -o ~/.config/sops/age/keys.txt    # macOS: ~/Library/Application Support/sops/age/keys.txt
# put the printed public key into .sops.yaml
for f in secrets/*.env.example; do cp "$f" "${f%.example}"; done
$EDITOR secrets/*.env                        # each template says where its value comes from
./scripts/secrets encrypt
```

**3. DNS + tunnel.** In Cloudflare: a wildcard `*.<DOMAIN>` record in
your home DNS pointing at the Docker host; a tunnel with one wildcard
public hostname handing everything to `caddy:443` (the comments in
`secrets/cloudflared.env.example` walk through it); Access policies per
hostname you expose remotely.

**4. Deploy:**

```bash
./scripts/secrets decrypt
docker compose up -d                          # the arr stack
docker compose -f compose.plex.yaml up -d     # the Plex stack (same or separate host)
```

**5. Wire the apps together** — in each app's UI, reference other
services by container name, never by IP: qBittorrent is
`http://gluetun:8081` (it lives in the VPN's namespace), SABnzbd
`http://sabnzbd:8080`, Sonarr `http://sonarr:8989`, and so on. Names
survive host moves; addresses don't.

Validate without deploying:

```bash
docker compose config --quiet
./scripts/secrets check
```

## The two-stack design

`compose.yaml` and `compose.plex.yaml` are separate on purpose, and the
separation is the security model: Plex answers the internet, so it
belongs on its own host (or VM) on its own network segment, holding no
secrets and mounting the library read-only. Seerr, Tautulli and Caddy
reach it at `PLEX_HOST`; your firewall should permit that direction and
deny the reverse.

Nothing stops you running both stacks on one machine while you get
started — set `PLEX_HOST=host.docker.internal` and split later. The
stack neither knows nor cares.

## GitOps hosts (optional)

`host/` contains Flatcar Butane profiles that provision each host from
scratch: mounts, networking, a systemd timer that pulls this repo every
ten minutes and reconciles with `docker compose up -d`. Pushing to
`main` becomes the deploy mechanism. See
[docs/flatcar.md](docs/flatcar.md).

If you adopt it, protect `main` accordingly: require review, or at
minimum let CI gate merges — with a pull-based deploy loop, CI is the
only thing standing between a bad merge and production.

## CI and updates

Two workflows ship with the repo:

- **validate** — every push and PR: lints and audits the workflows
  themselves (actionlint, zizmor), parses both compose stacks, and
  transpiles both Butane profiles with `--strict`.
- **scan** — every push and PR, plus weekly: Trivy scans the tree for
  leaked secrets (`trivy-secret.yaml` adds WireGuard/\*arr/Plex key
  formats the builtins miss) and scans every image the compose files
  resolve to. It fails only on fixable CRITICALs not accepted in
  `.trivyignore.yaml` — a risk ledger where every entry needs a
  statement and an expiry date, so acceptances are re-judged rather
  than forgotten.

Image and action updates come from
[Renovate](https://docs.renovatebot.com/), configured by
`renovate.json5`: install the hosted
[Renovate app](https://github.com/apps/renovate) on your fork and it
just works — its first run pins image digests on top of the version
tags, making deploys fully deterministic (the repo ships tag-only so
forks without Renovate never carry stale digests). Prefer keeping the
runs inside your own Actions? [docs/renovate.md](docs/renovate.md)
covers self-hosting with your own GitHub App.

Nothing automerges. Every image change is a visible event; the boring
ones are grouped so reviewing them is cheap.

## Notable defaults you may want to change

- Usenet (`sabnzbd`) is deliberately *not* behind the VPN: it is TLS to
  a paid provider, and tunnelling it costs throughput. Route it through
  gluetun if your threat model says otherwise.
- `recyclarr/recyclarr.yml` syncs TRaSH quality profiles into Sonarr and
  Radarr on a schedule; the profile choices in it are opinions, not
  requirements.

## License

[MIT](LICENSE).
