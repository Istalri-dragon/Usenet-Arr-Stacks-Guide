# 4 · SABnzbd — the download client (and what you must bring yourself)

**[← Prev: Docker Compose base](03-docker-compose.md)** · [Home](../README.md) · **Next: [Prowlarr →](05-prowlarr.md)**

**Web UI:** `http://SERVER_IP:8080`

---

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

## First-run wizard

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

Both are the **container** paths (inside the container your data root is `/data` —
see [the compose page](03-docker-compose.md#detail-2--the-volume-mapping-colon-the-line-that-makes-hardlinks-work)).
Because completed downloads land under `/data/usenet` and your library is
`/data/media`, both under the same mount, Radarr/Sonarr will be able to hardlink.
If you point SAB's completed folder somewhere *outside* `/data`, hardlinks break —
this is a common self-inflicted wound.

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

**[← Prev: Docker Compose base](03-docker-compose.md)** · [Home](../README.md) · **Next: [Prowlarr →](05-prowlarr.md)**
