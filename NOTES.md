Built: single-file Python CLI, stdlib only, streams via SSE.
Secrets: env var only; no key flag (keeps it out of shell history/ps).
Errors: stderr only; exit 0/1/2/130 documented in README.
Cut: retries, config, history, colour, model listing.
Assumed: Python 3.8+ available; one free model is a fine default.
Assumed: mid-stream errors arrive as an `error` object in a data line.
Blocker: OpenRouter requires a card to generate keys today. 
Tested: Usage (2), Missing Key (1), Auth 401 (1), Network (1). 
Unable to test: Happy path (200 OK) streaming due to key gating.
Ask: Is the contract route open to UK-based sole traders?
Ask: Would you want a retry on 429 for free models, or fail fast?
