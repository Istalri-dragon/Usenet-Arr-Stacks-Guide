# Running this stack on a NAS — Unraid, Synology, TrueNAS SCALE

[← Home](../README.md)

Everything in the main [README](../README.md) was written on a plain Linux host
on purpose — that's what keeps the guide honest. But most people run this on a
NAS, so here's how to translate the **exact same stack**.

**The good news:** only two things change per platform — the **paths** and the
**user IDs (PUID/PGID)**. The rule that matters most — *one shared `/data` on one
filesystem so hardlinks work* — is identical everywhere.

> **The universal NAS tell:** if imports are slow and your disk fills up twice as
> fast as it should, your download path and media path are on **different
> filesystems**. Put everything under one shared `/data` on one volume and it's
> fixed. Same rule, every platform.

---

## Unraid

Two ways to run it.

### Option 1 — Community Applications (GUI, one app at a time)

Search each app (SABnzbd, Radarr, Sonarr, Prowlarr, Bazarr, Jellyfin, Seerr,
Profilarr) in **Apps** (Community Applications) and install from
a template (there are several — different community members maintain them).
I recommend **hotio**'s: they're already set up for the folder structure we use.
With LinuxServer or others you may need to change the template's paths from the
defaults. For each app:

- Set **Appdata** to `/mnt/user/appdata/<app>`.
- Add a path mapping: **container path** `/data` → **host path** `/mnt/user/data`.
  (For Jellyfin, `/data/media` → `/mnt/user/data/media` is fine.)

### Option 2 — Compose Manager plugin (paste this repo's file)

Install **"Docker Compose Manager"** from Community Apps → add a new stack →
paste `docker-compose.yml` → set the values from `.env`. This keeps you on the
exact file in this repo.

> **My take:** The Compose Manager route is the one I'd recommend — though I moved
> away from Unraid myself over how it handles some things. Chief among them: the
> so-called "cache" disk that isn't actually a cache but a separate pool. If it
> fills up before the mover kicks in, it breaks things.

### Unraid specifics

- **PUID=99, PGID=100** — Unraid's `nobody:users`. Put these in your `.env` /
  template, *not* 1000.
- `APPDATA=/mnt/user/appdata`
- `DATA=/mnt/user/data`
- **The hardlink caveat that affects Unraid people specifically:** keep the whole
  `data` share on a **single filesystem**. If the share is set to move between the
  **cache pool** and the **array** (the default "Yes"/"Prefer" mover behaviour),
  files cross filesystems mid-move and your hardlinks/atomic moves break — you're
  back to slow copies and doubled disk. Set the `data` share's cache setting so
  downloads and media always sit together. TRaSH has an
  [Unraid-specific page](https://trash-guides.info/Hardlinks/How-to-setup-for/Unraid/)
  on exactly these share settings — follow it precisely.
- **Hardware transcoding:** Intel iGPU works via `/dev/dri` as in the compose.
  For NVIDIA, install the **Nvidia Driver** plugin and pass the GPU through
  instead.

---

## Synology (Container Manager)

> **My take:** I haven't used Synology myself — the cost versus a DIY NAS never
> made sense to me. I had a couple of people who own one review this section, so
> it should be accurate; if you hit issues, let me know.

Synology's **Container Manager** (DSM 7.2+) supports Compose directly.

- **Project → Create → upload or paste `docker-compose.yml`.**
- `APPDATA=/volume1/docker`
- `DATA=/volume1/data`
- Create the folders first in **File Station** (or over SSH), matching the
  structure in the README: `/volume1/data/usenet/{incomplete,movies,tv}` and
  `/volume1/data/media/{movies,tv}`.

### Synology specifics

- **Don't guess PUID/PGID.** Synology user IDs are **not** 1000. Enable SSH
  (`Control Panel → Terminal & SNMP`), then run `id youruser` and use whatever it
  reports. Common values are in the `1026+` range for the first admin user, but
  **check** — using the wrong ones is the #1 Synology permission problem.
- **Hardware transcoding (Jellyfin):** only works on Synology models with an
  **Intel iGPU** that exposes `/dev/dri`. Many DS models (especially AMD- or
  Realtek-based ones) don't have it — if `/dev/dri` doesn't exist on your box,
  remove that device line or the Jellyfin container won't start.
- If Container Manager complains about the `${VAR}` syntax, fill the values in
  its environment UI, or paste a version of the file with the values inlined.

---

## TrueNAS SCALE

Recent SCALE (**Electric Eel / 24.10+**) runs Docker Compose apps directly.

- Add a **Custom App** and paste the YAML, or use the Compose workflow if your
  version exposes it.
- Use **datasets** for both appdata and data, and keep them on the **same
  pool/dataset tree** so hardlinks work:
  - `DATA=/mnt/<pool>/data`
  - `APPDATA=/mnt/<pool>/appdata`
- Create the `usenet/{incomplete,movies,tv}` and `media/{movies,tv}` structure as
  child datasets or plain folders under `data` — just keep them on one pool.

### TrueNAS specifics

- **Find the app user's IDs** from the app shell with `id`, and set PUID/PGID to
  match. SCALE apps often run under a specific `apps` user rather than 1000.
- Make sure the dataset **ACLs/permissions** let that PUID/PGID read and write —
  SCALE's stricter ACLs are a common source of "permission denied" that you won't
  see on a plain Linux box.
- Keep appdata and data on the **same pool** if you can; crossing pools is the
  SCALE version of the Unraid cache/array hardlink trap.

---

## General NAS rule of thumb

If you take nothing else from this page: **one shared `/data`, one filesystem.**
The apps don't care whether they're on Unraid, Synology, TrueNAS, or a Raspberry
Pi — they care that `usenet/` and `media/` are siblings on the same volume so the
import is a hardlink and not a copy. Everything else on this page is just the
local spelling of the paths and user IDs.
