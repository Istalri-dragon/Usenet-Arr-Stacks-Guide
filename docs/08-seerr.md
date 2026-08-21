# 8 · Seerr — the request layer

**[← Prev: Media server](07-media-server.md)** · [Home](../README.md) · **Next: [Bazarr →](09-bazarr.md)**

**Web UI:** `http://SERVER_IP:5055`

Find your `SERVER_IP`:

```bash
hostname -I | awk '{print $1}'
```

---

Seerr is the nice front door: users search for something, click **Request**, and
it flows into Radarr or Sonarr automatically. It's what turns "SSH in and add a
movie in Radarr" into "click a poster."

## Overseerr and Jellyseerr are now Seerr

If you're following older guides you'll see them tell you to choose between
**Overseerr** and **Jellyseerr**. **That choice no longer exists.** In early 2026
those two projects **merged into a single app called Seerr** — one image, works
with Plex, Jellyfin, *and* Emby. If a guide is still framing it as
"Overseerr vs Jellyseerr," that's your signal it hasn't been updated. This stack
uses `ghcr.io/seerr-team/seerr:latest`.

*(Migrating an existing Overseerr/Jellyseerr? Your old config generally carries
over — see Seerr's official migration guide at <https://docs.seerr.dev/migration-guide/>.
This guide covers a fresh install.)*

## Set it up

On first load, Seerr walks you through:

1. **Sign in with your media server.**
   - **Jellyfin:** choose Jellyfin, enter `http://jellyfin:8096` (service name),
     and your Jellyfin admin login.
   - **Plex:** sign in with your plex.tv account and pick your server.
2. **Sync libraries** — let it pull in your Movies and TV libraries.
3. **Connect Radarr** → `Settings → Services → Add Radarr`:
   - **Server:** `http://radarr:7878`
   - **API Key:** Radarr's API key
   - **Root folder:** `/data/media/movies`, **Quality Profile:** pick one (after
     [quality profiles](10-quality-profiles.md) you'll have good ones)
4. **Connect Sonarr** the same way (`http://sonarr:8989`, root `/data/media/tv`).

## Test the whole flow

Do one real request end-to-end so you *see* the whole system work: request in
Seerr → Radarr grabs it → SABnzbd downloads → hardlink import into
`/data/media` → it appears in Jellyfin/Plex. When that loop closes, you have a
complete automated media pipeline.

---

**[← Prev: Media server](07-media-server.md)** · [Home](../README.md) · **Next: [Bazarr →](09-bazarr.md)**
