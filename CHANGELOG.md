# CHANGELOG — ask

## [v1.0.0] — Challenge Submission

### Good
- Built single-file Python CLI (`ask`), standard library only.
- Implemented strict SSE parsing, flushing `delta.content` live to stdout.
- Read API key exclusively from `OPENROUTER_API_KEY` environment variable.
- Error handling: mapped `HTTPError` and `URLError` cleanly to stderr.
- Graceful `KeyboardInterrupt` handling (exit 130, no traceback).
- Committed core logic, `NOTES.md`, and executable bit (commit 79180e2).
- Polish: Made SSE parser robust to missing space after `data:` prefix.
- Polish: Added test matrix to `README.md` and PR description.
- Context: Completed in 31 minutes (18:48-19:19) using Termux on an S24 phone.

### Bad
- Could not test the 200 OK happy path (streaming) because OpenRouter currently requires a credit card to generate an API key.
- Because auth is checked before routing, the 404 (model not found) test returned a 401, as expected with a fake key.

### Ugly
- Tried to update PR body with `gh pr edit --body "..."`, but the shell executed the markdown backticks as commands. Terminal spam ensued.
- Attempted to write a temp file to `/tmp` on Termux, which resulted in a permission denied error due to Android sandboxing.
