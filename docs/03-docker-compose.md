# 3 · The Docker Compose base — the three details everyone skips

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

```bash
git clone https://github.com/Istalri-dragon/Usenet-Arr-Stacks-Guide.git
cd Usenet-Arr-Stacks-Guide
cp .env.example .env
nano .env
```

Three details in here trip up almost everyone. Go slow on these — they're the
difference between "it works" and a day lost to a mystery.

## Detail 1 — PUID / PGID (don't hardcode 1000)

Every guide writes `PUID=1000` like it's the only possible option. Your ID may 
not might not be 1000 (but it probably is...). Find yours insteadof guessing:

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

## Detail 2 —  volume-mapping (the line that makes hardlinks work)

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
disk-doubling failure from [the folder-structure page](01-folder-structure.md).
Same colon, opposite outcome. Almost nobody explains this. It's why this repo
mounts `${DATA}:/data` and never splits it.

## Detail 3 — the shared network (apps find each other by name)

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

## Bring it up for the first time

With `.env` filled in and the [data folders](01-folder-structure.md) created:

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
[port reference](../README.md#appendix-port-reference)). The rest of the guide is
**wiring these together in the right order** — the order matters because each app
needs the one before it to already exist.

> **Update later:** to pull new versions, `docker compose pull && docker compose
> up -d`. Consider [Watchtower](https://containrrr.dev/watchtower/) or
> [Diun](https://crazymax.dev/diun/) if you want notifications, but manual is
> safer for a media stack you care about.

---

**[← Prev: Prerequisites](02-prerequisites.md)** · [Home](../README.md) · **Next: [SABnzbd →](04-sabnzbd.md)**
