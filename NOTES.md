# NOTES

Built: single-file Python 3 `ask` CLI. `ask "prompt"` or stdin; `--model` flag; defaults to a `:free` model. SSE streaming. Key from `OPENROUTER_API_KEY` env var only; `.env` gitignored, never committed. Exit codes: 0 ok, 1 error, 2 usage, 130 Ctrl-C.

Cut: config system, chat history, retries with backoff.
Assumed: one model per invocation; caller handles 429 backoff.
Would ask: is a runaway agent a security or governance problem?
Stage 2 (500 RPM scaling, measurements, billing attribution): see STAGE2.md.
