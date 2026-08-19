# The *arr Stack (Usenet Edition)

A complete, current (2026) Docker Compose stack for an automated media library,
built because I see the same few issues all the time and wanted a guide I could
point to that actually covered them.

**The stack:** SABnzbd → Prowlarr → Radarr + Sonarr → Jellyfin *or* Plex → Seerr,
with Bazarr for subtitles and Profilarr/Recyclarr for quality profiles.

This repo is the written companion to a video I plan to make but might never
actually get to, soooo — either way, it's built so someone starting from a
**blank machine** can reach a fully working setup without hitting one of the
walls I see people hit all the time. Every branch is finished. The "obvious"
prerequisites are here on purpose.

> **What this is:** legitimate open-source automation for a media library *you*
> fill from sources *you* have the right to use. It's the plumbing — organising,
> renaming, fetching subtitles, keeping quality consistent — not a content source.

> **Usenet only.** This edition deliberately leaves out torrents (and the VPN
> that torrents need in 2026). Usenet is a direct, encrypted connection to a
> provider you pay — there's no swarm, so there's nothing for a VPN to hide and
> nothing that "brings its own" networking gotchas. Fewer moving parts, fewer
> ways to get stuck. If you want the torrent path later, it can be added on.

**Reference pages (split out so this page stays a clean read-through):**

- [docs/nas.md](docs/nas.md) — running this on Unraid, Synology, or TrueNAS SCALE.
- [docs/troubleshooting.md](docs/troubleshooting.md) — every failure state, what the
  error actually says, and how to fix it.

---

## Table of contents

1. [The map — what you're building](#1-the-map--what-youre-building)
2. [The folder structure (read this before anything else)](#2-the-folder-structure-read-this-before-anything-else)
3. [Prerequisites — install Docker, sort out storage](#3-prerequisites--install-docker-sort-out-storage)
4. [The Docker Compose base](#4-the-docker-compose-base--the-three-details-everyone-skips)
5. [Bring it up for the first time](#5-bring-it-up-for-the-first-time)
6. [SABnzbd — the download client (and what you must bring yourself)](#6-sabnzbd--the-download-client-and-what-you-must-bring-yourself)
7. [Prowlarr — indexers in one place](#7-prowlarr--indexers-in-one-place)
8. [Radarr + Sonarr — the handshake that causes "not importing"](#8-radarr--sonarr--the-handshake-that-causes-not-importing)
9. [Media server — Jellyfin vs Plex (both finished)](#9-media-server--jellyfin-vs-plex-both-finished)
10. [Seerr — the request layer](#10-seerr--the-request-layer)
11. [Bazarr — subtitles (optional)](#11-bazarr--subtitles-optional)
12. [Quality profiles — the payoff](#12-quality-profiles--the-payoff)
13. [Wrap-up + what's next](#13-wrap-up--whats-next)
14. [Appendix: port reference](#appendix-port-reference)
15. [Appendix: currency notes (this stack is 2026-correct)](#appendix-currency-notes-this-stack-is-2026-correct)

> Throughout this guide, blocks marked **My take** are my personal opinion, not
> required setup — skip them if you just want the steps.

---

## 1. The map — what you're building

Before touching anything, here's the shape of the whole system, so you understand
what each part does and how they tie together.

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

Read it as two flows:

- **Requests flow right-to-left at the top:** you ask in Seerr → it tells Radarr
  or Sonarr → they search your indexers (managed by Prowlarr) → they hand a
  download job to SABnzbd.
- **Files flow left-to-right at the bottom:** SABnzbd downloads into `/data/usenet`
  → Radarr/Sonarr hardlink the finished file into `/data/media` → your media
  server (Jellyfin or Plex) shows it → Bazarr adds subtitles.

Every time we add a container below, it maps back to a box on this diagram.

**What does what:**

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

**Separate services you need (you bring these yourself):**

- **An indexer** (such as NZBGeek) that indexes the files that are on Usenet — this is *how you search*.
- **A Usenet provider** (such as Thundernews) that actually hosts the files you download — this is *who you download from*.

Full in-depth breakdowns of each already exist online if you want the technical
details behind how each one works. This is just the simple "what they do in this
context" explanation.

---

## 2. The folder structure (read this before anything else)

> This is deliberately step one. Get it wrong and everything downstream may still
> *work* — but it's slow and eats double the disk, so we cover it first.

(Major thanks to [TRaSH Guides](https://trash-guides.info/) for doing the work
on this in the first place — we're just borrowing their data structure without
the torrents part here. Torrents can always be added later with this design.)

Everything lives under **one shared data root**, `/data`. This means everything
uses the same path and prevents a ton of issues down the line:

```
/data
├── usenet/            ← SABnzbd downloads land here, split by category
│   ├── incomplete/    ← in-progress downloads (SAB's temp folder)
│   ├── movies/        ← finished movie downloads
│   └── tv/            ← finished TV downloads
└── media/             ← your actual library (this is what Jellyfin/Plex read)
    ├── movies/
    └── tv/
```

### Why one root? (this is the whole point)

When Radarr or Sonarr finishes a download, it moves the file from `usenet/` into
`media/`. You want that move to be a **hardlink**, not a copy.

A hardlink is a second *name*/*link* for the exact same file on disk. It's created
instantly and it uses **zero** extra space — the download and the library entry
are literally the same bytes with two names. When SAB later cleans up its copy,
the file stays put in your library, untouched.

But hardlinks have **one iron rule: both names have to be on the same
filesystem.** Keep downloads and media as siblings under one `/data` root on one
disk, and every import is instant.

### What "getting it wrong" looks like

If downloads and media are on **different** filesystems — or you map them into the
container as two separate volumes like `/downloads` and `/movies` — the hardlink
becomes illegal, and the app silently falls back to a **copy**:

- The import *crawls* instead of finishing instantly — it has to copy every byte
  from one place to another instead of just creating a link, wasting time, power,
  and drive wear.
- Your disk usage **doubles** until SAB cleans up the original — a 40 GB
  download is now 80 GB on disk for a while.
- On a big library this is the difference between "instant" and "why is my array
  full." (Unraid is bad for this since the cache drive is technically a storage
  pool, not a true cache — see [docs/nas.md](docs/nas.md).)

You can *see* the difference. A correct import is instant and `df -h` doesn't
move. A broken one shows a progress bar and the free space dropping. There's a
side-by-side of exactly this in [docs/troubleshooting.md](docs/troubleshooting.md#hardlinks-silently-became-copies).

This layout follows the [TRaSH Guides](https://trash-guides.info/) recommendation —
the reference the whole community trusts, so you're not inventing anything.
Technically the root folder can be called anything, but `data` (with `media`
under it) is the recommended convention.

### Create it once

On basically any standard Linux distro:

```bash
mkdir -p /data/usenet/{incomplete,movies,tv}
mkdir -p /data/media/{movies,tv}
```

(If `/data` is on a separate drive or NAS share, create this structure *on that
mount* — the point is that everything under `/data` is one filesystem. More on
that in [Prerequisites](#3-prerequisites--install-docker-sort-out-storage).)

On Unraid you'll need to create the root folder/share under **Shares** first
(who'd have guessed), then you can create the subfolders from Windows if that's
more familiar.

---

## 3. Prerequisites — install Docker, sort out storage

If you're on a clean, brand-new machine, you literally can't skip these steps.
There's no Docker to run the containers and nowhere for the files to go.

### Install Docker

```bash
curl -fsSL https://get.docker.com | sh
```

That's the **official** convenience script. It installs Docker Engine and the
Compose plugin. Confirm it worked:

```bash
docker --version
docker compose version
```

Both should print a version now. (If `docker compose version` errors but
`docker --version` works, you have an old standalone `docker-compose` — install
the Compose plugin, or use `docker-compose` with a hyphen throughout.)

**Optional but recommended — run Docker without `sudo`:**

```bash
sudo usermod -aG docker $USER
# then log out and back in (or run: newgrp docker)
```

### Storage — where `/data` actually lives

`/data` needs to be **one filesystem**, because that's what makes hardlinks
legal (section 2).

- **Single-disk box or VM:** nothing to do — `/data` is just a folder on your
  main disk. Fine for most people.
- **Separate media drive:** mount the drive, and put `/data` on it (or make
  `/data` a symlink/bind onto it). The critical part is that `usenet/` and
  `media/` end up on the **same** mount.
- **NAS (Unraid / Synology / TrueNAS):** the paths and the hardlink caveats are
  different enough to have their own page — see [docs/nas.md](docs/nas.md).

### The one thing nobody tells beginners

To open any of these apps, you type the **server's IP address and a port number**
into a browser. That's it. If your server's IP is `192.168.1.50`, then Radarr is
at `http://192.168.1.50:7878`, SABnzbd at `http://192.168.1.50:8080`, and so on.

Find your server's IP with:

```bash
hostname -I | awk '{print $1}'
```

You'll do this a dozen times below. If you've never opened a web UI by typing
`IP:port`, that's the whole trick — there's no app to download, it's just a web
page your server is serving. Throughout this guide, wherever you see
`SERVER_IP`, substitute that address.

---

## 4. The Docker Compose base — the three details everyone skips

Instead of running a dozen `docker run` commands by hand, we describe the whole
stack in one file — `docker-compose.yml` — and bring it up together. The file in
this repo is ready to go; you only edit the `.env` alongside it.

A single combined file does **not** trap you into restarting everything at once.
`docker compose` targets individual services — `docker compose restart radarr` or
`docker compose up -d radarr` bounces just that one and leaves Jellyfin/Plex (and
your viewers) untouched. So you rarely have a reason to bring the whole stack
down.

> **My take:** I still run **two** compose files — one for the user-facing apps
> (Jellyfin/Plex + Seerr) and one for the automation stack — mostly for tidiness.
> If you ever split them, know the catch: separate compose files get separate
> Docker networks, so service names stop resolving across them and Seerr can no
> longer reach Radarr/Sonarr by name. Making it work means creating one shared
> external network (`docker network create arr`) and attaching both files' services
> to it. This repo ships a single combined file precisely because it sidesteps all
> of that — which is why it's the path I'd point a first-timer to.

```bash
git clone https://github.com/Istalri-dragon/Usenet-Arr-Stacks-Guide.git
cd Usenet-Arr-Stacks-Guide
cp .env.example .env
nano .env
```

Three details in here trip up almost everyone. Go slow on these — they're the
difference between "it works" and a day lost to a mystery.

### Detail 1 — PUID / PGID (don't hardcode 1000)

Every guide writes `PUID=1000` like it's a law of physics. It's not — it's *your*
user's ID, and it might not be 1000 (but it probably is...). Find yours instead
of guessing:

```bash
id
# uid=1000(you) gid=1000(you) groups=1000(you),...
```

Use whatever `id` prints. On a normal Linux box or VM it's usually `1000`. On
**Unraid** it's `99` / `100`. On **Synology** it's something else entirely — you
have to check. This is what lets the apps write files your user can actually
read; get it wrong and you get permission errors (there's a worked example of
exactly that error in [troubleshooting](docs/troubleshooting.md#permission-denied--files-owned-by-the-wrong-user)).

Put your values in `.env`:

```ini
PUID=1000
PGID=1000
```

> **Note:** one service — **Seerr** — deliberately has *no* PUID/PGID. It runs as
> its own internal user (UID 1000) and ignores those variables. That's not an
> omission; it's how the Seerr image is built. Every other app uses them.

### Detail 2 — the volume-mapping colon (the line that makes hardlinks work)

Look at this line under `sabnzbd`:

```yaml
    volumes:
      - ${DATA}:/data
```

That colon splits two completely different things:

- **Left of the colon** (`${DATA}`, i.e. `/data`) is a **real path on your
  machine**.
- **Right of the colon** (`/data`) is a path that **only exists inside the
  container** — a window the container looks at your real folder through.

They happen to both read `/data` here, but they don't have to. What matters is
the *shape*: we mount the **whole** `/data` root as a single volume.

**Here's why that's the whole game.** Inside the container, `usenet/` and
`media/` are under one path (`/data`), on one filesystem — so a hardlink between
them is legal. The instant you "helpfully" split it into two lines —
`/data/usenet:/downloads` and `/data/media:/movies` — the container sees **two
separate mounts**, hardlinks become copies, and you're back to the slow,
disk-doubling failure from section 2. Same colon, opposite outcome. Almost
nobody explains this. It's why this repo mounts `${DATA}:/data` and never splits
it.

### Detail 3 — the shared network (apps find each other by name)

Everything in one Compose file automatically shares a private network, and the
apps find each other by **service name** — no IP addresses needed *between* apps.

So when we wire Radarr to its download client later, the host is `sabnzbd`, not
an IP. Radarr reaches Prowlarr at `http://prowlarr:9696`, Bazarr reaches Sonarr
at `http://sonarr:8989`, and so on.

Rule of thumb for the rest of the guide:

- **App-to-app** (inside the stack): use the **service name** — `sabnzbd`,
  `prowlarr`, `radarr`, `sonarr`. No IP, no `SERVER_IP`.
- **You, in a browser:** use `http://SERVER_IP:port`.

You *can* technically use the IP and it'll work — but if your server's address
isn't fixed, then when it changes you'd have to update it in every app. Service
names avoid that entirely. (You should still give your server a static IP or a
DHCP reservation regardless.)

So when Radarr asks for its download client's host, the answer is `sabnzbd` —
*not* `localhost` (inside the container that points at Radarr itself). Your
server's LAN IP would work too, but the service name is the cleaner choice.

---

## 5. Bring it up for the first time

With `.env` filled in and the data folders created:

```bash
docker compose up -d      # start everything in the background
docker compose ps         # see what's running
```

`docker compose ps` should list every service as `running` (Jellyfin may take a
minute). If a container is restarting in a loop, read its logs — that's where
the real error is:

```bash
docker compose logs -f sabnzbd     # or radarr, sonarr, etc. Ctrl-C to stop.
```

Now open each app in your browser at `http://SERVER_IP:PORT` (see the
[port reference](#appendix-port-reference)). The rest of the guide is **wiring
these together in the right order** — the order matters because each app needs
the one before it to already exist.

> **Update later:** to pull new versions, `docker compose pull && docker compose
> up -d`. Consider [Watchtower] or [Diun] if you want notifications, but manual
> is safer for a media stack you care about.

[Watchtower]: https://containrrr.dev/watchtower/
[Diun]: https://crazymax.dev/diun/

---

## 6. SABnzbd — the download client (and what you must bring yourself)

**Web UI:** `http://SERVER_IP:8080`

This is the thing that actually downloads. And here's the honest part most guides
gloss over: **you have to bring your own sources. A guide cannot hand them to
you, and anyone who pretends the download client works out of the box is setting
you up to be stuck.**

For Usenet you need **two** things, both from you:

1. **A paid Usenet provider** — this is *who you actually download from* (the
   servers that store the files). Usenet providers are a paid service; there's no
   working free tier worth using. Examples of the *category*: a "block" account
   (pay for a chunk of data) or an "unlimited" monthly plan. Pick based on
   retention, connections, and current sales — that choice is yours to research.
2. **At least one indexer** — this is *how you search* (a searchable catalogue of
   what's on Usenet). Separate from the provider. Some are invite-only; some are
   open signups.

Neither comes with SABnzbd. It's just how Usenet works, and knowing it up front
saves you an hour of "why does searching find nothing."

> **My take:** For a provider I use **Eweka** and **Thundernews**, picking
> whichever has the better sale that year. For an indexer I use **NZBGeek**
> exclusively — I've never felt the need for more with a proper automation stack.
> Don't bother with the free indexers. Invite-only ones, in my opinion, are more
> interested in false scarcity and being "exclusive" than in providing a better
> service; I've tried a few and didn't think they were worth the hassle.

### First-run wizard

On first load SAB runs a setup wizard. You can click through the defaults; the
parts that matter:

**1. Add your provider (Server):**
`Settings → Servers → Add Server`. Enter the hostname (e.g.
`news.thundernews.com`), port (usually `563` for SSL), username, password, and
connection count from *your provider's* dashboard. Leave **SSL on**. Click
**Test Server** — you want a green success.

If it fails, check your info is right: make sure there are no rogue spaces or
typos, double-check your password, and if that all checks out, contact their
support and check SAB's logs to see if they have more detail on *why* it failed
rather than just *that* it did.

**2. Set the folders (this is the part that ties into hardlinks):**
`Settings → Folders`:

- **Temporary Download Folder** (incomplete): `/data/usenet/incomplete`
- **Completed Download Folder**: `/data/usenet`

Both are the **container** paths (remember Detail 2 — inside the container your
data root is `/data`). Because completed downloads land under `/data/usenet` and
your library is `/data/media`, both under the same mount, Radarr/Sonarr will be
able to hardlink. If you point SAB's completed folder somewhere *outside*
`/data`, hardlinks break — this is a common self-inflicted wound.

**3. Create two categories:**
`Settings → Categories`. Add:

| Category | Folder |
|----------|--------|
| `movies` | `movies` |
| `tv`     | `tv`    |

A relative folder like `movies` becomes `/data/usenet/movies` (relative to the
completed folder). These category names matter — Radarr and Sonarr will send
their downloads to the `movies` and `tv` categories so finished files land in
the right subfolder. Keep them lowercase and exactly these names to match what
we set in Radarr/Sonarr later.

You may later see errors about the folders not existing. SAB should just create
them when you download something; however, if it doesn't — or if you want to be
proactive — go create the folders manually first.

**4. Grab your API key:**
`Settings → General → API Key`. Copy it — Radarr and Sonarr need it to talk to
SAB. (Keep it private; it's a password for your download client.) Again: do not
post this anywhere online. If you do, you can regenerate it in the respective
app — but then you have to update it everywhere it was used.

That's SAB. Don't add anything to search yet — searching is the *arr's job.

---

## 7. Prowlarr — indexers in one place

**Prowlarr Web UI:** `http://SERVER_IP:9696`

### What problem Prowlarr solves (so it's not just a box we check)

You *could* add each indexer directly inside Radarr, and again inside Sonarr.
That works. But you'd be doing it twice per indexer, and again every time you add
one. Prowlarr kills that repetition: you add indexers **once**, in Prowlarr, and
it pushes them out to Radarr and Sonarr automatically.

> **My take:** If you've only got one or two indexers, honestly skip Prowlarr. At
> that scale it's not actually less work — you still have to connect Radarr and
> Sonarr *to* Prowlarr, so for one indexer that's three things to configure
> instead of two (dropping it into each app directly). It's also another point of
> failure: it can spam some indexers, or fail for no obvious reason until a
> reboot. Unless you plan on collecting indexers like Pokémon, I'd add them
> directly to Radarr/Sonarr and skip the container entirely — it's still easier
> to set up now than to bolt on later if you change your mind.

### Add your indexers

`Indexers → Add Indexer`, search for each indexer you have access to, and enter
your credentials / API key from that indexer's site. Test each one — green means
Prowlarr can reach it.

> **Note — FlareSolverr:** the compose file also starts FlareSolverr (port 8191),
> a small helper that gets Prowlarr past Cloudflare "are you human" challenges on
> some public indexers. Add it in Prowlarr under `Settings → Indexers → Add Proxy
> → FlareSolverr` with host `http://flaresolverr:8191`, then tag any indexer that
> needs it. Most Usenet indexers (NZBGeek included) don't, so this is optional —
> if you don't want it, you can delete the `flaresolverr` service from the
> compose file.

### Connect Prowlarr to Radarr and Sonarr

This is the step that makes the "add once" bit work:

`Settings → Apps → Add App → Radarr`:

- **Prowlarr Server:** `http://prowlarr:9696`
- **Radarr Server:** `http://radarr:7878`
- **API Key:** Radarr's API key (in Radarr: `Settings → General → API Key`)

Repeat for **Sonarr** (`http://sonarr:8989`). Note the **service names** again —
`radarr`, `sonarr`, `prowlarr` — not IPs.

Once both apps are added, Prowlarr **syncs your indexers into them
automatically**. Flip over to Radarr → `Settings → Indexers` and you'll see them
appear without touching anything else.

---

## 8. Radarr + Sonarr — the handshake that causes "not importing"

**Radarr (movies):** `http://SERVER_IP:7878` · **Sonarr (TV):** `http://SERVER_IP:8989`

These two are the brains. Radarr does movies, Sonarr does TV — same ideas, so
we'll do **Radarr in full** and you repeat the pattern for Sonarr.

### Connect the download client

`Settings → Download Clients → Add → SABnzbd`:

- **Name:** `SABnzbd` (anything)
- **Host:** `sabnzbd`  ← the **service name** (not `localhost` — inside the container that points at Radarr itself; the LAN IP works but the name is cleaner)
- **Port:** `8080`
- **API Key:** the SAB API key you copied in section 6
- **Category:** `movies` (in Sonarr, set this to `tv`)

Click **Test** → green. Save. (Indexers should already be here, pushed in by
Prowlarr in the last step. If they're not, re-check the Prowlarr → Apps step.)

### Set the root folder

`Settings → Media Management → Root Folders → Add Root Folder`:

- **Radarr:** `/data/media/movies`
- **Sonarr:** `/data/media/tv`

This is where your organised library lives — the same `media/` folder Jellyfin/
Plex will read.

### The handshake (this is the section)

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

### See the failure on purpose

It's worth breaking this once so you can *recognise* it. In
[docs/troubleshooting.md](docs/troubleshooting.md#it-downloads-but-nothing-shows-up-not-importing)
there's a walk-through of the deliberately-broken state: the download completes,
Radarr's **Activity → Queue** shows the item stuck with a warning, and the
**System → Events / log** shows the actual message (e.g. a "not a preferred word
/ no files found eligible for import" or a permissions line). Once you've seen
what it looks like on *your* screen, "not importing" stops being scary and
becomes a two-minute fix.

### Do Sonarr

Same three steps: add SABnzbd (host `sabnzbd`, category `tv`), confirm indexers
synced from Prowlarr, set root folder `/data/media/tv`. That's it.

### Quick end-to-end test

In Radarr, add a movie you have the rights to, and hit **Search**. Watch:
Radarr grabs a release → it appears in SABnzbd downloading → on completion it
lands in `/data/media/movies`, renamed. If that loop closes, your core pipeline
works. If it doesn't, the [troubleshooting page](docs/troubleshooting.md) is
organised by exactly where it broke.

---

## 9. Media server — Jellyfin vs Plex (both finished)

This is the **only real fork** in the whole stack. Everything before it and
everything after it is identical either way — so here are **both**, finished, and
then we rejoin immediately.

Pick one. You can run both, but there's rarely a reason to.

- **Jellyfin** — slightly less feature-rich, but a more "free" platform.
- **Plex** — more features, but paid, and has some built-in ads.

> **My take:** You get what you pay for. Jellyfin is a nice product but, for me at
> least, it's still missing some features Plex has. I have a Plex Pass from back
> when it was cheap (it costs a lot more now). So if you want to save a buck, try
> Jellyfin — you may never notice what you don't know is missing. For me, though,
> I run both, keep an eye on Jellyfin's progress, and don't think it's quite on
> par with Plex yet.

**Choosing in the compose file:** Jellyfin ships **enabled** and Plex ships
**commented out**. To run Plex instead, comment out the whole `jellyfin:` block
and **uncomment** the `plex:` block. (Commenting a line out means putting a `#` in
front of it, which makes Docker ignore that part of the file — it's also how you
leave notes so you, or someone looking at it six months later, know what it does.)

### Option A — Jellyfin

**Web UI:** `http://SERVER_IP:8096`

Jellyfin is **fully local**: no account, no signing in to anyone's servers, and —
importantly — **hardware transcoding is free**.

In the web GUI:

1. Create your admin user (local, stays on your box).
2. **Add libraries** → `Movies` pointing at `/data/media/movies`, and `Shows`
   pointing at `/data/media/tv`. These are the **container** paths.
3. Let it scan.

**Hardware transcoding:** the compose file gives Jellyfin this line:

```yaml
    devices:
      - /dev/dri:/dev/dri
```

That hands your Intel or AMD iGPU to Jellyfin so it can transcode without melting
your CPU. In Jellyfin: `Dashboard → Playback → Transcoding` → set **Hardware
acceleration** to **VAAPI** (Intel/AMD) or **Intel QuickSync** and enable the
codecs. If your box has **no compatible GPU**, remove the `/dev/dri` device line
from the compose or the container won't start. (An **NVIDIA** GPU is wired up
differently — via the NVIDIA Container Toolkit and `runtime: nvidia` rather than
`/dev/dri`; see the Jellyfin hardware-acceleration docs.)

### Option B — Plex

**Web UI:** `http://SERVER_IP:32400/web`

Plex is the more polished, works-on-every-TV option. Two differences that matter:

1. It needs a **plex.tv account** and a **claim token** to link the server. That
   token comes from <https://www.plex.tv/claim> and **expires in about four
   minutes**, so you grab it right before you start the container.
2. **Hardware transcoding on Plex is behind a paid Plex Pass.** Same `/dev/dri`
   line, but the feature Jellyfin gives you free is a subscription here.

Once you've swapped the blocks (above), finish Plex:

1. Get a fresh token from <https://www.plex.tv/claim>, put it in `.env` as
   `PLEX_CLAIM=claim-xxxxxxxx`.
2. `docker compose up -d` **immediately** (before the token expires).
3. Open `http://SERVER_IP:32400/web`, sign in, and **add libraries** →
   `/data/media/movies` and `/data/media/tv`.

> Plex uses `network_mode: host` (it needs a spread of ports for discovery and
> streaming), which is why its port isn't in the `ports:` list like the others.

### Reconverge

Whichever you chose, the request layer next points at it the **same way**. Back
to one path.

---

## 10. Seerr — the request layer

**Web UI:** `http://SERVER_IP:5055`

Seerr is the nice front door: users search for something, click **Request**, and
it flows into Radarr or Sonarr automatically. It's what turns "SSH in and add a
movie in Radarr" into "click a poster."

### A currency beat worth saying out loud

If you're following older guides you'll see them tell you to choose between
**Overseerr** and **Jellyseerr**. **That choice no longer exists.** In early 2026
those two projects **merged into a single app called Seerr** — one image, works
with Plex, Jellyfin, *and* Emby. If a guide is still framing it as
"Overseerr vs Jellyseerr," that's your signal it hasn't been updated. This stack
uses `ghcr.io/seerr-team/seerr:latest`.

*(Migrating an existing Overseerr/Jellyseerr? Your old config generally carries
over — see Seerr's official migration guide at <https://docs.seerr.dev/migration-guide/>.
This guide covers a fresh install.)*

### Set it up

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
     section 12 you'll have good ones)
4. **Connect Sonarr** the same way (`http://sonarr:8989`, root `/data/media/tv`).

### Close the loop

Do one real request end-to-end so you *see* the whole system work: request in
Seerr → Radarr grabs it → SABnzbd downloads → hardlink import into
`/data/media` → it appears in Jellyfin/Plex. When that loop closes, you have a
complete automated media pipeline.

---

## 11. Bazarr — subtitles (optional)

**Web UI:** `http://SERVER_IP:6767`

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

## 12. Quality profiles — the payoff

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

### Option A — Profilarr (modern GUI, recommended for new installs)

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

### Option B — Recyclarr (classic, config-as-code)

The lightweight, set-and-forget option: it reads a `recyclarr.yml` and pushes
TRaSH profiles on a schedule. You edit YAML, but it's the "config as code" crowd's
favourite and it's tiny.

It's in `docker-compose.yml` **commented out**. To use it instead of Profilarr:
comment out the `profilarr:` block, uncomment `recyclarr:`, put your
`recyclarr.yml` in `${APPDATA}/recyclarr`, then run
`docker compose up -d recyclarr`. Start from the templates in the
[Recyclarr docs](https://recyclarr.dev/).

### Assign it

Whichever you used: assign the new profile to your movies (Radarr) and shows
(Sonarr) — for existing items, and as the default for new ones. Now Radarr is
picky in exactly the way you want. That's the difference between a library that
works and one that gets you exactly what you want.

---

## 13. Wrap-up + what's next

Recap the map: request in **Seerr** → **Radarr / Sonarr** → **SABnzbd** →
**hardlinked** into your library → playing in **Jellyfin / Plex**, with
**Bazarr** subtitles and **Profilarr** quality profiles doing their jobs.

**The one thing this guide didn't do** is make your setup reachable from outside
your house. That's **remote access + a reverse proxy**, and it's its own guide
because doing it wrong is a security footgun (you do *not* want SABnzbd exposed
raw to the internet). Until then, everything here is safe on your local network.

If something broke, the [troubleshooting page](docs/troubleshooting.md) is
organised by exactly where. On a NAS, start with [docs/nas.md](docs/nas.md) —
99% of NAS problems are the two things that differ there (paths and PUID/PGID).

---

## Appendix: port reference

Open each in a browser at `http://SERVER_IP:PORT`.

| App          | Port  | Purpose                                    |
|--------------|-------|--------------------------------------------|
| SABnzbd      | 8080  | Usenet download client                     |
| Prowlarr     | 9696  | Indexer manager                            |
| FlareSolverr | 8191  | Cloudflare solver (used by Prowlarr)       |
| Radarr       | 7878  | Movies                                     |
| Sonarr       | 8989  | TV                                         |
| Bazarr       | 6767  | Subtitles                                  |
| Jellyfin     | 8096  | Media server (default)                     |
| Plex         | 32400 | Media server (alternative; `/web`)         |
| Seerr        | 5055  | Requests                                   |
| Profilarr    | 6868  | Quality profiles (default)                 |

**Internal host names** (app-to-app, no IP): `sabnzbd`, `prowlarr`,
`flaresolverr`, `radarr`, `sonarr`, `bazarr`, `jellyfin`, `seerr`, `profilarr`.

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
