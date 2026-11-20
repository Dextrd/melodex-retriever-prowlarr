# Prowlarr retrieval plugin

A Melodex retrieval plugin (contract v1.0) that searches the torrent indexers you've set up in
[Prowlarr](https://prowlarr.com) and downloads with your [qBittorrent](https://www.qbittorrent.org). What you can find
is what your indexers offer; it's up to you to use ones whose music you're allowed to download.
[PandaCD](https://pandacd.io), a public tracker for Creative Commons and artist-permitted music, is a good one to start
with.

It runs beside [Melodex](https://github.com/Dextrd/Melodex) as a container of its own, which Melodex reaches over the
[retrieval plugin contract](https://github.com/Dextrd/Melodex/blob/main/docs/contracts/README.md).

## What it does

- **Search.** Asks Prowlarr for the artist and album (or the date, for concerts), and for a song also the artist and
  title. The best-named torrents have their .torrent file read, and a torrent is offered only when it has a file for
  every requested track; for a song, the file that's that song. Torrents that only come as a magnet link can't be
  read ahead, so they rank lower and are checked once qBittorrent has their file list. Between equally good matches,
  FLAC comes first, then the better-seeded.
- **Fetch.** Adds the torrent to qBittorrent stopped, with the `melodex` category and tag, turns off every file that
  wasn't asked for (the rest of the album, covers, logs), starts it, and reports progress. When the files are done,
  it copies them to staging, leaving qBittorrent to seed. A torrent that was already in qBittorrent keeps its files;
  the requested ones are added to it. Copies are reflinks where the filesystem can make them (see below).
  While it runs, the torrent also carries a `melodex-<job>` tag, so songs sharing a torrent know who's still using
  it; the tag is deleted once the fetch is done. The first time the plugin reaches qBittorrent after starting, it
  deletes any such tags left behind (by a fetch cut short, or by older versions of the plugin). The `melodex` tag stays: it
  marks the torrents Melodex added.
- **Cancel.** Removes a torrent it added, with what it downloaded. One that was already there stays.

Melodex gives up on a fetch that makes no progress for 30 minutes, and cancels it.

## Settings

| Setting | Default | Meaning |
| --- | --- | --- |
| `prowlarrUrl` | `http://prowlarr:9696` | Where this plugin reaches Prowlarr. |
| `prowlarrApiKey` | | In Prowlarr under Settings → General. Needed. |
| `qbittorrentUrl` | `http://qbittorrent:8080` | qBittorrent's Web UI. |
| `qbittorrentUsername`, `qbittorrentPassword` | | Leave empty when qBittorrent lets this plugin in without logging in. |
| `category` | `melodex` | The qBittorrent category torrents are added with. |
| `savePath` | `/downloads/melodex` | Where qBittorrent saves them, as qBittorrent sees it. |
| `downloadsFolder` | | The same folder as this plugin sees it, when it's mounted somewhere else. |
| `format` | `best` | `best` prefers FLAC and takes other formats otherwise; `flac` only lossless; `mp3` only MP3s. |
| `minSeeders` | `1` | Torrents with fewer seeders aren't offered. |
| `minScore` | `0.6` | Torrents that match less well than this aren't offered. |
| `afterwards` | `keep` | `keep` leaves the torrent seeding; `remove` takes it out, with its files, once Melodex has a copy. |
| `reflinks` | on | Copies files as reflinks, so music you keep seeding isn't stored twice. Where they can't be made, files are copied as usual. |

## Adding it to Melodex

Add this under `services:` in the compose file you run Melodex with, beside `core` and `beets-worker`, so it shares
their `staging` volume:

```yaml
  retriever-prowlarr:
    image: ghcr.io/dextrd/melodex-retriever-prowlarr:latest
    container_name: melodex-retriever-prowlarr
    user: "${PUID:-1000}:${PGID:-1000}"
    restart: unless-stopped
    volumes:
      - staging:/staging
      - ${DOWNLOADS_PATH:-./downloads}:/downloads:ro
```

`DOWNLOADS_PATH` is qBittorrent's downloads folder, which qBittorrent mounts at `/downloads` too; set it in the `.env` file beside your compose file, or write the path in its place.

Then start it with `docker compose up -d`. `latest` follows each release; to choose when it changes, use a version
such as `2` or `2.0.0` instead. On SELinux hosts with podman, give it the same `security_opt` as Melodex's own
containers.

Where qBittorrent mounts it somewhere else, say the pool at `/data` with torrents in `/data/torrents`, set **Save
folder** to the path qBittorrent sees (`/data/torrents/melodex`) and **Save folder for this plugin** to the path the
plugin sees (`/downloads/melodex`). The health check warns when **Save folder** isn't under qBittorrent's own default
save path, as qBittorrent can't write anywhere it doesn't have: fetches then fail with "Permission denied".

Then add `http://retriever-prowlarr:8080` under Settings → Plugins in Melodex (it's one of the templates under Add),
and fill in Prowlarr's API key in its settings.

## Sharing disk space with what you seed

A song you keep seeding would otherwise be on disk twice: in qBittorrent's downloads and in your library. A hardlink
can't help, because Melodex tags the library's copy, and with a hardlink that would change the file qBittorrent
seeds and break the torrent. A reflink can: it's a separate file that shares its data on disk with the original
until one of them is changed, so tagging the library's copy leaves the seeding one alone, and only the few blocks the
tags changed take space.

The plugin copies into staging as reflinks, and Melodex's import worker carries them on into the library (see
[the import worker](https://github.com/Dextrd/Melodex/blob/main/docs/import-worker.md#reflinks)). That needs:

- btrfs, XFS made with reflink support, or ZFS 2.2 or later. Not ext4, network shares, or unRAID's `/mnt/user`.
- qBittorrent's downloads, the staging volume and your library all on that same filesystem. Make `staging` a folder
  on it rather than a volume Docker manages.

**Check reflinks** in the plugin's settings tries it, with a file qBittorrent has downloaded, and says which step
works and why not. Where reflinks can't be made, everything still works, with ordinary copies.

## Tests

```bash
pip install -r requirements.txt -r requirements-dev.txt
pytest
```

The tests answer for Prowlarr and qBittorrent with fakes, so they don't need either.
