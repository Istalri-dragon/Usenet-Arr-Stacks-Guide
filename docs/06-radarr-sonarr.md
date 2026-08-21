# 6 · Radarr + Sonarr

**[← Prev: Prowlarr](05-prowlarr.md)** · [Home](../README.md) · **Next: [Media server →](07-media-server.md)**

**Radarr (movies):** `http://SERVER_IP:7878` · **Sonarr (TV):** `http://SERVER_IP:8989`

Find your `SERVER_IP`:

```bash
hostname -I | awk '{print $1}'
```

---

These two are the brains. Radarr does movies, Sonarr does TV — same ideas, so
we'll do **Radarr in full** and you repeat the pattern for Sonarr.

## Connect the download client

`Settings → Download Clients → Add → SABnzbd`:

- **Name:** `SABnzbd` (anything)
- **Host:** `sabnzbd`  ← the **service name**, not `localhost` (inside the container that points at Radarr itself; the LAN IP works but the name is cleaner)
- **Port:** `8080`
- **API Key:** the SAB API key you copied on the [SABnzbd page](04-sabnzbd.md)
- **Category:** `movies` (in Sonarr, set this to `tv`)

Click **Test** → green. Save. (Indexers should already be here if you set up
[Prowlarr](05-prowlarr.md). If not, add them directly here.)

## Set the root folder

`Settings → Media Management → Root Folders → Add Root Folder`:

- **Radarr:** `/data/media/movies`
- **Sonarr:** `/data/media/tv`

This is where your organised library lives — the same `media/` folder Jellyfin/
Plex will read.

## Download folder vs. root folder

Here's the thing that causes ~90% of *"it downloads but nothing shows up in my
library."*

- SABnzbd drops finished movie files in its category folder: `/data/usenet/movies`.
- Radarr's root folder is: `/data/media/movies`.
- **Both are under `/data`, inside the same mount** — so when the download
  finishes, Radarr **hardlinks** the file across into your library. Instant, no
  extra disk. Then it renames it cleanly.

If those two ever end up on **different** mounts, or the SAB category path is
wrong, the import fails outright or silently turns into a slow copy. That symptom
— "download completed, library empty" — is almost always this path relationship
being broken.

## What "not importing" looks like

It's worth breaking this once so you can *recognise* it. In
[Troubleshooting](troubleshooting.md#it-downloads-but-nothing-shows-up-not-importing)
there's a walk-through of the deliberately-broken state: the download completes,
Radarr's **Activity → Queue** shows the item stuck with a warning, and the
**System → Events / log** shows the actual message (e.g. a "not a preferred word
/ no files found eligible for import" or a permissions line). Once you've seen
what it looks like on *your* screen, "not importing" stops being scary and
becomes a two-minute fix.

## Do Sonarr

Same three steps: add SABnzbd (host `sabnzbd`, category `tv`), confirm indexers
are present, set root folder `/data/media/tv`. That's it.

## Quick end-to-end test

In Radarr, add a movie you have the rights to, and hit **Search**. Watch:
Radarr grabs a release → it appears in SABnzbd downloading → on completion it
lands in `/data/media/movies`, renamed. If that loop closes, your core pipeline
works. If it doesn't, [Troubleshooting](troubleshooting.md) is organised by
exactly where it broke.

---

**[← Prev: Prowlarr](05-prowlarr.md)** · [Home](../README.md) · **Next: [Media server →](07-media-server.md)**
