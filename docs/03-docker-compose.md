# 3 · The Docker Compose file

**[← Prev: Prerequisites](02-prerequisites.md)** · [Home](../README.md) · **Next: [SABnzbd →](04-sabnzbd.md)**

---

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
> to it. This repo ships a single combined file because it sidesteps all
> of that — which is why it's the path I'd point a first-timer to.

## Get the files onto your server

First get `docker-compose.yml` onto your server — two ways, pick whichever's
easier. (You'll create and fill in `.env` in the "Fill in your `.env`" section
below, so don't worry about it here.)

### Option A — copy-paste (no git needed)

1. Make a folder and go into it:
   ```bash
   mkdir arr-stack && cd arr-stack
   ```
2. Create the compose file. Open [`docker-compose.yml`](../docker-compose.yml)
   here on GitHub and hit the **copy raw file** button (the little clipboard icon
   at the top-right of the file view, or open **Raw** and select-all). Then on
   your server:
   ```bash
   nano docker-compose.yml
   ```
   Paste, then save and exit: **Ctrl+O**, **Enter**, **Ctrl+X**.

### Option B — git clone (if you already use git)

```bash
git clone https://github.com/Istalri-dragon/Usenet-Arr-Stacks-Guide.git
cd Usenet-Arr-Stacks-Guide
```

---

Three details in here trip up almost everyone. Go slow on these — they're the
difference between "it works" and a day lost to a mystery.

## PUID / PGID

Every guide writes `PUID=1000` like it's the only possible option. Your ID may 
not be 1000 (but it probably is...). Find yours instead of guessing:

```bash
id
# uid=1000(you) gid=1000(you) groups=1000(you),...
```

Use whatever `id` prints for your user. On a normal Linux box or VM it's usually `1000`. 
On **Unraid** it's `99` / `100`. On **Synology** it's something else entirely — you
have to check. This is what lets the apps write files your user can actually
read; get it wrong and you get permission errors (there's a worked example of
exactly that error in [Troubleshooting](troubleshooting.md#permission-denied--files-owned-by-the-wrong-user)).

Put your values in `.env`:

```ini
PUID=1000
PGID=1000
```

> **Note:** one service — **Seerr** — deliberately has *no* PUID/PGID. It runs as
> its own internal user (UID 1000) and ignores those variables. That's not an
> omission; it's how the Seerr image is built. Every other app uses them.

## The volume mapping

Look at this line under `sabnzbd`:

```yaml
    volumes:
      - ${DATA}:/data
```

First, `${DATA}` isn't a literal path — it's a **variable**. Docker Compose reads
your `.env` file and fills these in at runtime: because your `.env` has
`DATA=/data`, Compose swaps `${DATA}:/data` for `/data:/data` before it does
anything. The `${PUID}`, `${PGID}`, `${TZ}`, and `${APPDATA}` from earlier work
the same way — all pulled from `.env`. (That's the whole reason you only ever edit
`.env`, never the compose file itself.)

The reason to do it this way is you only have to edit one line in the .env file now
instead of every volume mapping in the compose file. Also if you add more compose files
later you can simply reuse the variable rather than hard-coding the path or creating a
new one.

Now — that colon splits two completely different things:

- **Left of the colon** (`${DATA}`, i.e. `/data`) is a **real path on your
  machine**.
- **Right of the colon** (`/data`) is a path that **only exists inside the
  container** — a window the container looks at your real folder through.

They happen to both read `/data` in this example, but they don't have to. 
What matters is the *shape*: we mount the **whole** `/data` root as a single volume.

**Here's why that's important.** Inside the container, `usenet/` and
`media/` are under one path (`/data`), on one filesystem — so a hardlink between
them is legal. The instant you "helpfully" split it into two lines —
`/data/usenet:/downloads` and `/data/media:/movies` — the container sees **two
separate mounts**, hardlinks become copies, and you're back to the slow,
disk-doubling failure from [the folder-structure page](01-folder-structure.md).
Same colon, opposite outcome. 

## The container network

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
DHCP reservation regardless, but that's for accessing it via a browser and such.)

So when Radarr asks for its download client's host, the answer is `sabnzbd` —
*not* `localhost` (inside the container that points at Radarr itself). Your
server's LAN IP would work too, but the service name is the cleaner choice.

---

## Fill in your `.env`

Everything the stack needs lives in one small file, `.env`, sitting next to your
`docker-compose.yml`. Create it from the template — use whichever matches how you
got the files:

- **Copy-paste (no git):** open [`.env.example`](../.env.example) on GitHub, copy
  all of it, then paste into a new file and save:
  ```bash
  nano .env
  ```
- **Cloned the repo:** the template is already there, so just copy it:
  ```bash
  cp .env.example .env
  nano .env
  ```

Then set each value — there are only a handful:

| Variable | Set it to |
|----------|-----------|
| `PUID` / `PGID` | Run `id` and use the `uid=` / `gid=` numbers it prints (**Detail 1** above explains why). Usually `1000` / `1000`; **Unraid** is `99` / `100`; **Synology** varies. |
| `TZ` | Your timezone as a *TZ database* name — e.g. `America/New_York`, `Europe/London`, or `UTC`. ([Full list.](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones)) It only affects timestamps in logs and schedules. |
| `DATA` | The shared data root from [page 1](01-folder-structure.md). On a normal box leave it `/data`; on a NAS point it at your share (e.g. `/mnt/user/data`) — see [Running on a NAS](nas.md). |
| `APPDATA` | Where each app stores its own config (small, frequently written). `/appdata` is fine; on a NAS use something like `/mnt/user/appdata`. |
| `PLEX_CLAIM` | Leave **blank** unless you're running Plex instead of Jellyfin — then grab a token as described on [the media-server page](07-media-server.md#option-b--plex). |

Save and exit (**Ctrl+O**, **Enter**, **Ctrl+X**). This is the only file you edit —
the compose file pulls every one of these values from it.

---

## Bring it up for the first time

With `.env` filled in and the [data folders](01-folder-structure.md) created:

```bash
docker compose up -d      # start everything in the background
docker compose ps         # see what's running
```

On this **first** start, a couple of containers (Seerr, Profilarr) may be stuck
restarting — that's the permissions gotcha below, not a real problem. Fix it once.

### Set folder ownership (do this once)

Docker creates any missing folders as **root**, and your shared `/data` root is
yours to own as well. But most of these apps run as **you** (your `PUID`/`PGID`),
not root — so when a folder is root-owned, the app can't write to it, and you get
the classic first-run failures: SABnzbd's *"error accessing"*, Seerr's
*"permission denied"*, containers stuck restarting. Hand the folders to your user:

```bash
source .env      # loads your DATA and APPDATA paths from .env
sudo chown -R "$(id -u):$(id -g)" "$DATA" "$APPDATA"
docker compose restart
```

`$(id -u):$(id -g)` is just your own user's numbers — the same ones you put in
`PUID`/`PGID`. (Seerr is a special case: it always runs as UID **1000** no matter
your `PUID`, so if your `id` isn't 1000, also run
`sudo chown -R 1000:1000 "$APPDATA/seerr"`. See
[Troubleshooting](troubleshooting.md#permission-denied--files-owned-by-the-wrong-user).)

Now `docker compose ps` should list every service as `running` (Jellyfin may take
a minute). If one is *still* restarting, read its logs — that's where the real
error is:

```bash
docker compose logs -f sabnzbd     # or radarr, sonarr, etc. Ctrl-C to stop.
```

Now open each app in your browser at `http://SERVER_IP:PORT` (see the
[port reference](../README.md#appendix-port-reference)). The rest of the guide is
**wiring these together in the right order** — the order matters because each app
needs the one before it to already exist.

> **Update later:** to pull new versions, `docker compose pull && docker compose
> up -d`. Consider [Watchtower](https://containrrr.dev/watchtower/) or
> [Diun](https://crazymax.dev/diun/) if you want notifications, but manual is
> safer for a media stack you care about.

---

**[← Prev: Prerequisites](02-prerequisites.md)** · [Home](../README.md) · **Next: [SABnzbd →](04-sabnzbd.md)**
