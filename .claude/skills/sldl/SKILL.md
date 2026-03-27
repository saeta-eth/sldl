---
name: sldl
description: Use when an AI assistant needs to download music from Soulseek using sldl, build sldl commands, manage a music library, or answer questions about sldl options. Covers programmatic invocation, input types (Spotify, YouTube, CSV, Bandcamp, MusicBrainz, Soulseek links, text queries), download modes, file conditions, name formatting, configuration profiles, on-complete actions, and troubleshooting.
---

# sldl — AI Assistant Reference

sldl is a CLI music downloader for the Soulseek P2P network. It accepts various inputs, searches Soulseek for matching files, ranks results by quality, and downloads the best match.

Credentials are assumed to be stored in `sldl.conf` — never ask the user for their password in conversation.

## Invocation Rules

**NEVER use `-t` / `--interactive`** — it requires keyboard input. Always use non-interactive mode (the default).

**Always preview before downloading** — show the user what will happen, then download after confirmation.

**Always specify `-p <path>`** — never let sldl download to the current working directory by accident.

**Use `--no-progress`** — suppresses progress bars that clutter your stdout.

## Workflow

### 1. Determine What the User Wants

| User wants | Mode | Key flags |
|------------|------|-----------|
| A specific song | Normal (default) | (none) |
| A specific album | Album | `-a` |
| All songs by an artist | Aggregate | `-g` |
| All albums by an artist | Album Aggregate | `-ag` |
| Songs from a playlist | Normal | (none, pass URL) |
| Albums from a playlist | Album | `-a` (pass URL) |

### 2. Preview

```bash
# Dry-run: what tracks would be processed (no network)
sldl "Artist - Album" -a --print tracks --no-progress

# Search Soulseek and show what would be downloaded (no download)
sldl "Artist - Song" --print results --no-progress

# Machine-readable JSON output
sldl "Artist - Song" --print json --no-progress

# All results sorted by quality as JSON
sldl "Artist - Song" --print json-all --no-progress
```

Show preview output to user. Ask for confirmation before proceeding.

### 3. Download

```bash
sldl "Artist - Song" -p ~/Music --no-progress
```

### 4. Verify

After download completes, check that files exist at the expected path. Use `--print index` to inspect download history if needed.

## Input Types

Input type is auto-detected. Override with `--input-type` if needed.

| Type | Input | Notes |
|------|-------|-------|
| **Spotify** | Playlist/album URL, `spotify-likes`, `spotify-albums` | Private playlists need credentials in config |
| **YouTube** | Playlist URL | `--youtube-key` for reliability; `--get-deleted --yt-dlp` for deleted videos |
| **CSV** | Path to `.csv` file | Auto-detects column names; use `--title-col`, `--artist-col` for custom |
| **Bandcamp** | Track/album/artist/wishlist URL | `--from-html` if Cloudflare blocks |
| **MusicBrainz** | Release, release-group, or collection URL | Release-group picks most common version |
| **Soulseek link** | `slsk://user/path` | Paths ending in `/` = album download |
| **Search string** | `"Artist - Title"` or key=value | Fallback; most common for AI use |
| **List file** | `.txt` path + `--input-type list` | One input per line with optional conditions |

### String Input (Most Common)

```bash
# Simple format: "Artist - Title" for songs, "Artist - Album" for albums (-a)
sldl "Radiohead - OK Computer" -a -p ~/Music

# Key-value format (more precise, better results)
sldl "artist=Radiohead, title=Creep, length=236" -p ~/Music

# Accepted properties: title, artist, album, length (seconds), artist-maybe-wrong, album-track-count
```

### List File Format

Write a `.txt` file for batch downloads with per-item conditions:

```ini
# input                        conditions (optional)           pref. conditions (optional)
"Artist - Song"                "format=mp3; br>128"            "br >= 320"
a:"Artist - Album"             format=flac
a:"Another Album"              strict-album=true
```

Invoke: `sldl "list.txt" --input-type list -p ~/Music --no-progress`

## Download Modes

| Mode | Flag | Behavior |
|------|------|----------|
| **Normal** | (default) | One file per track |
| **Album** | `-a` | Entire folder including non-audio files |
| **Aggregate** | `-g` | All distinct songs by artist, most-shared first |
| **Album Aggregate** | `-ag` | All distinct albums by artist, most-shared first |

Album mode auto-activates for Spotify/Bandcamp album URLs, MusicBrainz releases, or CSV rows without a title.

Aggregate modes skip tracks shared by only 1 user (configurable: `--min-shares-aggregate`).

## File Conditions

Two tiers filter and rank results:

**Required** (`--format`, `--min-bitrate`, etc.) — files not meeting these are rejected. No defaults.

**Preferred** (`--pref-format`, `--pref-min-bitrate`, etc.) — files meeting these rank higher. Defaults:

```ini
pref-format = mp3
pref-length-tol = 3          # seconds
pref-min-bitrate = 200       # kbps
pref-max-bitrate = 2500
pref-max-samplerate = 48000
pref-strict-title = true
pref-strict-album = true
```

### Condition Flags

```
--format <fmts>               Required formats: mp3,flac,ogg,wav,opus,m4a,aac,alac
--length-tol <sec>            Max length difference
--min-bitrate / --max-bitrate
--min-samplerate / --max-samplerate
--min-bitdepth / --max-bitdepth
--strict-title                File name must contain track title
--strict-artist               File path must contain artist name
--strict-album                File path must contain album name
--banned-users <list>         Comma-separated users to ignore
--strict-conditions           Reject files with unknown properties
```

All have `--pref-*` counterparts. Inline: `--cond "br>=320; format=flac"` and `--pref "format=flac; br>=800"`.

**Caution with `--strict-conditions`:** The standard Soulseek client doesn't broadcast bitrate. Enabling strict conditions with a min-bitrate filter will exclude all files from those users.

## Name Format

Template for output filenames. Variables in `{}` are replaced.

```bash
--name-format "{artist( - )title|filename}"
```

**Syntax:** `{var}` = value, `{a(sep)b}` = separator only if both non-null, `{a|b}` = fallback chain.

**Recommended templates:**
- `"{artist( - )title|filename}"` — safe default with fallback
- `"{albumartist(/)album(/)track(. )title|(missing-tags/)slsk-foldername(/)slsk-filename}"` — organized folders

**Tag variables:** `artist`, `artists`, `albumartist`, `albumartists`, `title`, `album`, `year`, `track`, `disc`, `length`

**Source variables:** `sartist`, `stitle`, `salbum`, `slength`, `uri`, `snum`, `row`/`line`

**Other:** `slsk-filename`, `slsk-foldername`, `ext`, `path`, `path-noext`, `default-folder`, `input`, `item-name`, `extractor`, `type`, `state`, `failure-reason`, `is-audio`, `artist-maybe-wrong`, `bindir`

## Configuration

Config file location (first found wins):
1. `~/.config/sldl/sldl.conf`
2. `~/AppData/Roaming/sldl/sldl.conf`
3. `$XDG_CONFIG_HOME/sldl/sldl.conf`
4. `{sldl binary dir}/sldl.conf`

```ini
username = your-username
password = your-password
path = ~/Music/sldl
pref-format = flac
fast-search = true
```

### Profiles

```ini
[lossless]
pref-format = flac,wav
pref-min-bitrate = 800
```

Activate: `sldl "..." --profile lossless`. List all: `sldl --profile help`.

### Auto Profiles

Applied automatically when conditions match:

```ini
[youtube]
profile-cond = input-type == "youtube"
path = ~/downloads/sldl-youtube
```

**Variables:** `input-type` (youtube|csv|string|bandcamp|spotify), `download-mode` (normal|aggregate|album|album-aggregate), `interactive` (bool), `album` (bool), `aggregate` (bool)

**Operators:** `&&`, `||`, `==`, `!=`, `!`

## Search Options

```
--fast-search                  Take first match meeting preferred conditions (faster)
--remove-ft                    Strip "feat."/"ft." before searching
--regex <pattern>              Remove regex from titles/artists (prefix T:/A:/L: for field-specific)
--artist-maybe-wrong           Also search without artist name
-d, --desperate                Search each field separately, then filter (slower, broader)
--yt-dlp                       Fallback to yt-dlp for tracks not found
--search-timeout <ms>          Max search time (default: 6000)
--max-stale-time <ms>          Max download time without progress (default: 30000)
```

**Rate limits:** Soulseek bans for 30 min if too many searches. Default throttle: 34 per 220 sec. Adjust with `--searches-per-time` and `--searches-renew-time`.

## Album Options

```
-a, --album                    Album download mode
--album-track-count <num>      Expected tracks (5+ = at least 5, 5- = at most 5)
--album-art <option>           default | largest | most
--album-art-only               Only download album art
--no-browse-folder             Don't browse for complete folder
--failed-album-path <path>     Move failed albums (or "delete" / "disable")
--album-parallel-search        Search multiple albums in parallel
```

## Skip & Index

```
--no-skip-existing             Re-download everything
--skip-not-found               Skip tracks not found in previous runs
--skip-music-dir <path>        Skip tracks already in music library (filename match)
--skip-check-cond              Check file conditions when skipping
--skip-check-pref-cond         Check preferred conditions when skipping
--no-write-index               Don't create index file
--index-path <path>            Custom index path ({playlist-name} variable available)
```

The index (`_index.csv`) tracks history across runs. Use `--print index` to inspect it, `--print index-failed` for failures.

## On-Complete Actions

Run commands after each download. Useful for post-processing (convert formats, move files, update libraries).

```bash
--on-complete "[prefixes:]command"
```

**Prefixes:** `1:` (success only), `2:` (failure only), `a:` (album only), `t:` (track only), `s:` (shell execute), `h:` (hidden), `r:` (read output), `u:` (update index from output)

**Chain:** `--on-complete "cmd1" --on-complete "+ cmd2"` (note `+ ` prefix with space)

**Variables:** All name-format variables, plus `{exitcode}`, `{stdout}`, `{stderr}`, `{first-exitcode}`, `{first-stdout}`, `{first-stderr}`

## Printing & Inspection

| Flag | Output | Use case |
|------|--------|----------|
| `--print tracks` | Track list (no search) | Verify input parsing |
| `--print tracks-full` | Extended track info | Debug metadata |
| `--print results` | Search results | Preview what would download |
| `--print results-full` | Results with full paths | Debug file selection |
| `--print json` | First result as JSON | Parse programmatically |
| `--print json-all` | All results as JSON | Analyze all options |
| `--print link` | slsk:// link | Get direct Soulseek link |
| `--print index` | Download index as JSON | Inspect history |
| `--print index-failed` | Failed downloads | Diagnose failures |

## NDJSON Progress

With `--progress-json`, sldl streams one JSON object per line:

```json
{"type": "trackList", "timestamp": "...", "data": {...}}
{"type": "downloadProgress", "timestamp": "...", "data": {...}}
{"type": "jobComplete", "timestamp": "...", "data": {...}}
```

Types: `trackList`, `searchStart`, `searchResult`, `downloadStart`, `downloadProgress`, `trackStateChanged`, `overallProgress`, `jobComplete`

## Common Recipes

```bash
# Download a song
sldl "Radiohead - Creep" -p ~/Music --no-progress

# Download with known length for better matching
sldl "artist=Radiohead, title=Creep, length=236" -p ~/Music --no-progress

# Download album, prefer FLAC, at least 10 tracks
sldl "Radiohead - OK Computer" -a -p ~/Music --pref-format flac --atc 10+ --no-progress

# Download Spotify playlist
sldl "https://open.spotify.com/playlist/ID" -p ~/Music --no-progress

# Download YouTube playlist with deleted video recovery
sldl "https://youtube.com/playlist/ID" --get-deleted --yt-dlp -p ~/Music --no-progress

# Preview all songs by artist available on Soulseek
sldl "artist=Radiohead" -g --print results --no-progress

# Find songs not in library
sldl "artist=Radiohead" -g --skip-music-dir ~/Music --print results --no-progress

# Download all albums by artist
sldl "artist=Radiohead" -ag -p ~/Music --no-progress

# Download album art only
sldl "Artist - Album" -a --album-art-only --album-art largest -p ~/Music --no-progress

# Batch download from list file
sldl "wishlist.txt" --input-type list -p ~/Music --no-progress
```

## Troubleshooting

| Problem | Solution |
|---------|----------|
| No results found | Try `--desperate` for broader search |
| Wrong song/album downloaded | Add `--strict-title` and/or `--strict-artist` |
| Banned by Soulseek (30 min) | Wait, then lower `--searches-per-time` |
| Download stalls | Lower `--max-stale-time` (default 30000ms) |
| Not enough album tracks | Increase `--album-track-count` tolerance |
| Previously failed tracks | Inspect with `--print index-failed` |
| Want to retry failed | Use `--no-skip-existing` or remove index |
| Slow performance | Add `--fast-search` and/or `--concurrent-downloads 4` |
| Wrong file format | Set `--format flac` (required) or `--pref-format flac` (preferred) |

## Search Tips

- **Less input is better** — provide only what's needed to uniquely identify the track
- `--remove-ft` strips "feat." artists that may not match on Soulseek
- `--regex "[\[\(].*?[\]\)]"` removes bracketed/parenthesized text (live, remix, video, etc.)
- `--regex A:.*` removes artist entirely (useful for "Various Artists")
- You can find an album by searching for one of its songs with `-a`
- Including `album=` in string input improves ranking via pref-strict-album

## CLI Shortcuts

- Combine single-char flags: `-ag` = `-a -g`
- Acronyms for long flags: `--atc` = `--album-track-count`, `--Mbr` = `--max-bitrate`
- Disable flags: `--skip-existing false`
- Equals syntax: `--format=flac`

## Wishlist / Recurring Downloads

Create a list file and config profile for repeated use:

```ini
# sldl.conf
[wishlist]
input = ~/sldl/wishlist.txt
input-type = list
index-path = ~/sldl/wishlist-index.sldl
album-parallel-search = true
```

Run: `sldl --profile wishlist --no-progress`. The index tracks what has already been downloaded across runs.
