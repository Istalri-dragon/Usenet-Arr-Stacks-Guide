# 9 · Bazarr — subtitles *(optional)*

**[← Prev: Seerr](08-seerr.md)** · [Home](../README.md) · **Next: [Quality profiles →](10-quality-profiles.md)**

**Web UI:** `http://SERVER_IP:6767`

Find your `SERVER_IP`:

```bash
hostname -I | awk '{print $1}'
```

---

Optional, but most people want it. Bazarr watches your library and fetches
subtitles automatically.

1. **Connect it to Radarr and Sonarr** → `Settings → Radarr` and `Settings →
   Sonarr`:
   - **Address:** `radarr` / `sonarr` (service names), **Port** `7878` / `8989`,
     plus each API key.
2. **Set the paths** so Bazarr's view of the library matches Radarr/Sonarr's.
   Because Bazarr mounts the same `/data` root, the paths line up — if it asks
   for a path mapping, map Radarr's `/data/media/movies` to the same
   `/data/media/movies`.
3. **Pick languages** → `Settings → Languages`, create a profile (e.g. English),
   and assign it as the default for movies and series.
4. Choose subtitle **providers** → `Settings → Providers` (OpenSubtitles and
   friends; some need a free account).

It sees the same library because it mounts the same `/data`. Consistent paths,
again, paying off.

> **My take:** I haven't actually set Bazarr up myself — Plex's built-in
> OpenSubtitles search is enough for me. This section is based on Bazarr's docs
> and on setting it up for a friend, so if something here doesn't work, let me
> know and I'll revisit it.

---

**[← Prev: Seerr](08-seerr.md)** · [Home](../README.md) · **Next: [Quality profiles →](10-quality-profiles.md)**
