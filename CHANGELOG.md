# CHANGELOG — ask

## [v1.0.0] — Challenge Submission

### Good
- Built single-file Python CLI (`ask`), standard library only.
- Implemented strict SSE parsing, flushing `delta.content` live to stdout.
- Read API key exclusively from `OPENROUTER_API_KEY` environment variable.
- Error handling: mapped `HTTPError` and `URLError` cleanly to stderr.
- Graceful `KeyboardInterrupt` handling (exit 130, no traceback).

### Bad
- Could not test the 200 OK happy path (streaming) because OpenRouter currently requires a credit card to generate an API key.
- Because auth is checked before routing, the 404 (model not found) test returned a 401, as expected with a fake key.

### Ugly
- (none)
