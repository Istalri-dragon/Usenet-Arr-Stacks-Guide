# 1 · Folder structure

[← Home](../README.md) · **Next: [Prerequisites →](02-prerequisites.md)**

---

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

## Why one root?

When Radarr or Sonarr finishes a download, it moves the file from `usenet/` into
`media/`. You want that move to be a **hardlink**, not a copy.

A hardlink is a second *name*/*link* for the exact same file on disk. It's created
instantly and it uses **zero** extra space — the download and the library entry
are literally the same bytes with two names. When SAB later cleans up its copy,
the file stays put in your library, untouched.

But hardlinks have **one iron rule: both names have to be on the same
filesystem.** Keep downloads and media as siblings under one `/data` root on one
disk, and every import is instant.

## What "getting it wrong" looks like

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
  pool, not a true cache — see [Running on a NAS](nas.md).)

You can *see* the difference. A correct import is instant and `df -h` doesn't
move. A broken one shows a progress bar and the free space dropping. There's a
side-by-side of exactly this in [Troubleshooting](troubleshooting.md#hardlinks-silently-became-copies).

This layout follows the [TRaSH Guides](https://trash-guides.info/) recommendation —
the reference the whole community trusts.
Technically the root folder can be called anything, but `data` (with `media`
under it) is the recommended convention.

## Create it once

On basically any standard Linux distro:

```bash
mkdir -p /data/usenet/{incomplete,movies,tv}
mkdir -p /data/media/{movies,tv}
```

(If `/data` is on a separate drive or NAS share, create this structure *on that
mount* — the point is that everything under `/data` is one filesystem. More on
that in [Prerequisites](02-prerequisites.md).)

On Unraid you'll need to create the root folder/share under **Shares** first
(who'd have guessed), then you can create the subfolders from Windows if that's
more familiar. See [Running on a NAS](nas.md).

---

[← Home](../README.md) · **Next: [Prerequisites →](02-prerequisites.md)**
