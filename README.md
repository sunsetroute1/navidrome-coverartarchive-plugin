# navidrome-coverartarchive-plugin

**Album cover art for Navidrome — from MusicBrainz and the Cover Art Archive.**

A free, open-source Navidrome metadata plugin (`.ndp`) that resolves album cover
art from **MusicBrainz + the Cover Art Archive**. Works for mainstream and obscure
releases alike — anything catalogued in MusicBrainz/CAA.

| | |
| --- | --- |
| **What it does** | Resolves album cover art via MusicBrainz lookup → Cover Art Archive |
| **What it does not do** | Artist photos, lyrics, biographies, or scrobbling |
| **Cost** | Free — MusicBrainz + CAA are free; no API keys |
| **Navidrome** | v0.62.0+ (plugin system / WASM metadata agents) |
| **Download** | [`dist/coverartarchive.ndp`](dist/coverartarchive.ndp) — drop in, enable, done |

## Why this exists

The [Cover Art Archive](https://coverartarchive.org) is the canonical open
artwork repository for MusicBrainz — community-sourced, free, no API keys.
Commercial agents (Apple Music, Deezer) cover a different catalog and often miss
obscure, self-released, or P2P-sourced albums that MusicBrainz knows well.

This plugin queries MusicBrainz and CAA directly whenever Navidrome asks it for
album art. It is not inherently a “fallback” — **where you place it in the
`Agents` list controls priority** (see [Agent priority](#agent-priority)).

## Quick install

1. Download [`dist/coverartarchive.ndp`](dist/coverartarchive.ndp).
2. Copy it to your Navidrome plugins folder (`<DataFolder>/plugins`).
3. Add `coverartarchive` to your agent chain in `navidrome.toml`:

   ```toml
   Agents = "artist-nfo-metadata,coverartarchive,apple-music,deezer"
   ```

   **Order matters.** Navidrome tries agents left-to-right and uses the first
   album-art hit. Put `coverartarchive` **early** to prefer CAA art; put it
   **later** to prefer Apple Music/Deezer first.

4. Open Navidrome → **Settings → Plugins** → enable **coverartarchive**.

No restart required. Navidrome's file watcher discovers new `.ndp` files
automatically (they start disabled until you toggle them on).

## How it works

When Navidrome needs album art, it calls the plugin's `GetAlbumImages` with the
album name, artist, and optional MusicBrainz ID:

1. **Known MBID** → query CAA directly (`/release/{mbid}`, then `/release-group/{mbid}`).
2. **No MBID** → search MusicBrainz for the best matching release-group
   (`releasegroup:"Album" AND artist:"Artist"`).
3. Return the **front cover** from CAA in 250 / 500 / 1200 px plus full resolution.

Outbound HTTP is restricted to `musicbrainz.org`, `coverartarchive.org`,
`archive.org`, and its subdomains (see `manifest.json`). CAA image requests may
redirect to an Internet Archive mirror such as `ia600604.us.archive.org`;
Navidrome re-checks the allowed-host list at each redirect. Navidrome caches agent
results.

## Verify it's working

**In Navidrome:** pick an album that Apple Music/Deezer miss but MusicBrainz
knows. After enabling the plugin, art should appear in the web UI and Subsonic
clients (Symfonium, etc.). You may need to refresh or re-open the album once.

**Standalone (no Navidrome needed):**

```powershell
cd tools/harness
go run . ..\..\dist\coverartarchive.ndp   # extracts and runs plugin.wasm
```

Or point at a freshly built `plugin.wasm`. Expected output includes real CAA URLs:

```
=== GetAlbumImages: "OK Computer" by "Radiohead" ===
  RESULT: 4 image(s):
    size=250   https://coverartarchive.org/release/.../-250.jpg
    size=500   https://coverartarchive.org/release/.../-500.jpg
    size=1200  https://coverartarchive.org/release/.../-1200.jpg
    size=0     https://coverartarchive.org/release/....png
```

## Build from source

Requires Go 1.24+ (standard Go — no TinyGo). The plugin must be built against
the Navidrome PDK version matching your server (ABI compatibility).

```powershell
./build.ps1
```

Output: `dist/coverartarchive.ndp`.

Manual build details and project layout are in the sections below.

## Project layout

| Path | Purpose |
| --- | --- |
| `main.go` | Plugin source (`metadata.AlbumImagesProvider`) |
| `manifest.json` | Navidrome plugin manifest + HTTP allow-list |
| `dist/coverartarchive.ndp` | Prebuilt binary for Navidrome v0.62.0 |
| `build.ps1` | Reproducible build script |
| `tools/harness/` | Extism test runner (same WASM runtime as Navidrome) |

## Agent priority

Navidrome calls agents **in order** and **stops at the first success** for album
art. The plugin always queries MusicBrainz + CAA when it is reached; it cannot
run in parallel with other agents in the same lookup.

| Goal | Suggested `Agents` order |
| --- | --- |
| Prefer canonical CAA art when available | `coverartarchive` before `apple-music`, `deezer` |
| Prefer commercial catalogs, use CAA when they miss | `apple-music`, `deezer` before `coverartarchive` |
| CAA only (no commercial agents) | `coverartarchive` alone or with `artist-nfo-metadata` |

Example (CAA-first — recommended if you want MusicBrainz art whenever it exists):

```toml
Agents = "artist-nfo-metadata,coverartarchive,apple-music,deezer,spotify,lastfm"
```

Example (commercial-first — CAA only when iTunes/Deezer have nothing):

```toml
Agents = "artist-nfo-metadata,apple-music,deezer,coverartarchive,spotify,lastfm"
```

## Notes

- MusicBrainz rate-limits to ~1 req/sec; the plugin sends a proper
  `User-Agent`. Navidrome's cache keeps live usage well within limits.
- CAA image URLs may redirect through `archive.org` to a mirror subdomain.
  The manifest allows `archive.org` and `*.archive.org` so Navidrome can follow
  these redirects.
- Plugin ID (and agent name) is **`coverartarchive`** — derived from the
  `.ndp` filename, not the manifest display name.

## License

MIT — see [LICENSE](LICENSE).

## Links

- [Navidrome](https://www.navidrome.org/)
- [Navidrome plugin docs](https://www.navidrome.org/docs/developers/plugins/)
- [Cover Art Archive](https://coverartarchive.org/)
- [MusicBrainz](https://musicbrainz.org/)
