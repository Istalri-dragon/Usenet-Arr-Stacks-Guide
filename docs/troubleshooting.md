# Troubleshooting — the failure states, and what they actually say

[← Home](../README.md)

Most guides only show the happy path. This page is the opposite: it shows what
things look like when they're **broken**, so when your screen looks wrong you can
tell whether it's normal or a real problem — and fix it fast.

It's organised by **symptom**. Jump to the one that matches.

- [It downloads but nothing shows up ("not importing")](#it-downloads-but-nothing-shows-up-not-importing)
- [Hardlinks silently became copies (slow imports, doubled disk)](#hardlinks-silently-became-copies)
- [Permission denied / files owned by the wrong user](#permission-denied--files-owned-by-the-wrong-user)
- [I can't open an app in my browser](#i-cant-open-an-app-in-my-browser)
- [Radarr/Sonarr can't connect to SABnzbd](#radarrsonarr-cant-connect-to-sabnzbd)
- [Searching finds nothing / indexer errors](#searching-finds-nothing--indexer-errors)
- [A container keeps restarting](#a-container-keeps-restarting)
- [Jellyfin won't start after I added a GPU](#jellyfin-wont-start-after-i-added-a-gpu)
- [Reading logs (the one skill that solves most of these)](#reading-logs)

---

## It downloads but nothing shows up ("not importing")

**The single most common failure in the whole stack.** SAB says the download
finished, but the movie never appears in your library.

**Where to look:** Radarr → **Activity → Queue**. The item is usually sitting
there with an orange/red warning icon. Hover it, or open **System → Events**
(turn log level to Debug in `Settings → General` if needed).

**What the message actually looks like** (one of these):

- `No files found are eligible for import in /data/usenet/movies/<name>`
- `Found archive file, might need to be extracted`
- `Import failed, path does not exist or is not accessible by Radarr: /data/usenet/movies/...`
- a permissions line (see the [permissions section](#permission-denied--files-owned-by-the-wrong-user))

**The usual causes, in order of likelihood:**

1. **Path mismatch.** SAB's completed/category folder and
   Radarr's root folder aren't both under the shared `/data` mount. Check:
   - SAB → `Settings → Folders → Completed Download Folder` = `/data/usenet`
   - SAB → `Settings → Categories` → `movies` = `movies`, `tv` = `tv`
   - Radarr → root folder = `/data/media/movies`
   All four are **container** paths. If SAB's completed folder points somewhere
   *outside* `/data`, Radarr can't reach it — fix that and it imports.
2. **Different mounts.** If the paths look right but imports are *slow*, you don't
   have a mismatch, you have a hardlink-that-became-a-copy — see
   [the next section](#hardlinks-silently-became-copies).
3. **Category not set in Radarr.** Radarr → `Settings → Download Clients →
   SABnzbd → Category` must be `movies` (Sonarr: `tv`). If blank, SAB dumps files
   in the wrong place.
4. **Permissions.** The file exists but Radarr can't touch it — see below.

**Fix it, then force it:** after correcting the path, click the item in the Queue
and choose **manual import**, or trigger Radarr to re-scan. You should see it
hardlink instantly into `/data/media/movies`.

---

## Hardlinks silently became copies

**Symptom:** imports *work* but take a long time, and `df -h` shows free space
dropping when Radarr imports — the same file is now on disk twice until SAB
cleans up.

**Why:** hardlinks are only legal *within one filesystem*. If `usenet/` and
`media/` are on different mounts — or you mapped them into the container as two
separate volumes — the app can't hardlink and silently **copies** instead.

**Confirm it.** Watch free space during an import:

```bash
watch -n1 df -h /data
```

- **Correct (hardlink):** the import is instant and the free-space number
  **doesn't move**.
- **Broken (copy):** you see a progress bar and free space **drops** by the size
  of the file.

You can also test hardlinks directly inside the container:

```bash
# from the host:
docker exec -it radarr sh
# then inside:
touch /data/usenet/movies/testfile
ln /data/usenet/movies/testfile /data/media/movies/testlink && echo "HARDLINK OK" || echo "CROSS-DEVICE — this is the problem"
rm /data/usenet/movies/testfile /data/media/movies/testlink
```

If you get `Invalid cross-device link`, your two folders are on different
filesystems.

**Fixes:**

- **Compose:** make sure every file-touching app mounts the **whole** root the
  same way — `- ${DATA}:/data` — and *not* split sub-paths like
  `/data/usenet:/downloads`. This repo's compose already does it right; a custom
  edit is the usual culprit.
- **NAS:** this is almost always the Unraid cache/array mover or a TrueNAS
  cross-pool setup. See [docs/nas.md](nas.md).
- **Separate drives:** put `usenet/` and `media/` on the *same* mount. They must
  be siblings on one filesystem.

---

## Permission denied / files owned by the wrong user

**Symptom:** an app can't write where it needs to. The real errors people hit on
first run:

- **SABnzbd** → `Settings → Folders`, red bar:
  `Failed — download_dir directory: /data/usenet/incomplete error accessing`
- **Seerr** crash-loops; `docker compose logs seerr` shows
  `Error: EACCES: permission denied, mkdir '/app/config/logs/'`
- **Radarr / Sonarr** log `Import failed ... Access to the path '/data/media/movies/<file>' is denied.`
- …or files land in your library **owned by the wrong user** and the media server can't read them.

**Why:** the host folders are owned by **root** — Docker creates any missing
folder as root, and `/data` is often made with `sudo mkdir` — but the apps run as
**you** (`PUID`/`PGID`), so they can't write into them. The LinuxServer apps
(Radarr, Sonarr, SABnzbd…) fix their own `/config` on startup, but nobody fixes
the shared `/data` mount for you, and the non-LinuxServer apps (**Seerr**,
**Profilarr**) don't fix their config folder either — so those are where it bites.

**Fix — make yourself the owner (one time):**

```bash
cd ~/arr-stack           # wherever your docker-compose.yml lives
source .env              # loads DATA and APPDATA from your .env
sudo chown -R "$(id -u):$(id -g)" "$DATA" "$APPDATA"
docker compose restart
```

To check the numbers first, or if it still fails:

```bash
id                       # your user's uid/gid
ls -ln "$DATA"           # numeric owner of the data root (0 0 = root = the problem)
docker exec sabnzbd id   # what a container actually runs as
```

If `ls -ln` shows `0 0` and your `id` is `1000`, the chown above is the whole fix.

**Seerr (and Profilarr) are the exception:** Seerr always runs as its own internal
**UID 1000**, whatever `PUID` you set — so its folder must be owned by `1000`
specifically. If your user isn't 1000, run `sudo chown -R 1000:1000 "$APPDATA/seerr"`
(and the same for `profilarr` if it complains).

---

## I can't open an app in my browser

**Symptom:** `http://SERVER_IP:7878` times out or refuses to connect.

**Checklist:**

1. **Is the container running?** `docker compose ps` — the app should say
   `running`. If it's restarting, read its logs ([below](#reading-logs)).
2. **Right IP?** Get the server's LAN IP with `hostname -I | awk '{print $1}'`.
   `localhost`/`127.0.0.1` only works *on the server itself*, not from another
   machine.
3. **Right port?** See the [port table](../README.md#appendix-port-reference).
   Plex is `32400/web`, not `32400`.
4. **Firewall.** On the server: `sudo ufw status`. If active, allow the port
   (e.g. `sudo ufw allow 7878`).
5. **Same network?** Your browser machine and the server must be on the same LAN
   (this stack is local-only until you add a reverse proxy).

---

## Radarr/Sonarr can't connect to SABnzbd

**Symptom:** in Radarr → `Download Clients → Test`, SABnzbd fails.

**The usual cause:** you put **`localhost` / `127.0.0.1`** in the Host field.
Inside its own container, `localhost` means *Radarr itself* — not SAB — so it
connects to nothing and the test fails. Use the service name instead. (Your
server's LAN IP works too, because SAB's port `8080` is published to the host —
the service name is just cleaner and doesn't break if that IP ever changes.)

- **Host:** `sabnzbd`  (the service name; your server's LAN IP also works —
  just not `localhost`)
- **Port:** `8080`
- **API Key:** from SAB → `Settings → General → API Key`

Also confirm SAB is actually up (`docker compose ps`) and that Radarr and SAB are
in the **same compose project** (they share the network automatically here).

---

## Searching finds nothing / indexer errors

**Symptom:** Radarr/Sonarr searches return zero results, or Prowlarr's indexer
test fails.

**Check, in order:**

1. **Do you have an indexer at all?** Prowlarr → `Indexers`. Usenet needs a
   working indexer *and* a paid provider in SAB — a guide can't supply these
   (see [SABnzbd](04-sabnzbd.md)).
2. **Did Prowlarr sync to the apps?** Prowlarr → `Settings → Apps` should list
   Radarr and Sonarr, and Radarr → `Settings → Indexers` should show the synced
   indexers. If not, re-check the App entries (URLs `http://radarr:7878` /
   `http://sonarr:8989` + API keys).
3. **Indexer credentials.** Test each indexer in Prowlarr; a red result usually
   means a wrong API key or an expired account.

---

## A container keeps restarting

**Symptom:** `docker compose ps` shows a service constantly `restarting`.

**Do this:**

```bash
docker compose logs --tail=50 <service>
```

The real reason is almost always in the last few lines. Common ones:

- **Jellyfin + `/dev/dri` but no GPU:** the device doesn't exist — remove the
  `devices: /dev/dri:/dev/dri` line (see [below](#jellyfin-wont-start-after-i-added-a-gpu)).
- **Bad `.env` value:** a missing `PUID`/`PGID`/`TZ` or a typo. `docker compose
  config` will show how your variables expanded.
- **Port already in use:** `Bind for 0.0.0.0:7878 failed: port is already
  allocated` — something else uses that port; stop it or change the left side of
  the port mapping.

---

## Jellyfin won't start after I added a GPU

**Symptom:** Jellyfin was fine, then you enabled hardware transcoding and now it
won't boot, or `docker compose logs jellyfin` mentions `/dev/dri`.

**Cause:** the compose passes `/dev/dri` to the container, but your box either has
no Intel/AMD iGPU or doesn't expose that device.

**Fixes:**

- **No compatible GPU:** remove the two `devices:` lines under `jellyfin` and
  restart. Jellyfin will transcode on CPU (slower, but works).
- **Have the GPU but no `/dev/dri`:** on the host, `ls -l /dev/dri`. If it's empty,
  your kernel/driver isn't exposing it (common on some NAS models). Enable the
  iGPU / install the driver for your platform first.
- **NVIDIA GPU:** don't use `/dev/dri`. Install the NVIDIA Container Toolkit and
  configure the container for NVIDIA runtime instead — see the Jellyfin
  hardware-acceleration docs.

---

## Reading logs

Ninety percent of everything above is diagnosed the same way — **read the
container's log.** This is the one skill worth building.

```bash
# follow a running app live (Ctrl-C to stop):
docker compose logs -f radarr

# last 50 lines of one service:
docker compose logs --tail=50 sabnzbd

# everything, once:
docker compose logs

# is my compose file / .env even valid? (shows expanded variables)
docker compose config
```

Inside the *arr apps themselves, the GUI log is under **System → Events** (bump
the log level to Debug in `Settings → General` while chasing a problem, then set
it back). The error text in the log is almost always literal and specific — read
it before changing anything, and you'll fix the right thing the first time.
