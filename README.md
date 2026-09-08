# Feed Digest Extension for FreshRSS (fork)

A fork of [fengchang/xExtension-FeedDigest](https://github.com/fengchang/xExtension-FeedDigest) that summarizes RSS articles into a dedicated **"AI Summaries"** stream, without touching the source articles.

Built for a local LLM (Ollama + Qwen) but works with any OpenAI-compatible API.

## How it differs from upstream

| | Upstream | This fork |
| --- | --- | --- |
| Destination | Summary articles inside each source feed | One dedicated "AI Summaries" feed + category |
| Granularity | One combined summary per batch (batch > 1) | One summary per article (batch is always 1) |
| Source articles | Marked read after summarization | **Untouched**: content, read state, and flags never modified |
| Skipped articles | "Not summarized" note written into the article | **Untouched**: left unread, silently re-evaluated (zero LLM cost) |
| Idempotency | Summary detection by feed/title heuristics | Deterministic guid `ai-summary-<source entry id>` — a source can never get two summaries, and read-state toggles never trigger re-summarization |
| Long articles | batch=1 path hard-codes a 50,000-char limit | Honors the configured `max_content_length` (default 4000) before prompt construction |
| LLM payload | Plain chat completion | `reasoning_effort: none` (no hidden reasoning overhead), `response_format: json_object` (strict JSON), `max_tokens: 500` |
| Summary style | 2–4 sentences | Idea-dense: every sentence carries a distinct idea, no filler; typically 3–6 sentences |

## Behavior

- Runs during the regular FreshRSS actualize (cron). Per-feed opt-in via the feed's **Feed Digest** settings (`feed_digest_enabled`, `feed_digest_batch_size` clamped to 1).
- For each unread, un-summarized article of an opt-in feed (oldest first, up to 200 per feed per pass):
  - text too short / image-only → skipped, untouched, re-checked on later passes;
  - otherwise one API call → a summary-only entry is created in the AI Summaries feed:
    - title: `Summary: <original title> (<source feed name>)`
    - content: the summary text, a "Read the full article" link, and a `Source: <feed> — <title>` line
    - date: the original article's date, so the stream stays in feed order
    - guid: `ai-summary-<source entry id>` (stable across retries)
- The AI Summaries feed/category is created deterministically on first use (`internal://ai-summaries`, 1-year TTL so it is never fetched) and never recreated.
- Idle passes (nothing new) cost zero LLM calls.

## Configuration

System-level (Settings → Extensions → Feed Digest → Configure):

- `api_endpoint` — OpenAI-compatible base URL, e.g. `http://100.118.57.3:11434/v1`
- `secret_key` — API key (`ollama` or anything non-empty for local servers)
- `model` — e.g. `qwen3.6:35b-a3b`
- `dest_language` — e.g. `English`
- `max_content_length` — input budget in characters per article (default 4000). Truncation happens before the prompt is built; keep the resulting prompt comfortably below the model's context window.

Per-feed: enable summarization in the feed's settings (checkbox + batch size; batch is effectively always 1).

## Deployment (Docker volume model)

Clone into the FreshRSS `extensions` volume, then enable in config:

```bash
git clone --depth 1 https://github.com/nartiz/xExtension-FeedDigest \
    /var/lib/docker/volumes/freshrss_freshrss_extensions/_data/FeedDigest
```

- system `data/config.php`: `extensions_enabled['Feed Digest'] = true` plus the `extensions['Feed Digest']` settings above
- per-feed attributes in the DB (`feed` table, `attributes` JSON): `{"feed_digest_enabled": true, "feed_digest_batch_size": 1}`

For full-text extraction on excerpt-only feeds, pair with [freshrss-af-readability](https://github.com/Niehztog/freshrss-af-readability) (clone as `Af_Readability/` into the same volume, enable per-feed in the user config).

## Verification notes (local Qwen backend)

- With `reasoning_effort: none`, Qwen 3.6 35B-A3B via Ollama produces ~100-token summaries in ~8–10 s per article. Without it, the model emits ~1,200+ tokens of hidden reasoning per call (~38 s).
- Ollama's context slot on this deployment is 4096 tokens; the 4000-char input budget keeps prompts around 1.5k tokens.
- Measured end-to-end (8 feeds, 150-article backlog): 13 m 40 s, 149/150 summarized, sources byte-identical before/after, zero read-state changes.

## License

GNU Affero General Public License v3.0 (AGPL-3.0). See [LICENSE](LICENSE). Fork of [fengchang/xExtension-FeedDigest](https://github.com/fengchang/xExtension-FeedDigest).
