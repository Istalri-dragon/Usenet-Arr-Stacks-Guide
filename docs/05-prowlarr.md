# 5 · Prowlarr — indexers in one place *(optional)*

**[← Prev: SABnzbd](04-sabnzbd.md)** · [Home](../README.md) · **Next: [Radarr + Sonarr →](06-radarr-sonarr.md)**

**Web UI:** `http://SERVER_IP:9696`

Find your `SERVER_IP`:

```bash
hostname -I | awk '{print $1}'
```

---

## What Prowlarr does

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

## Add your indexers

`Indexers → Add Indexer`, search for each indexer you have access to, and enter
your credentials / API key from that indexer's site. Test each one — green means
Prowlarr can reach it.

## Connect Prowlarr to Radarr and Sonarr

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

**[← Prev: SABnzbd](04-sabnzbd.md)** · [Home](../README.md) · **Next: [Radarr + Sonarr →](06-radarr-sonarr.md)**
