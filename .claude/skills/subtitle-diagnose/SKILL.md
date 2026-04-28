---
name: subtitle-diagnose
description: Use when the user reports missing subtitles, asks why subs aren't working, mentions a show/movie has no subtitles, or says "subtitles are missing". Diagnoses the root cause by checking media files, Bazarr config, and provider status.
argument-hint: <show or movie name>
allowed-tools: [Read, Grep, Glob, Bash]
---

# Subtitle Diagnose

Diagnose why subtitles are missing for: $ARGUMENTS

## Steps

### 1. Find the media files

Search for the show/movie on disk. Try both TV and movies:

```bash
ls "/mnt/HardDrive/media/tv/" | grep -i "<name>"
ls "/mnt/HardDrive/media/movies/" | grep -i "<name>"
```

### 2. Check embedded subtitle streams

For each episode/file (sample a few), check if subtitles are embedded:

```bash
docker exec jellyfin /usr/lib/jellyfin-ffmpeg/ffprobe -v error -select_streams s \
  -show_entries stream=index,codec_name,codec_type:stream_tags=language,title \
  -of compact "/data/tv/<path>" 2>&1
```

If output is empty, the file has **no embedded subtitles**.

### 3. Check for external subtitle files

```bash
ls "/mnt/HardDrive/media/tv/<show>/"*.{srt,ass,sub,ssa} 2>/dev/null
```

### 4. Check Bazarr configuration

Read the API key:
```bash
grep "apikey" bazarr/config/config/config.yaml | head -1
```

Then check if the series has a language profile and what subs are missing:
```bash
# Find the series in Bazarr (check the Bazarr UI URL path for the seriesid)
docker exec bazarr curl -s -H "X-API-KEY: <key>" \
  "http://127.0.0.1:6767/bazarr/api/episodes?seriesid%5B%5D=<id>" | python3 -c "
import json, sys
data = json.load(sys.stdin)
for ep in data.get('data', []):
    ms = ep.get('missing_subtitles', [])
    if ms:
        s, e = ep.get('season'), ep.get('episode')
        langs = [m.get('name') for m in ms]
        print(f'S{s:02d}E{e:02d}: missing {langs}')
"
```

### 5. Check provider status

```bash
docker logs bazarr 2>&1 | grep -i "throttl\|All providers" | tail -10
```

### 6. Check SubGen (Whisper) status

```bash
docker ps --format '{{.Names}} {{.Status}}' | grep subgen
docker logs subgen --tail 5 2>&1
```

### 7. Check key settings

```bash
docker exec bazarr curl -s -H "X-API-KEY: <key>" \
  "http://127.0.0.1:6767/bazarr/api/system/settings" | python3 -c "
import json, sys
g = json.load(sys.stdin)['general']
print('enabled_providers:', g['enabled_providers'])
print('minimum_score:', g['minimum_score'])
print('adaptive_searching:', g['adaptive_searching'])
print('wanted_search_frequency:', g['wanted_search_frequency'])
"
```

## Common Root Causes

- **No embedded subs + no external files**: BluRay rips often lack subs. Need Bazarr/Whisper to fetch/generate them.
- **All providers throttled**: OpenSubtitles free tier = 20 downloads/day. Wait or use Whisper.
- **Whisper subs rejected**: minimum_score too high (>61). Must be ≤50 for Whisper to work.
- **No language profile assigned**: Check Bazarr series page — profile column should show "Default".
- **Adaptive searching blocking retries**: If enabled, Bazarr skips episodes it already tried for weeks.

## Report

Summarize findings clearly: what's missing, why, and what to do about it.
