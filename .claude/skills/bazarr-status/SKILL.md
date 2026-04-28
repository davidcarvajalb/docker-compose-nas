---
name: bazarr-status
description: Use when the user asks about Bazarr health, provider status, subtitle queue, wanted count, throttling, or SubGen/Whisper status. Quick health check of the entire subtitle pipeline.
allowed-tools: [Read, Grep, Bash]
---

# Bazarr Status Check

## Steps

### 1. Read API key

```bash
grep "apikey" bazarr/config/config/config.yaml | head -1
```

The key is under `auth:` > `apikey:`.

### 2. Check settings

```bash
docker exec bazarr curl -s -H "X-API-KEY: <key>" \
  "http://127.0.0.1:6767/bazarr/api/system/settings" | python3 -c "
import json, sys
g = json.load(sys.stdin)['general']
print('=== Bazarr Settings ===')
print('Providers:', g['enabled_providers'])
print('Min score (series):', g['minimum_score'])
print('Min score (movies):', g['minimum_score_movie'])
print('Search frequency:', g['wanted_search_frequency'], 'hours')
print('Adaptive searching:', g['adaptive_searching'])
print('Serie default profile:', g['serie_default_profile'])
print('Movie default profile:', g['movie_default_profile'])
"
```

### 3. Check wanted queue

```bash
docker exec bazarr curl -s -H "X-API-KEY: <key>" \
  "http://127.0.0.1:6767/bazarr/api/episodes/wanted?length=1" | python3 -c "
import json, sys; print('Wanted series episodes:', json.load(sys.stdin).get('total', 0))
"

docker exec bazarr curl -s -H "X-API-KEY: <key>" \
  "http://127.0.0.1:6767/bazarr/api/movies/wanted?length=1" | python3 -c "
import json, sys; print('Wanted movies:', json.load(sys.stdin).get('total', 0))
"
```

### 4. Language breakdown (sample 100 wanted)

```bash
docker exec bazarr curl -s -H "X-API-KEY: <key>" \
  "http://127.0.0.1:6767/bazarr/api/episodes/wanted?length=100" | python3 -c "
import json, sys
data = json.load(sys.stdin)
langs = {}
for ep in data.get('data', []):
    for m in ep.get('missing_subtitles', []):
        name = m.get('name', '?')
        langs[name] = langs.get(name, 0) + 1
print('=== Missing by language (100 sample) ===')
for lang, count in sorted(langs.items(), key=lambda x: -x[1]):
    print(f'  {lang}: {count}')
"
```

### 5. Check provider throttling

```bash
docker logs bazarr 2>&1 | grep -i "throttl\|All providers" | tail -10
```

### 6. Check SubGen (Whisper AI)

```bash
docker ps --format '{{.Names}} {{.Status}}' | grep subgen
docker logs subgen --tail 10 2>&1
```

## Report

Present a clear summary table:
- Providers: which are enabled, which are throttled
- Queue: total wanted (series + movies), language breakdown
- SubGen: running/stopped, recent activity
- Settings: anything that looks misconfigured (score too high, adaptive blocking, etc.)
- Recommendations if anything is off
