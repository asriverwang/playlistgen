# PlaylistGen — API Reference

Base URL: `http://localhost:5678` (or whatever `PORT` and `MUSIC_SERVER_URL` are set to).

**Never construct player URLs manually. Always use the `url` field returned by the server.**

---

## Playlist generation

### Fully automated (recommended)

One call — interpret → fetch → curate → save:

```
POST /api/generate
{"prompt": "obscure 80s synth for late night driving", "max_count": 30}

→ {"title": "🎵 ...", "url": "http://<host>:5678/player?saved=XXXX", "count": 28, "tags": {...}}
```

Share only the returned `url`. Do not modify it.


---

## Other endpoints

| Endpoint | Description |
|---|---|
| `GET /api/stats` | Library stats: total songs, top artists/albums |
| `GET /api/search?q=<text>` | Keyword search on title/artist/album (50 results) |
| `GET /api/interpret?q=<prompt>&provider=minimax\|claude` | LLM prompt → structured tags |
| `GET /api/catalog/vocab` | Full tag→count dict for all fields |
| `GET /api/catalog/vocab/refresh` | Rebuild vocab from DB, rewrite catalog JSON file |
| `GET /` | Web search UI |
| `GET /player?saved=<key>` | Playlist player page |

---

## Song fields

| Field | Type | Notes |
|---|---|---|
| `path` | string | Absolute filesystem path — use as song ID in all API calls |
| `url` | string | Relative path — use as audio `src` in browser |
| `title` | string | |
| `artist` | string | |
| `album` | string | |
| `year` | string | |
| `genre` | string | e.g. `"Rock"`, `"Electronic"` |
| `subgenre` | string | e.g. `"Shoegaze"`, `"Synthpop"` |
| `mood` | string | Comma-separated, e.g. `"Melancholic, Dreamy"` |
| `usage_context` | string | e.g. `"night"`, `"driving"` |
| `energy` | string | `"low"`, `"medium"`, or `"high"` |
| `language` | string | e.g. `"English"`, `"Mandarin"` |
| `region` | string | e.g. `"UK"`, `"USA"`, `"Taiwan"` |
| `popularity` | string | `"mainstream"`, `"indie"`, `"niche"`, `"obscure"` |
| `duration` | float | Seconds |
