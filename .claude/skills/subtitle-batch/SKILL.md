---
name: subtitle-batch
description: Mass-trigger subtitle downloads for a series or the entire wanted queue via Bazarr API.
argument-hint: <series name> [language]
disable-model-invocation: true
allowed-tools: [Read, Grep, Bash]
---

# Subtitle Batch Process

Mass-trigger subtitle search for: $ARGUMENTS

Default language is `en` (English). Specify a language code as second argument to override.

## Steps

### 1. Read API key

```bash
grep "apikey" bazarr/config/config/config.yaml | head -1
```

### 2. Find the series

If a series name was given, search Bazarr for matching series:

```bash
docker exec bazarr curl -s -H "X-API-KEY: <key>" \
  "http://127.0.0.1:6767/bazarr/api/series" | python3 -c "
import json, sys
data = json.load(sys.stdin)
for s in data.get('data', []):
    if '<search term>'.lower() in s.get('title', '').lower():
        print(f\"ID: {s['sonarrSeriesId']} - {s['title']} - Profile: {s.get('profileId')}\")
"
```

### 3. Get missing episodes

```bash
docker exec bazarr curl -s -H "X-API-KEY: <key>" \
  "http://127.0.0.1:6767/bazarr/api/episodes?seriesid%5B%5D=<seriesid>" | python3 -c "
import json, sys
data = json.load(sys.stdin)
for ep in data.get('data', []):
    ms = ep.get('missing_subtitles', [])
    if any(m.get('code2') == '<lang>' for m in ms):
        eid = ep.get('sonarrEpisodeId')
        s, e = ep.get('season'), ep.get('episode')
        print(f'{eid}|S{s:02d}E{e:02d}')
"
```

### 4. Trigger subtitle search for each episode

Process one at a time — each call blocks until Bazarr/Whisper completes (~3 min per episode):

```bash
docker exec bazarr curl -s --max-time 600 -X PATCH \
  -H "X-API-KEY: <key>" -H "Content-Type: application/json" \
  -d '{"episodeid":<eid>, "seriesid":<sid>, "language":"<lang>", "forced":"false", "hi":"false"}' \
  "http://127.0.0.1:6767/bazarr/api/episodes/subtitles"
```

Run these sequentially in a background Bash command. Add `sleep 5` between each to let Bazarr breathe.

### 5. Monitor progress

After launching the batch, check SubGen activity:

```bash
docker logs subgen --tail 5 2>&1
```

## Important Notes

- **Whisper subs require minimum_score ≤ 50** (they score ~61%). Check before running.
- Each Whisper transcription takes ~2-3 min on the RTX 3070 Ti with large-v3 model.
- Run the batch as a background command so the user isn't blocked.
- Report how many episodes were triggered and estimated completion time.
