# 7 · Media server — Jellyfin or Plex

**[← Prev: Radarr + Sonarr](06-radarr-sonarr.md)** · [Home](../README.md) · **Next: [Seerr →](08-seerr.md)**

---

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

## Option A — Jellyfin

**Web UI:** `http://SERVER_IP:8096`

Find your `SERVER_IP`:

```bash
hostname -I | awk '{print $1}'
```

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

## Option B — Plex

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


## Accessing it from outside your network

Everything above is **local-only**, which is the safe default. Reaching your media
server from outside the house is a bigger topic — its own guide — but here's the
short version per platform:

- **Plex** authenticates through plex.tv, so you turn on **Settings → Remote
  Access** (it forwards its port, `32400`, via UPnP or a manual port-forward).
  One 2025 change to know: **remote playback of your own media now requires a paid
  subscription** — a **Plex Pass** on the server account covers everyone who uses
  it, or each user can have a **Plex Pass / Remote Watch Pass**. Port-forwarding
  alone no longer gets you free remote streaming.
- **Jellyfin** has no relay service, so remote access is more hands-on: a
  **reverse proxy** (with HTTPS) plus **Dynamic DNS** to follow your changing
  public IP. More work, but free.

Whichever you use, only ever expose the **media server** — **never port-forward
SABnzbd, Radarr, Sonarr, Prowlarr, or the rest.** They have weak or no
authentication, and putting them on the open internet is a genuine security hole.
(The safe way to reach those remotely is a reverse proxy with authentication — a
topic for that separate guide.)

## Reconverge

Whichever you chose, the request layer next ([Seerr](08-seerr.md)) points at it
the **same way**. Back to one path.

---

**[← Prev: Radarr + Sonarr](06-radarr-sonarr.md)** · [Home](../README.md) · **Next: [Seerr →](08-seerr.md)**
