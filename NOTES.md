Built: single-file Python CLI, stdlib only, streams via SSE.
Secrets: env var only; no key flag (keeps it out of shell history/ps).
Errors: stderr only; exit 0/1/2/130 documented in README.
Cut: retries, config, history, colour, model listing.
Assumed: Python 3.8+ available; mid-stream errors arrive as an `error` object.
Blocker: OpenRouter requires a card to generate keys today.
Tested: Usage (2), Missing Key (1), Auth 401 (1), Network (1).
Unable to test: Happy path (200 OK) streaming due to key gating.
Time/Env: 31 mins (18:48-19:19), built via AI on Termux (S24 phone).
Ask: Contract route open to UK sole traders? Retry on 429, or fail fast?
