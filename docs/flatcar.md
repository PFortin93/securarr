# Flatcar provisioning

The optional GitOps layer: both hosts run
[Flatcar Container Linux](https://www.flatcar.org/), provisioned by
Ignition from the Butane sources in `host/`. Skip this entirely if you
just want the stack on an existing Docker host — `compose.yaml` neither
knows nor cares.

## The model

Ignition runs **once, at first boot** — it is not configuration
management. Changing a Butane file describes the *next* provisioning,
not the running node. Day-to-day convergence is the sync loop instead:
`securarr-sync.timer` fires `securarr.service` every 10 minutes, which
pulls this repo (read-only deploy key), decrypts secrets where
`NEEDS_SECRETS=yes`, runs `docker compose up -d`, and prunes dangling
images. Pushing to `main` changes the hosts within ten minutes.

Both profiles are idempotent (`overwrite: true` on every file/link), so
re-provisioning an existing host is supported.

## What the profiles provide

Flatcar ships Docker and containerd as bundled sysexts, enabled by
default. On top of that the profiles declare:

- **Sysexts**: the Compose plugin on both; NVIDIA driver + runtime on
  the Plex host only. Downloading a sysext does nothing by itself — the
  `/etc/extensions/*.raw` symlinks activate them.
- **`/var/lib/securarr/bin/`**: `sops` (vendored static binary — no
  package manager exists) and the `securarr-sync` script. The sync
  unit, its timer, and the reboot-window dropin for `locksmithd`
  (04:00–06:00, so OS auto-updates never reboot mid-stream).
- **Mounts**: NFS media (`ro` on Plex), the config disk at
  `/var/lib/securarr`, scratch at `/mnt/scratch` — all gated into
  `securarr.service` via `RequiresMountsFor`, so the stack cannot start
  against missing storage.
- **Static networking** matched on **MAC address**, never interface
  name (virtio enumeration order is not stable).
- **tmpfiles.d** entries creating scratch subdirectories as uid 1000 —
  Docker auto-creates missing bind sources as root, which uid-1000
  containers then cannot write.
- **`/etc/securarr/`**: pinned GitHub host keys and `env`
  (`STACK_REPO`, `COMPOSE_FILE`, `NEEDS_SECRETS`).

**Not in Ignition, placed out of band after first boot** (the configs
are committed; these must never be): `/etc/securarr/deploy.key` (a
read-only deploy key for your fork) and — on hosts with
`NEEDS_SECRETS=yes` — `/etc/securarr/age.key`. The sync script refuses
to run without them rather than starting half-configured.

## Sharp edges (encoded in the configs; listed so nobody re-learns them)

- **`/opt` is a trap**: systemd-sysext mounts a tmpfs over it when any
  merged extension ships `/opt` content (the NVIDIA runtime does),
  shadowing whatever Ignition wrote. All stateful paths live under
  `/var/lib`.
- **A guest reboot is not a power cycle.** QEMU reads device lists and
  the fw_cfg blob at process launch only; delivering a new Ignition or
  attached disk requires a full VM stop and start from the hypervisor.
- **Timers need explicit `Unit=`** — without it a timer fires its
  name-twin service, which may not exist, and dies silently.
- **GPU driver flavour**: Pascal cards need the proprietary NVIDIA
  sysext; the `-open` modules are Turing+. The wrong one fails silently
  into CPU transcoding.

## Provisioning a host

1. Edit the Butane profile (search for `REPLACE`), then transpile:

   ```bash
   docker run --rm -i quay.io/coreos/butane:release --strict \
     < host/butane-plex.yaml > ignition.json
   ```

   CI validates both profiles on every push as well.

2. Create the VM (or bare-metal boot medium) with the MAC addresses the
   profile declares, attach the config and scratch disks (formatted and
   labelled per the mount units: `mkfs.ext4 -L plex-config`, etc.), and
   write the Flatcar image to the boot disk:

   ```bash
   curl -sL https://stable.release.flatcar-linux.net/amd64-usr/current/flatcar_production_image.bin.bz2 \
     | bunzip2 | sudo dd of=/dev/<boot-disk> bs=4M conv=fsync status=progress
   ```

3. Hand the Ignition JSON to the machine. On QEMU/KVM that is:
   `-fw_cfg name=opt/org.flatcar-linux/config,file=/path/to/ignition.json`
   — other hypervisors have their own delivery mechanism; see the
   [Flatcar docs](https://www.flatcar.org/docs/latest/installing/) for
   yours.

4. Boot, then place the out-of-band keys in `/etc/securarr/` and run the
   checklist below. To re-provision an existing node instead:
   `sudo touch /boot/flatcar/first_boot`, then a full power cycle.

Keep any staged Ignition file on the hypervisor current with the repo —
a stale copy re-provisions old bugs.

## Verify after any (re)provision

In order of how quietly each fails:

- **Hardware transcode** (Plex host): a session must show `(hw)`. The
  device node existing proves nothing.
- `systemd-sysext status` lists everything expected; `docker compose
  version` works.
- `findmnt /var/lib/securarr /mnt/media /mnt/scratch` — and on Plex,
  `touch /mnt/media/.probe` must fail (read-only at two layers).
- `systemctl is-active securarr-sync.timer` — **the timer, not the
  service.** The service being inactive between runs is normal.
- Arr host: `docker compose exec gluetun wget -qO- https://ipinfo.io/ip`
  returns your VPN provider's address, never yours.
