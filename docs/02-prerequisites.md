# 2 · Prerequisites — install Docker, sort out storage

**[← Prev: Folder structure](01-folder-structure.md)** · [Home](../README.md) · **Next: [Docker Compose base →](03-docker-compose.md)**

---

If you're on a clean, brand-new machine, you literally can't skip these steps.
There's no Docker to run the containers and nowhere for the files to go.

## Install Docker

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

## Storage — where `/data` actually lives

`/data` needs to be **one filesystem**, because that's what makes hardlinks
legal (see [Folder structure](01-folder-structure.md)).

- **Single-disk box or VM:** nothing to do — `/data` is just a folder on your
  main disk. Fine for most people.
- **Separate media drive:** mount the drive, and put `/data` on it (or make
  `/data` a symlink/bind onto it). The critical part is that `usenet/` and
  `media/` end up on the **same** mount.
- **NAS (Unraid / Synology / TrueNAS):** the paths and the hardlink caveats are
  different enough to have their own page — see [Running on a NAS](nas.md).

## The one thing nobody tells beginners

To open any of these apps, you type the **server's IP address and a port number**
into a browser. That's it. If your server's IP is `192.168.1.50`, then Radarr is
at `http://192.168.1.50:7878`, SABnzbd at `http://192.168.1.50:8080`, and so on.

Find your server's IP with:

```bash
hostname -I | awk '{print $1}'
```

You'll do this a dozen times. If you've never opened a web UI by typing
`IP:port`, that's the whole trick — there's no app to download, it's just a web
page your server is serving. Throughout these pages, wherever you see
`SERVER_IP`, substitute that address. (The full list is in the
[port reference](../README.md#appendix-port-reference).)

---

**[← Prev: Folder structure](01-folder-structure.md)** · [Home](../README.md) · **Next: [Docker Compose base →](03-docker-compose.md)**
