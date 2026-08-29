# Secrets

Encrypted with [SOPS](https://github.com/getsops/sops) against an
[age](https://github.com/FiloSottile/age) key. Only the `.enc.env` files
are committed; the decrypted `.env` files exist at runtime and are
gitignored.

## Files

| | |
|---|---|
| `secrets/*.env.example` | Templates. Committed. Start here. |
| `secrets/*.enc.env` | Encrypted. Committed. |
| `secrets/*.env` | Decrypted. Gitignored. Created at deploy time. |
| `.sops.yaml` | Which keys can decrypt what |
| `scripts/secrets` | The helper for all of it |

## First-time setup

```bash
brew install sops age        # or your distro's packages

# Generate a key into the platform config dir (path table below)
age-keygen -o ~/Library/Application\ Support/sops/age/keys.txt   # macOS
# age-keygen -o ~/.config/sops/age/keys.txt                      # Linux

# Put the printed PUBLIC key into .sops.yaml, replacing the placeholder.

# Fill in the templates, then encrypt
for f in secrets/*.env.example; do cp "$f" "${f%.example}"; done
$EDITOR secrets/*.env
./scripts/secrets encrypt
./scripts/secrets clean      # remove the plaintext copies
```

## Commands

```bash
./scripts/secrets decrypt        # before docker compose up
./scripts/secrets encrypt        # after editing a plaintext file
./scripts/secrets edit vpn       # edit in place, never written in clear
./scripts/secrets check          # confirm every file still opens
./scripts/secrets clean          # remove the decrypted copies
```

## Split by trust domain

`env_file` injects *everything* in a file into a container, so files are
split by who may legitimately see what.

| File | Goes to | Contains |
|---|---|---|
| `vpn` | gluetun | WireGuard keys and addresses |
| `arr-keys` | sonarr, radarr, recyclarr, unpackerr | Sonarr and Radarr API keys |
| `caddy` | caddy | Cloudflare API token, DNS-01 only |
| `cloudflared` | cloudflared | Tunnel connector token |

The line is drawn at trust domain rather than strict per-service
minimalism. The four consumers of `arr-keys` are the same domain — same
network, same operator, and two of them are the apps that own the keys.
`gluetun` is a different domain, so the VPN key stays in its own file and
never reaches anything else.

Adding one means writing `secrets/<name>.env`, adding `<name>` to the
`SECRETS` array in `scripts/secrets`, encrypting, and referencing it with
`env_file` on the services that need it.

## The \*arr API keys are inputs, not outputs

This is the part worth understanding before a rebuild.

Sonarr and Radarr generate an API key on first run and store it in
`config.xml`. If that were the source of truth, deploying onto empty
config directories would invent new keys — and recyclarr and unpackerr,
carrying the old values, would fail to authenticate against apps that had
silently moved on.

So the direction is inverted. `arr-keys.env` *sets* the key on each app
using the Servarr environment override:

```
SONARR__AUTH__APIKEY=…
RADARR__AUTH__APIKEY=…
```

The same file supplies the same values to recyclarr (via `!env_var` in
its config) and unpackerr. One source of truth, four consumers, no drift
possible — and a fresh deploy comes up already wired.

The duplicated values under different variable names are deliberate: each
tool expects its own spelling, and keeping them together makes rotation a
single edit.

### Rotating an \*arr key

1. `./scripts/secrets edit arr-keys`
2. Change the value everywhere it appears — three variables per app.
3. `docker compose up -d sonarr radarr recyclarr unpackerr`

The apps adopt the new key on start. Nothing needs touching in their UIs.

Prowlarr also generates its own key, but nothing in this stack consumes
it, so it is left alone. The same pattern applies if that changes.

## The age key

**Not in this repo, and not recoverable if lost** — losing it means every
`.enc.env` is unreadable and every secret must be regenerated from
source.

SOPS looks in the platform config directory, which differs by OS:

| | |
|---|---|
| macOS | `~/Library/Application Support/sops/age/keys.txt` |
| Linux | `~/.config/sops/age/keys.txt` |

This trips people up: putting the key at `~/.config` on a Mac produces a
confusing "identity did not match any of the recipients" error even
though the key is right there.

Back it up somewhere that is not this machine. A password manager entry
is fine; it is a single short line.

## Adding a recipient

Each machine that must decrypt — the deploy host, a second workstation —
needs its own age key, with its public key added as a recipient.

On the new machine:

```bash
age-keygen -o <platform config dir>/sops/age/keys.txt
chmod 600 <that file>
```

Then add the printed public key to `.sops.yaml`:

```yaml
creation_rules:
  - path_regex: secrets/.*\.env$
    age: >-
      age1existing...,
      age1new...
```

Adding a recipient does **not** re-encrypt existing files. From a machine
that can already decrypt:

```bash
./scripts/secrets decrypt
./scripts/secrets encrypt
```

Then verify from the new machine with `./scripts/secrets check`.

## What is in git, and what that implies

Encrypted secrets in a git repository are safe in the sense that the
ciphertext is useless without the key. They are not safe in the sense of
being revocable — history is forever, and a key that leaks later exposes
every historical version.

Two consequences:

**Rotate anything that was ever committed in plaintext**, even briefly.
Encrypted-after-the-fact is not encrypted.

**Never commit a plaintext secret "just for a moment."** A single commit
puts it in history permanently.

Verify before committing:

```bash
git add -An . | sed "s/^add '//;s/'$//" | xargs grep -l 'PRIVATE_KEY\|API_KEY' 2>/dev/null
```

Anything returned that is not a `.enc.env` or an `.example` is a problem.
