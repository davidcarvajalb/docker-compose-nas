---
name: bazarr-knowledge
description: Background knowledge for working with Bazarr, subtitles, SubGen/Whisper, and Jellyfin media. Apply when the user mentions Bazarr, subtitles, Whisper, SubGen, subtitle providers, or Jellyfin subtitle issues.
user-invocable: false
---

# Bazarr & Subtitle Pipeline Knowledge

## Bazarr API Access

- **API key location:** `bazarr/config/config/config.yaml` under `auth:` > `apikey:`
- **Access pattern:** `docker exec bazarr curl -s -H "X-API-KEY: <key>" http://127.0.0.1:6767/bazarr/api/<endpoint>`
- **Config changes** to `config.yaml` require `docker restart bazarr` to take effect

## Key API Endpoints

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/system/settings` | GET | All settings (providers, scores, frequency) |
| `/api/system/languages/profiles` | GET | Language profiles |
| `/api/episodes?seriesid[]=<id>` | GET | Episodes for a series with missing_subtitles |
| `/api/episodes/subtitles` | PATCH | Trigger subtitle search for one episode. Body: `{episodeid, seriesid, language, forced:"false", hi:"false"}` |
| `/api/episodes/wanted?length=N` | GET | Wanted queue (paginated) |
| `/api/movies/wanted?length=N` | GET | Wanted movies queue |
| `/api/series` | GET | All series |

## SubGen (Whisper AI)

- **Container:** `subgen`
- **GPU:** RTX 3070 Ti (8GB VRAM), uses CUDA
- **Model:** large-v3 or large-v3-turbo
- **Transcription time:** ~2-3 min per episode
- **Models stored in:** `subgen/models/`
- **Whisper subtitle scores:** ~61% for episodes, ~33% for movies
- **Bazarr minimum_score must be ≤ 50** to accept Whisper results (default is 90 — this WILL block Whisper)

## Media Paths

| Location | Host Path | Container Path |
|---|---|---|
| TV Shows | `/mnt/HardDrive/media/tv/` | `/data/tv/` |
| Movies | `/mnt/HardDrive/media/movies/` | `/data/movies/` |
| `DATA_ROOT` | `/mnt/HardDrive/media` | `/data` |

## ffprobe (Checking Embedded Subs)

Path inside jellyfin container: `/usr/lib/jellyfin-ffmpeg/ffprobe`

```bash
docker exec jellyfin /usr/lib/jellyfin-ffmpeg/ffprobe -v error -select_streams s \
  -show_entries stream=index,codec_name,codec_type:stream_tags=language,title \
  -of compact "/data/tv/<path>"
```

Empty output = no embedded subtitles.

## Gemini Translation

- Configured in Bazarr Settings > Subtitles > Translating
- **Model IDs differ from marketing names** — use the API name:
  - `gemini-2.0-flash` (stable, works until June 2026)
  - `gemini-3-flash-preview` (newer, preview)
  - NOT `gemini-3-flash` (404 error)

## Common Gotchas

- **Adaptive searching:** When enabled, Bazarr skips episodes it already tried, waiting weeks. Disable temporarily to clear large queues.
- **All providers throttled:** OpenSubtitles free = 20 downloads/day. Whisper has no limits but is slower.
- **BluRay rips often lack embedded subs** — this is normal, not a download issue.
- **Config edits via file:** Bazarr does NOT hot-reload config.yaml. Must `docker restart bazarr`.
- **Bazarr API is blocking:** PATCH to `/episodes/subtitles` waits for the provider to finish. For Whisper, this means ~3 min per call.
