# Jellyfin Issue Logger

Log a Jellyfin issue and its solution to Obsidian for future reference.

## Instructions

1. Ask the user to describe the issue they encountered and how they fixed it (or are fixing it). If they already provided this context in the conversation, use that instead of asking again.

2. Search the Obsidian vault for the existing Jellyfin troubleshooting note:
   - Use `mcp__obsidian-vault__search_notes` with query "Jellyfin Troubleshooting" searching frontmatter.
   - Also try reading `Homelab/Jellyfin Troubleshooting.md` directly.

3. Format the new issue entry as:

```
## <Short descriptive title>

**Date:** <today's date, YYYY-MM-DD>
**Symptom:** <what was observed / what went wrong>
**Cause:** <root cause if known>
**Solution:** <step-by-step fix>
**Notes:** <any additional context, links, or caveats — omit if empty>

---
```

4. If the note already exists, use `mcp__obsidian-vault__write_note` with mode `append` to add the new entry to `Homelab/Jellyfin Troubleshooting.md`.

5. If the note does NOT exist, create it with `mcp__obsidian-vault__write_note` using this template as the base, followed by the first issue entry:

```
---
tags:
  - homelab
  - jellyfin
  - troubleshooting
---

# Jellyfin Troubleshooting

A log of Jellyfin issues encountered and their solutions.

---

<issue entry here>
```

6. After writing, confirm to the user what was logged and the note path.
