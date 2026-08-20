# 10 · Quality profiles — the payoff *(optional but recommended)*

**[← Prev: Bazarr](09-bazarr.md)** · [Home](../README.md)

---

This is where the setup goes from *"it works"* to *"it's actually good"* — and
it's the part most video guides never reach.

Right now Radarr will grab basically the first matching release it finds. Quality
profiles make it grab the **right** one — good encodes, correct HDR, sane file
sizes — and skip the junk (fake releases, mislabeled files, formats your TV can't
play).

The community reference for this is [TRaSH Guides](https://trash-guides.info/).
You don't hand-copy it — you use a tool to **sync** it. Two options; pick one.

> **My take:** I actually hand-copied the few profiles I cared about rather than
> syncing, but for a fresh setup the tools below are far less fiddly and stay
> up to date on their own.

## Option A — Profilarr (modern GUI, recommended for new installs)

**Web UI:** `http://SERVER_IP:6868`

A web UI that syncs TRaSH-style quality profiles and custom formats into Radarr/
Sonarr — **no YAML**. It's already running from the compose
(`ghcr.io/dictionarry-hub/profilarr:latest`).

1. Open it, and **import/pull** the TRaSH-based database it ships with.
2. **Connect it to Radarr and Sonarr** (their URLs `http://radarr:7878` /
   `http://sonarr:8989` and API keys).
3. Choose the profiles you want (e.g. an HD-1080p or a UHD/Remux profile) and
   **Sync**.
4. Flip to Radarr → `Settings → Profiles` and you'll see the custom formats and
   quality profiles land.

## Option B — Recyclarr (classic, config-as-code)

The lightweight, set-and-forget option: it reads a `recyclarr.yml` and pushes
TRaSH profiles on a schedule. You edit YAML, but it's the "config as code" crowd's
favourite and it's tiny.

It's in `docker-compose.yml` **commented out**. To use it instead of Profilarr:
comment out the `profilarr:` block, uncomment `recyclarr:`, put your
`recyclarr.yml` in `${APPDATA}/recyclarr`, then run
`docker compose up -d recyclarr`. Start from the templates in the
[Recyclarr docs](https://recyclarr.dev/).

## Assign it

Whichever you used: assign the new profile to your movies (Radarr) and shows
(Sonarr) — for existing items, and as the default for new ones. Now Radarr is
picky in exactly the way you want. That's the difference between a library that
works and one that gets you exactly what you want.

---

## That's the whole stack 🎉

Recap: request in **Seerr** → **Radarr / Sonarr** → **SABnzbd** → **hardlinked**
into your library → playing in **Jellyfin / Plex**, with **Bazarr** subtitles and
**Profilarr** quality profiles doing their jobs.

**The one thing this guide didn't do** is make your setup reachable from outside
your house. That's **remote access + a reverse proxy**, and it's its own guide
because doing it wrong is a security footgun (you do *not* want SABnzbd exposed
raw to the internet). Until then, everything here is safe on your local network.

If something broke, [Troubleshooting](troubleshooting.md) is organised by exactly
where. On a NAS, start with [Running on a NAS](nas.md) — 99% of NAS problems are
the two things that differ there (paths and PUID/PGID).

---

**[← Prev: Bazarr](09-bazarr.md)** · [Home](../README.md)
