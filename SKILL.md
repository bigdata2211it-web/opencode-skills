---
name: mybd-cryfs
description: >
  Set up and operate an encrypted credentials vault with cryfs (FUSE) on any OS.
  Covers installation across Linux distros (Debian/Ubuntu, Fedora, Arch, NixOS,
  Alpine), macOS (macFUSE), and Windows (WinFsp + cryfs). Includes migration of
  an existing plaintext folder into cryfs, daily open/close workflow, auto-lock
  via --unmount-idle, agent-safe usage patterns, backup, password change, and
  troubleshooting. Use when user says "encrypt my credentials", "cryfs vault",
  "mybd", "protect passwords folder", "fuse encryption setup", or asks how to
  keep a secrets directory encrypted-at-rest while usable on demand. Also use
  when an AI agent needs to follow safe open/close discipline for a cryfs-mounted
  secrets store.
---

# mybd-cryfs — encrypted credentials vault with cryfs

## Goal

Keep a folder of secrets (logins/passwords/TSV/JSON) encrypted at rest, usable
on demand after entering a password, and auto-locked when idle. Works for a
human and for an AI agent working under the human's explicit permission.

## Threat model

| Threat | Covered |
|--------|---------|
| Another user/process reads plaintext files | Yes — ciphertext on disk only |
| AI agent reads secrets without permission | Yes — mountpoint empty until human opens |
| Disk theft | Yes — ciphertext useless without password |
| Password leaks via logs/history | Mitigated — pass via stdin/env, not config |
| Forgot to close | Yes — `--unmount-idle N` auto-unmounts |
| Forgot password | **No recovery** — AES/XChaCha key is derived from password |
| Disk failure without backup | **No recovery** — keep an external backup of the cipher dir |

## How cryfs works

- Backend dir holds encrypted blocks (hex-named) + `cryfs.config`.
- `cryfs.config` stores the encryption key, itself encrypted by your password
  (scrypt KDF).
- Mounting decrypts the key, FUSE presents a plaintext view at the mountpoint.
- Unmounting drops the plaintext view; ciphertext stays on disk.
- Default cipher: `xchacha20-poly1305` (AEAD). Block size 16 KiB.
- File names, directory structure, sizes — all encrypted, nothing leaks.

## Install

### Linux — Debian/Ubuntu/Mint
```bash
sudo apt update && sudo apt install cryfs
```

### Linux — Fedora/RHEL
```bash
sudo dnf install cryfs
```
If not in repos, use the upstream AppImage or build from source
(see https://github.com/cryfs/cryfs).

### Linux — Arch/Manjaro
```bash
sudo pacman -S cryfs
```
AUR alternative: `cryfs-git`.

### Linux — NixOS
```nix
# configuration.nix
environment.systemPackages = with pkgs; [ cryfs ];
# FUSE is usually enabled; otherwise:
# boot.extraModulePackages = [ ];
```

### Linux — Alpine
```bash
sudo apk add cryfs fuse
# ensure /dev/fuse exists and user is in the fuse/group that owns /dev/fuse
```

### Linux — generic
Verify FUSE is usable:
```bash
ls /dev/fuse                 # must exist
cryfs --version              # must print a version
groups | grep -Eo 'fuse|disk' # some distros require membership
```
If `/dev/fuse` is missing, load the module and install the fuse package:
```bash
sudo modprobe fuse
sudo apt/dnf/pacman/apk install fuse fuse3   # pick the right one
```

### macOS
```bash
brew install --cask macfuse
brew install cryfs
```
Reboot after installing macFUSE (kernel extension approval may be required in
System Settings → Privacy & Security).

### Windows
1. Install WinFsp: https://winfsp.dev/rel/
2. Download cryfs Windows build from https://github.com/cryfs/cryfs/releases
   (look for a Windows asset; if none, cryfs on Windows is experimental —
   consider gocryptfs + WinFsp or cryptomator as alternatives).
3. Add both to PATH.
4. Verify in PowerShell: `cryfs --version`.

> Windows/macOS support in cryfs is less mature than Linux. For production
> secrets on those OSes, also consider `gocryptfs` (WinFsp/macFUSE) or
> `cryptomator` (GUI, cross-platform, AES-256).

## Paths (no hardcoding — choose once)

Pick three paths and stick to them. Example layout used below:

| Role | Path |
|------|------|
| Cipher backend (on disk, always) | `~/.vault/mydb.cryfs` |
| Mountpoint (empty when closed) | `~/.vault/mydb` |
| Local state (cryfs-internal) | `~/.local/share/cryfs` (default) |

You can put the backend anywhere: same disk, external drive, cloud-synced
folder (it is already encrypted — cloud cannot read it).

## Migrate an existing plaintext folder

Assume plaintext lives at `~/secrets` and you want it encrypted.

```bash
# 1. Create backend + a temporary mountpoint for migration
mkdir -p ~/.vault/mydb.cryfs
mkdir -p /tmp/vault-tmp

# 2. Initialize cryfs with a password (read from stdin, no echo in history)
export CRYFS_FRONTEND=noninteractive CRYFS_NO_UPDATE_CHECK=true
printf 'YOUR_PASSWORD\n' | cryfs ~/.vault/mydb.cryfs /tmp/vault-tmp

# 3. Copy plaintext in
cp -a ~/secrets/. /tmp/vault-tmp/

# 4. Verify — both must match
diff <(cd ~/secrets && find . -type f -printf '%s %p\n' | sort) \
     <(cd /tmp/vault-tmp && find . -type f -printf '%s %p\n' | sort) \
     && echo SIZES_MATCH
diff <(cd ~/secrets && find . -type f -exec md5sum {} \; | sort -k2) \
     <(cd /tmp/vault-tmp && find . -type f -exec md5sum {} \; | sort -k2) \
     && echo MD5_MATCH

# 5. Confirm password works: unmount, remount, count files
fusermount -u /tmp/vault-tmp   # macOS: umount /tmp/vault-tmp ; Windows: cryfs-unmount
printf 'YOUR_PASSWORD\n' | cryfs ~/.vault/mydb.cryfs /tmp/vault-tmp
ls /tmp/vault-tmp | wc -l      # must equal count in ~/secrets

# 6. Only after both checks pass: remove plaintext, create final mountpoint
fusermount -u /tmp/vault-tmp
rm -rf ~/secrets
mkdir -p ~/.vault/mydb
rmdir /tmp/vault-tmp
```

> **Never delete the plaintext original until the cipher has been verified
> twice (sizes + md5, then remount with password).**

## Daily use

### Human — open
```bash
cryfs --unmount-idle 10 ~/.vault/mydb.cryfs ~/.vault/mydb
# Password: ********
# ... work in ~/.vault/mydb ...
```
`--unmount-idle N` takes a **plain integer** of minutes. `10m` is rejected.

### Human — close manually
```bash
# Linux
fusermount -u ~/.vault/mydb
# macOS
umount ~/.vault/mydb
# Windows (PowerShell)
cryfs-unmount "$HOME\.vault\mydb"
```

### AI agent — open (only when the human explicitly allows it in this session)
```bash
printf 'PASSWORD\n' | cryfs --unmount-idle 10 ~/.vault/mydb.cryfs ~/.vault/mydb
```
- Password comes from stdin, never from a config file, never hardcoded in a
  script, never written to AGENTS.md, logs, or chat history beyond the one
  turn the human sends it in.
- Prefer `read -s` from the human's terminal over echoing the password into
  command history. If you must use `printf`, ensure the line is not logged.
- After unmount the password is gone from memory.

### AI agent — close (mandatory at end of work)
```bash
cd ~                              # leave the mountpoint first
fusermount -u ~/.vault/mydb 2>/dev/null \
  || umount ~/.vault/mydb 2>/dev/null \
  || cryfs-unmount ~/.vault/mydb
```

## Wrapper scripts

### `mybd-open`
```bash
#!/bin/bash
set -e
BASE="${MYBD_BASE:-$HOME/.vault/mydb.cryfs}"
MNT="${MYBD_MNT:-$HOME/.vault/mydb}"
mountpoint -q "$MNT" 2>/dev/null && { echo "already mounted: $MNT"; exit 0; }
export CRYFS_FRONTEND=noninteractive CRYFS_NO_UPDATE_CHECK=true
exec cryfs --unmount-idle "${MYBD_IDLE:-10}" "$BASE" "$MNT"
```

### `mybd-close`
```bash
#!/bin/bash
MNT="${MYBD_MNT:-$HOME/.vault/mydb}"
if mountpoint -q "$MNT" 2>/dev/null; then
  fusermount -u "$MNT" 2>/dev/null \
    || umount "$MNT" 2>/dev/null \
    || cryfs-unmount "$MNT"
  echo "closed"
else
  echo "not mounted"
fi
```

### `mybd-status`
```bash
#!/bin/bash
MNT="${MYBD_MNT:-$HOME/.vault/mydb}"
if mountpoint -q "$MNT" 2>/dev/null; then echo "OPEN: $MNT"; else echo "CLOSED"; fi
```

Install:
```bash
mkdir -p ~/.local/bin
cp mybd-open mybd-close mybd-status ~/.local/bin/
chmod +x ~/.local/bin/mybd-*
# add to PATH if needed
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
```

All scripts honor env overrides (`MYBD_BASE`, `MYBD_MNT`, `MYBD_IDLE`) so the
same scripts work for any vault location without editing.

## Auto-lock

`--unmount-idle N` unmounts after N minutes with no filesystem access. Active
reads/writes reset the timer. After auto-unmount the mountpoint is empty
again and a new `mybd-open` is required.

Do not rely solely on auto-lock. Always close manually when finished.

## Agent discipline (put this in the vault's AGENTS.md)

```
- This folder is a cryfs vault. The human opened it before launching you.
- You do NOT know and do NOT need the password.
- When you finish all work, run `mybd-close` (or fusermount -u / umount /
  cryfs-unmount on the mountpoint).
- Before closing, `cd ~` out of the mountpoint so it is not busy.
- Never write the password anywhere. Never leave the vault mounted "for later".
- If the human asks for more after you closed, ask them to run `mybd-open`
  again — opening is the human's job, closing is yours.
```

## Backup

The cipher backend is already encrypted — copy it anywhere without revealing
the password.

```bash
# to USB
cp -a ~/.vault/mydb.cryfs /media/usb/mydb-backup-$(date +%F)/
# to an archive
tar -cf - -C ~/.vault mydb.cryfs | xz > ~/backups/mydb-$(date +%F).tar.xz
```

Restore: copy the backend back, `mybd-open`, enter password.

Without a backup:
- Forgot password → data lost forever.
- Disk failure → data lost forever.
- Accidental `rm -rf` on backend → data lost forever.

## Portability across systems / disks

The vault is **fully portable** and independent of the OS install. The cipher
backend is just a directory of encrypted blocks — nothing in it depends on the
host system, user ID, distro, or `/home`. Two things are required to recover
access on any new system: **the cipher backend dir** and **the password**.

### Recommendation: keep the backend on a NON-SYSTEM disk

If you have more than one disk/partition, put the backend on a **data disk**
that is separate from the OS (`/`) and from `/home`. Then reinstalling or
replacing the OS never touches the vault.

Example layout (paths are illustrative — pick your own):

| Mount | Device | FS | Role |
|-------|--------|----|------|
| `/` (system) | `/dev/sda1` | ext4/btrfs | OS, `/home`, packages — wipeable |
| `/media/USER/DATA` | `/dev/sdb1` | btrfs | data disk, holds `<vault>.cryfs` |

Corresponding paths (replace `USER`, `DATA`, `vault` with your own):

| Role | Path (example) |
|------|----------------|
| Cipher backend (on data disk) | `/media/USER/DATA/.vault.cryfs` |
| Mountpoint (anywhere, e.g. same data disk) | `/media/USER/DATA/vault` |

On a fresh system, recover access:

```bash
sudo apt install cryfs                       # or dnf/pacman/brew/etc.
mkdir -p /media/USER/DATA/vault              # empty mountpoint on the new system
cryfs /media/USER/DATA/.vault.cryfs /media/USER/DATA/vault
# enter the SAME password → all files are back
```

### If you only have ONE disk

If there is no separate data disk, the vault can still live on the same disk
as the OS — but **this is risky**:

- Reinstalling the OS, reformatting `/`, or deleting `/home` **will wipe the
  vault** unless you move the backend off-disk first.
- A single disk has a single point of failure: if that disk dies, both the
  system and the vault are lost together.

Mitigations when you cannot use a separate disk:
- Put the backend under a path you will remember to back up before any OS
  reinstall, e.g. `/home/USER/.vault/vault.cryfs`, and **copy it to external
  media before reinstalling** (the backend is already encrypted — copy it as
  raw files, no password needed to back it up).
- Keep an external/cloud backup of the backend dir. The backup does not need
  to be on a mounted disk; any copy of the encrypted blocks is sufficient.

### General rules

- The disk/partition that holds the backend is itself a backup as long as it
  is not reformatted, repartitioned, or physically failed. An external/cloud
  backup of the backend is still recommended, but is optional if the data
  disk is left untouched.
- Never point `mybd-open` at a backend inside a path you plan to wipe with the
  OS (e.g. under `/` on a system you will reinstall) — move it to a data disk
  or to external media first.
- The wrapper scripts in `~/.local/bin` are not part of the vault; they are
  trivially reproducible from this SKILL.md on any new system.

## Change password

cryfs cannot reliably re-encrypt the config in place in 0.11.x. Migrate to a
new vault instead:

```bash
mybd-open                                           # open old
mkdir -p ~/.vault/mydb.cryfs.new /tmp/vnew
printf 'NEW_PASSWORD\n' | cryfs ~/.vault/mydb.cryfs.new /tmp/vnew
cp -a "$MYBD_MNT"/. /tmp/vnew/
# verify (sizes + md5) as in migration
fusermount -u /tmp/vnew
mybd-close
rm -rf ~/.vault/mydb.cryfs
mv ~/.vault/mydb.cryfs.new ~/.vault/mydb.cryfs
mybd-open   # use new password
```

Keep the old backend until the new one is verified.

## Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| `Error 10: Invalid arguments` | Options must come before positional args; `--unmount-idle` takes a bare int, not `10m` |
| `fuse: mountpoint is not empty` | Empty the mountpoint, or allow nonempty: `cryfs ... -- -o nonempty` |
| `fusermount: Device or resource busy` | A shell/agent has cwd inside the mountpoint. `cd ~` everywhere, then close. Check with `lsof +D <mountpoint>` |
| Wrong password silently rejected | Re-enter; cryfs has no rate limit. If `cryfs.config` is corrupted and there is no backup, data is lost |
| cryfs stays in background after unmount | Normal — it daemonizes. `pgrep -af cryfs` to see it; do not `kill -9` during writes |
| macOS: mount fails | Approve the macFUSE kext in System Settings → Privacy & Security, reboot |
| Windows: mount fails | Ensure WinFsp service is running; run shell as the same user that owns the backend |

## Verification checklist

- [ ] Closed: mountpoint is not a mountpoint, `ls` is empty.
- [ ] Backend dir contains hex-named subdirs + `cryfs.config`.
- [ ] `file <backend>/<hex>/*` reports `data` (binary, unreadable).
- [ ] Open: mountpoint is a mountpoint, file count matches original.
- [ ] Password is not in `~/.bash_history`, not in any config, not in any doc.
- [ ] `mybd-close` succeeds after `cd ~`.
- [ ] Auto-lock fires after N idle minutes.
- [ ] External backup of the backend exists (or the user has explicitly accepted the no-backup risk).

## Alternatives

- **gocryptfs** — faster, simpler, very stable. Same FUSE model. Good if the
  vault grows large. `sudo apt install gocryptfs`.
- **cryptomator** — GUI-first, cross-platform (incl. Windows/macOS), AES-256.
  Good for non-CLI users.
- **encfs** — old, known weaknesses. Avoid for new setups.
- **LUKS/cryptsetup** — full-block-device encryption. Overkill for one folder.
- **Veracrypt** — container file. Works but less transparent than cryfs.

## Persistence of this skill

Active whenever the user is setting up, migrating, operating, or
troubleshooting a cryfs-encrypted secrets folder, or when an agent needs the
open/close discipline for such a vault. Stays available across turns; no
on/off toggle needed.
