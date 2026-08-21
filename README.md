# The *arr Stack (Usenet Edition)

A complete, current (2026) Docker Compose stack for an automated media library —
split into short, one-page-per-app guides so you're never staring at a wall of text.

**The stack:** SABnzbd → Prowlarr → Radarr + Sonarr → Jellyfin *or* Plex → Seerr,
with Bazarr for subtitles and Profilarr/Recyclarr for quality profiles.

Built because I see the same few issues all the time and wanted a guide I could
point to that actually covers them — including the "obvious" steps most guides
skip. It's the written companion to a video I plan to make but might never get to;
either way, someone starting from a **blank machine** can reach a fully working
setup without hitting one of the walls I see people hit all the time.

> **What this is:** legitimate open-source automation for a media library *you*
> fill from sources *you* have the right to use. It's the plumbing — organising,
> renaming, fetching subtitles, keeping quality consistent — not a content source.

> **Usenet only.** This edition deliberately leaves out torrents (and the VPN
> that torrents need in 2026). Usenet is a direct, encrypted connection to a
> provider you pay — there's no swarm, so there's nothing for a VPN to hide and
> nothing that "brings its own" networking gotchas. Fewer moving parts, fewer
> ways to get stuck.

> Blocks marked **My take** (on the pages below) are my personal opinion, not
> required setup — skip them if you just want the steps.

---

## How it fits together

```
                         ┌─────────────┐
      you ask for a      │    Seerr    │   request page ("I want this movie")
      movie/show  ─────► │   :5055     │
                         └──────┬──────┘
                                │ tells
                                ▼
   ┌───────────┐         ┌─────────────┐         ┌───────────────┐
   │ Prowlarr  │  feeds  │   Radarr    │  hands   │    SABnzbd    │
   │  :9696    │ ──────► │  (movies)   │ ───────► │    :8080      │
   │ indexers  │  indexers│  :7878     │ download │ (Usenet DL)   │
   └───────────┘         │   Sonarr    │  job     └───────┬───────┘
                         │   (TV)      │                  │ downloads to
                         │   :8989     │ ◄────────────────┘ /data/usenet
                         └──────┬──────┘   then Radarr/Sonarr HARDLINK
                                │ import   the file into /data/media
                                ▼
                         ┌─────────────┐         ┌───────────────┐
                         │  Jellyfin   │         │    Bazarr     │
                         │  or Plex    │         │    :6767      │
                         │  :8096      │         │  (subtitles)  │
                         │ what you    │         └───────────────┘
                         │ actually    │
                         │ watch       │   Profilarr :6868 tunes what
                         └─────────────┘   Radarr/Sonarr are allowed to grab
```

Two flows: **requests** go right-to-left across the top (you ask in Seerr → it
tells Radarr/Sonarr → they search your indexers → they hand a job to SABnzbd);
**files** go left-to-right across the bottom (SABnzbd downloads into
`/data/usenet` → Radarr/Sonarr hardlink into `/data/media` → your media server
plays it → Bazarr adds subtitles).

| App | Role |
|-----|------|
| **SABnzbd** | The download client. Actually pulls files from your provider. |
| **Prowlarr** | Indexer manager. One place to add "where to search"; syncs to Radarr/Sonarr. |
| **Radarr** | The brain for **movies**. Decides what to grab, tells SAB, organises the result. |
| **Sonarr** | The brain for **TV**. Same idea as Radarr. |
| **Jellyfin / Plex** | The media server — what you actually watch on. Pick one. |
| **Seerr** | The request page. "I want this" → it tells Radarr/Sonarr. |
| **Bazarr** | Fetches subtitles for what's in your library. Optional. |
| **Profilarr / Recyclarr** | Quality control — makes Radarr/Sonarr grab the *right* release, not just the first. |

**Two services you must bring yourself:**

- **An indexer** (such as NZBGeek) — *how you search* Usenet.
- **A Usenet provider** (such as Thundernews) — *who you actually download from*.

Neither comes with the stack; both are covered on the [SABnzbd page](docs/04-sabnzbd.md).

---

## Setup

New to this? Do the pages top to bottom. Each one is short and links to the next.

| # | Page | What it covers |
|---|------|----------------|
| 1 | **[Folder structure](docs/01-folder-structure.md)** ⚠️ | Read this first — the one decision that, done wrong, quietly wrecks everything. |
| 2 | [Prerequisites](docs/02-prerequisites.md) | Install Docker, sort out storage, how to open an app in a browser. |
| 3 | [Docker Compose base](docs/03-docker-compose.md) | The `.env`, the three details everyone skips, bringing it all up. |
| 4 | [SABnzbd](docs/04-sabnzbd.md) | The Usenet download client — and the provider + indexer you supply. |
| 5 | [Prowlarr](docs/05-prowlarr.md) | Indexer manager. *Optional if you only have 1–2 indexers.* |
| 6 | [Radarr + Sonarr](docs/06-radarr-sonarr.md) | The brains — and the handshake that causes "not importing." |
| 7 | [Media server](docs/07-media-server.md) | Jellyfin vs Plex, both finished. |
| 8 | [Seerr](docs/08-seerr.md) | The request front-end. |
| 9 | [Bazarr](docs/09-bazarr.md) | Subtitles. *Optional.* |
| 10 | [Quality profiles](docs/10-quality-profiles.md) | Profilarr / Recyclarr — the payoff that makes it *good*. *Optional but recommended.* |

**The core loop is just 1 → 4 → 6 → 7 → 8** (folders, SABnzbd, Radarr/Sonarr, a
media server, Seerr). Prowlarr, Bazarr, and quality profiles are enhancements you
can add whenever — they're marked *optional* above so you can see the short path.

**Reference (when something breaks or you're on a NAS):**

- [Troubleshooting](docs/troubleshooting.md) — every failure state, what the error actually says, and the fix.
- [Running on a NAS](docs/nas.md) — Unraid, Synology, TrueNAS SCALE.

---

## Appendix: port reference

Open each in a browser at `http://SERVER_IP:PORT`.

| App          | Port  | Purpose                                    |
|--------------|-------|--------------------------------------------|
| SABnzbd      | 8080  | Usenet download client                     |
| Prowlarr     | 9696  | Indexer manager                            |
| Radarr       | 7878  | Movies                                     |
| Sonarr       | 8989  | TV                                         |
| Bazarr       | 6767  | Subtitles                                  |
| Jellyfin     | 8096  | Media server (default)                     |
| Plex         | 32400 | Media server (alternative; `/web`)         |
| Seerr        | 5055  | Requests                                   |
| Profilarr    | 6868  | Quality profiles (default)                 |

**Internal host names** (app-to-app, no IP): `sabnzbd`, `prowlarr`, `radarr`,
`sonarr`, `bazarr`, `jellyfin`, `seerr`, `profilarr`.

---

## Appendix: currency notes (this stack is 2026-correct)

- **Seerr**, not Overseerr/Jellyseerr — those two merged into Seerr in early 2026
  (`ghcr.io/seerr-team/seerr:latest`). Any guide still telling you to pick between
  them is out of date.
- **Profilarr** is at `ghcr.io/dictionarry-hub/profilarr` — the modern GUI
  alternative to Recyclarr's YAML.
- **Usenet-only by design** — no VPN/gluetun and no torrent client in this
  edition; Usenet doesn't need them.
- Everything aligned to **[TRaSH Guides](https://trash-guides.info/)**.

---

## Legal note

The *arr apps are legitimate automation tools. Use them with content and sources
you have the right to use. This repo covers software architecture only, and never
shows copyrighted content being acquired.
