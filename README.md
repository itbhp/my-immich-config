# Immich — Local Docker Setup

Self-hosted [Immich](https://immich.app/) running on macOS via Docker Desktop or Rancher Desktop, with media stored on an external ExFAT drive and the Postgres database on the internal disk.

## Why this layout

Immich has three kinds of state:

- **Originals** (`library/`, `upload/`, `profile/`, `backups/`) — large, grows over time, benefits from external storage.
- **Hot caches** (`thumbs/`, `encoded-video/`) — regeneratable derivatives. Kept on the **internal disk** to avoid the ExFAT/virtiofs race that corrupts derivative writes (see "Broken thumbnails" below).
- **Postgres database** (`postgres/`) — small (hundreds of MB), but requires a POSIX filesystem (permissions, symlinks). **ExFAT breaks it.** Lives on the internal disk.

Overridden via `docker-compose.override.yml`, which bind-mounts `./thumbs` and `./encoded-video` on top of `/data/thumbs` and `/data/encoded-video`.

## Prerequisites

- macOS with [Docker Desktop](https://www.docker.com/products/docker-desktop/) or [Rancher Desktop](https://rancherdesktop.io/) installed and running.
- An external drive formatted ExFAT (cross-platform with Linux).
- Phone and Mac on the same Wi-Fi network (for the Immich mobile app).

## One-time container runtime configuration

Containers run inside a Linux VM. External drives mounted at `/Volumes` are **not** shared with that VM by default — they must be added explicitly.

### Docker Desktop

1. Open **Docker Desktop → Settings → Resources → File sharing**.
2. Add `/Volumes` to the list.
3. Click **Apply & Restart**.

### Rancher Desktop

Rancher Desktop uses Lima under the hood and has no file-sharing UI. Create an override file:

```sh
cat > ~/Library/Application\ Support/rancher-desktop/lima/_config/override.yaml <<'EOF'
mounts:
  - location: "/Volumes"
    writable: true
EOF
```

Then quit and restart Rancher Desktop.

Verify from any container that the drive is visible:

```sh
docker run --rm -v "/Volumes/<DriveName>:/test" alpine ls /test
```

Without this step, bind-mounts to `/Volumes/...` either fail or (worse) silently create empty directories inside the VM, losing data.

## Configuration

### 1. Mount the external drive

Plug in the ExFAT drive. macOS will mount it at `/Volumes/<DriveName>`. Verify:

```sh
ls /Volumes/
```

### 2. Edit `.env`

```env
# Media on the external drive — change <DriveName> to your actual drive
UPLOAD_LOCATION=/Volumes/<DriveName>/immich

# Postgres data stays on the internal disk (ExFAT can't host it)
DB_DATA_LOCATION=./postgres

# TZ=Europe/Rome
IMMICH_VERSION=v3

# Random password — do not reuse
DB_PASSWORD=<generate-with-openssl-rand-hex-24>

DB_USERNAME=postgres
DB_DATABASE_NAME=immich
```

Generate a password:

```sh
openssl rand -hex 24
```

## Running

**Start:**

```sh
docker compose up -d
```

Immich creates its subdirectories (`library/`, `upload/`, `thumbs/`, `encoded-video/`, `profile/`, `backups/`) under `UPLOAD_LOCATION` on first run. Postgres initializes a fresh database in `./postgres/`.

**Stop:**

```sh
docker compose down
```

**Status / logs:**

```sh
docker compose ps
docker compose logs -f immich-server
```

### Startup order matters

Always: **plug in drive → verify it's mounted → `docker compose up -d`**.

If you run `up` before mounting, Docker creates an empty folder at the mount path. macOS then mounts the real drive as `/Volumes/<DriveName> 1`, and your Immich library points at the phantom folder. You'd lose any new uploads when it's cleaned up.

### First-start crash loop on ExFAT (virtiofs timing bug)

On a fresh data directory (ExFAT + virtiofs via Rancher Desktop or Docker Desktop), `immich-server` may crash-loop on the first `up -d` with:

```
Failed to read: "<UPLOAD_LOCATION>/profile/.immich (/data/profile/.immich) - ..."
ENOENT: no such file or directory, open '/data/profile/.immich'
```

Immich writes sentinel `.immich` files in each subfolder and immediately reads them back. On ExFAT the read sometimes misses because the virtiofs cache hasn't synced yet. The files **do** end up on disk — the subsequent read in a later process just needs the cache to catch up.

**Fix:** once the four data subfolders (`library/`, `upload/`, `profile/`, `encoded-video/`) have been created with `.immich` files inside, restart the server:

```sh
docker compose restart immich-server
```

The retry reads the existing files successfully and the stack comes up healthy. This only affects fresh starts — later restarts aren't impacted.

### Broken thumbnails / "error loading" tiles in the web UI

Same root cause as the first-start crash: virtiofs on ExFAT lags on write-then-immediate-read. One worker generates a thumbnail JPEG, another worker tries to read it for face detection — and the read sees `ENOENT` even though the file is on disk. Symptoms:

- Photos show "error loading" tiles at `http://localhost:2283/photos`.
- Logs show `ENOENT: no such file or directory, open '/data/thumbs/.../preview.jpeg'`.
- Follow-up jobs (face detection, smart search) fail with `ffprobe exited with code 1` or similar on the same assets.

**Regenerate missing derivatives from the admin UI:**

1. Open http://localhost:2283/admin/jobs-status
2. On each of these jobs, click **"Missing"** (not "All", unless you really want to redo everything):
   - **Generate Thumbnails** — fixes the broken tiles.
   - **Metadata Extraction** — EXIF, dimensions, timestamps.
   - **Smart Search** — CLIP embeddings used by text search.
   - **Face Detection** — recognizes faces for the People view.

These are server-side jobs. **The phone does not need to be connected** — Immich reads the originals already on disk and derives everything locally. You can close the mobile app or leave Wi-Fi without affecting progress. Progress counters on the same page refresh live.

This repo already has the durable fix in place: `docker-compose.override.yml` moves `thumbs/` and `encoded-video/` onto the internal disk (`./thumbs/`, `./encoded-video/`) so the race can't happen. Only `library/`, `upload/`, `profile/`, `backups/` remain on ExFAT — and those are written once, not in rapid write-then-read cycles.

**If you ever recreate those two folders from scratch** (e.g. after wiping them), Immich's DB still remembers the `.immich` sentinel files existed and will refuse to start. Fix:

```sh
touch ./thumbs/.immich ./encoded-video/.immich
docker compose restart immich-server
```

### "not a directory" error on `docker compose up`

If you see:

```
failed to fulfil mount request: open /Volumes/<DriveName>/immich: not a directory
```

…but `ls "/Volumes/<DriveName>/immich"` on the Mac shows the folder is there, then the SSD was plugged in **after** Rancher Desktop / Docker Desktop started. The VM's `/Volumes` share was scanned at VM boot, and it doesn't see the SSD.

**Fix:**

1. `docker compose down`
2. Fully quit Rancher Desktop (menu bar icon → Quit, not just close the window) — or Docker Desktop.
3. Confirm the SSD is mounted on the Mac (visible in Finder under Locations).
4. Start Rancher Desktop / Docker Desktop again, wait until it's fully ready.
5. Verify the VM now sees the drive:
   ```sh
   docker run --rm -v "/Volumes/<DriveName>/immich:/test" alpine ls /test
   ```
   Should list `backups encoded-video library profile thumbs upload`.
6. `docker compose up -d`

If step 5 still fails after a clean restart, re-check the Lima override at `~/Library/Application Support/rancher-desktop/lima/_config/override.yaml` has the `/Volumes` mount (see "One-time container runtime configuration" above). Last resort: Rancher Desktop → Troubleshooting → Factory Reset — rebuilds the VM; data on the SSD and `./postgres/` is untouched.

### Shutdown order matters

Always: **`docker compose down` → unplug drive**.

Unplugging while containers hold the mount can corrupt in-flight writes.

## Accessing Immich

### From the Mac (web UI)

```
http://localhost:2283
```

First launch asks you to create an admin user.

### From the phone (Immich mobile app)

The phone can't reach `localhost` on the Mac. Use either option below.

**Option A — mDNS (easier):**

```
http://<hostname>.local:2283/api
```

The `.local` hostname is advertised by macOS automatically (Bonjour). Works out of the box with Android 12+ and iOS. The `/api` suffix is required by the mobile app.

To shorten the hostname: **System Settings → General → Sharing → Local hostname → Edit** → set e.g. `immich`, then use `http://immich.local:2283/api`.

**Option B — LAN IP:**

```sh
ipconfig getifaddr en0
```

Use `http://<that-ip>:2283/api`. IP can change on reconnect — fix with a DHCP reservation on your router.

### Troubleshooting the connection

1. Test from the phone's browser first: `http://<hostname>.local:2283` — if the web UI loads, the mobile app will connect too.
2. Phone must be on the same Wi-Fi as the Mac.
3. macOS firewall: **System Settings → Network → Firewall** — allow Docker or disable temporarily.

## ExFAT caveats for media

The media directory works on ExFAT with these limitations:

- **No hardlinks/symlinks.** Immich's "external library" feature uses them — avoid it. The normal upload flow (file copies from `upload/` to `library/`) is fine.
- **No POSIX permissions.** Harmless for media files; this is why Postgres can't live here.
- **Case-insensitive filenames.** Matches macOS/Windows behavior but may surprise if the drive is read on Linux with mixed-case names.
- **No inotify across the VM boundary.** "Watch folder" tools inside the container won't see host-side changes in real time.

## Backup

### Local backup

The media directory is already on an external drive — back it up by copying `/Volumes/<DriveName>/immich/library` to another location (originals only; `thumbs/` and `encoded-video/` are regeneratable).

The Postgres database lives in `./postgres/` next to the compose file. Back it up either by:

- Stopping the stack and copying `./postgres/`, or
- Using Immich's built-in database dump (writes into `backups/` under `UPLOAD_LOCATION`).

The `.env` file holds the DB password — back it up separately. Losing it means the database can't be opened again.

### Offsite backup with rclone

The SSD is one copy. For 3-2-1 safety (3 copies, 2 media, 1 offsite), sync `library/` to an object store. Example with Backblaze B2 (~$6/TB/month), encrypted so the provider can't read your photos.

**Install and configure:**

```sh
brew install rclone
rclone config
```

In the wizard: create a `b2` remote (paste keyID + applicationKey from Backblaze → App Keys, restricted to one bucket), then wrap it in a `crypt` remote named `b2crypt` pointing at `b2:<bucket>/immich` with a random password and salt. **Save the password and salt in a password manager — losing them means losing the data.**

**What to sync:**

- ✅ `library/` — originals
- ❌ `thumbs/`, `encoded-video/` — regeneratable
- ❌ `upload/` — transient staging
- ❌ `postgres/` — **never rclone a live database**; use Immich's DB dump job and rclone the `backups/` folder instead

**Sync commands:**

```sh
# Dry-run first
rclone sync --dry-run --progress \
  "/Volumes/<DriveName>/immich/library" b2crypt:library

# Real run
rclone sync --progress --transfers 8 --checkers 16 \
  "/Volumes/<DriveName>/immich/library" b2crypt:library
```

`sync` is destructive on the destination (mirrors deletes). Use `rclone copy` for append-only if you prefer never losing offsite copies — at the cost of storage for deleted photos.

**Verify a restore** before trusting the backup:

```sh
mkdir /tmp/restore-test
rclone copy b2crypt:library/<some-folder> /tmp/restore-test
# open the files — do they look right?
rm -rf /tmp/restore-test
```

A backup you've never restored is a prayer.

## Freeing phone space after backup

The Immich mobile app can delete photos from your phone **only after confirming they're on the server**.

1. Open the Immich app → **Backup** screen.
2. Tap **"Free up space"** (or "Delete backed up assets" on some versions).
3. Review the list, confirm. Photos go to the OS's Recently Deleted (30-day recovery window on iOS and Android).

**Before deleting from the phone, sanity-check:**

- All photos visible in the web UI at http://localhost:2283/photos.
- Thumbnails loading (run the regeneration jobs if not — see above).
- Click into a few photos and download the full original to confirm it's really there.
- You have a **second copy** of the library (SSD alone is not enough — see offsite backup above).

## Importing existing photos from Amazon Photos

For photos no longer on the phone but still on Amazon Photos.

### Download from Amazon

There's no official sync API. Three options, in order of recommendation:

1. **"Request Your Information"** (https://www.amazon.com/gp/privacycentral/dsar/preview.html) — select Amazon Photos. Amazon prepares a full export (can take days), emails a link. Preserves the most metadata.
2. **Web bulk download** — https://www.amazon.com/photos → Select All → Download. Capped at ~5 GB per zip. Tedious for large libraries; filter by year to keep it manageable.
3. **Unofficial CLI** (`amazon-photos` on PyPI) — reverse-engineered, may break anytime, violates Amazon ToS. Use at your own risk.

Unzip the exported archives to a folder on your SSD, e.g. `/Volumes/<DriveName>/amazon-export/`.

### Upload into Immich with the CLI

```sh
npm install -g @immich/cli

# Grab an API key from the web UI → Account Settings → API Keys
immich login http://localhost:2283/api <your-api-key>

# Dry-run
immich upload --dry-run --recursive "/Volumes/<DriveName>/amazon-export/"

# Real upload
immich upload --recursive "/Volumes/<DriveName>/amazon-export/"
```

Immich dedups on SHA-1 hash, so:
- Re-running the CLI is safe (duplicates are skipped server-side).
- Photos you already backed up from the phone won't be re-added.

Useful flags: `--album <name>`, `--concurrency 4`. Check `immich upload --help`.

After upload, run the admin jobs (http://localhost:2283/admin/jobs-status → "Missing" on Thumbnails, Metadata Extraction, Smart Search, Face Detection) so the new assets get indexed.

### Known gotchas

- **EXIF dates**: Amazon sometimes strips or alters metadata (HEIC → JPEG conversions, Live Photos). "Request Your Information" preserves more than web zips.
- **Live Photos / Motion Photos**: often split into separate still + video files by Amazon. Immich can't auto-re-pair them.
- **Don't cancel Amazon Photos** until you've verified every photo arrived in Immich and your offsite backup is working.

## Reboots and data persistence

Rebooting the Mac or quitting Rancher Desktop / Docker Desktop **does not** lose data. Postgres stores its files in `./postgres/` on your Mac via a bind mount — containers are just processes, stopping them doesn't touch host files. On next start, Postgres opens the same files and resumes.

**Safe operations:**

- Rebooting the Mac.
- Quitting Rancher Desktop / Docker Desktop.
- `docker compose down` (stops containers, bind-mounted data stays).
- `docker compose stop` / `start` / `restart`.

**Operations that lose data:**

- `docker compose down -v` — removes named volumes. Postgres uses a bind mount so it survives, but the `model-cache` named volume gets wiped (Immich re-downloads ML models on next start — slow but recoverable).
- `rm -rf postgres/` or deleting the external drive's `immich/` folder.
- Resetting / deleting the Rancher Desktop or Docker Desktop VM. Bind mounts under `/Users/` survive (data is on the Mac filesystem); named volumes do not.

**Tip:** before rebooting, run `docker compose down` so Postgres flushes cleanly. It has WAL + crash recovery and almost always survives an ungraceful kill, but a clean stop is free insurance.

## Resetting everything

To wipe all data and start fresh (destructive):

```sh
docker compose down -v
rm -rf /Volumes/<DriveName>/immich ./postgres
docker compose up -d
```

## Updating Immich

```sh
docker compose pull
docker compose up -d
```

Pinned to `v3` in `.env` — follow a major release? Change `IMMICH_VERSION` and re-run the two commands above. Check the [release notes](https://github.com/immich-app/immich/releases) first; some versions require manual DB migrations.

