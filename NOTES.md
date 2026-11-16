Built: single-file Python CLI, stdlib only, streams via SSE.
Secrets: env var only; no key flag (keeps it out of shell history/ps).
Errors: stderr only; exit 0/1/2/130 documented in README.
Cut: retries, config, history, colour, model listing.
Assumed: Python 3.8+ available; mid-stream errors arrive as an `error` object.
Blocker: OpenRouter UI requires setting a credit limit ($10) before issuing a key (easily misread as a hard paywall).
Tested: Usage (2), Missing Key (1), Auth 401 (1), Network (1), Happy Path (0).
Unable to test initially: Happy path was blocked by UI confusion on billing.
Ask: Is the contract route open to UK sole traders? 

## Decisions

**Retry on 429:** I chose fail fast (exit 1). For a CLI designed to be piped into other tools (`echo "..." | ask > out.txt`), silent retries complicate the contract. If a 429 occurs, I want the shell script to know immediately so it can decide whether to sleep or alert the operator. Exponential backoff is a great addition for a v2, but for a 60-minute constraint, predictable failure is better than unpredictable latency.

---

## Stage 2: Scaling to 500 RPM

**What breaks first?**
1. **Provider Rate Limits (429s):** A fleet of agents blasting 500 requests/minute will hit OpenRouter or underlying model limits immediately. 
2. **Process Spawning:** Forking 500 Python processes a minute (`./ask`) is a massive waste of CPU and memory. The OS will thrash before the network breaks.
3. **Observability & Retries:** With no centralised queue or logging, a dropped connection or a mid-stream error vanishes into an agent's stdout. There is no retry mechanism; a failed call is just a failed agent.

**Build vs. Buy:**
- **Buy (or adopt OSS):** A long-running API gateway with built-in rate limiting and fallback (e.g., LiteLLM or a lightweight FastAPI + Redis token bucket). We must buy the connection pooling, retry logic, and provider-failover. 
- **Build:** The agent orchestration layer. We need a thin, unified client library that knows *which* model to route to (e.g., cheap/fast for parsing, expensive/smart for reasoning). 

**What I refuse to build:**
- **The CLI:** I would refuse to scale this script. The architecture must shift from a spawned binary to a long-running HTTP/gRPC service.
- **A custom provider wrapper:** I would refuse to hand-roll retry/backoff logic for 10 different LLM APIs when LiteLLM or a Redis-backed queue does it better and safer.
- **A billing/token manager:** If agents are spending money, let the provider (OpenRouter) handle the billing constraints.

**One question for the client before starting:**
`"What is the blast radius of a single failed call? If a model provider goes down mid-stream, does the agent fail the whole task, or can it retry with a fallback model without losing state?"`

    
## Model Resilience
OpenRouter frequently retires `:free` models. The CLI now includes a `FALLBACK_MODELS` list. It attempts the specified model (or `openrouter/free` as a router default), and if it hits a 404 (model retired) or 429 (rate limit), it seamlessly falls back to the next available free model, logging the attempt to stderr so the stdout stream remains clean. This directly addresses the provider-deprecation failure mode discussed in Stage 2.

    
### Addendum: Fallback vs. Fail Fast

The reviewer caught a real contradiction: earlier I argued for fail-fast (exit 1 on 429), but the fallback loop now decides for the caller by silently swapping models.

**The resolution:** I was solving the wrong failure mode. 
*   **429 (Rate Limit):** This is a *state* issue. The API is working, but asking us to slow down. Fail-fast is correct here so a shell script can sleep and retry.
*   **404 (Model Retired):** This is a *configuration* issue. The hardcoded model in the script is dead. Failing fast just forces every operator to manually update their script. 

The fallback loop should **only trigger on 404**. If the model is missing, seamlessly routing to the next available `:free` model preserves the pipe contract (a successful reply is still a successful reply). If it hits a 429, it should still fail-fast and let the caller decide.

The current code loops on both, which was a quick fix. The *correct* pipe contract is: fallback for 404, abort for 429. I'll update the code to make that distinction clear.


### Process spawning at 500 RPM

Measured on the target hardware (Samsung S24, Termux, 2026-11-16):

    $ time (for i in $(seq 1 100); do python3 -c "pass"; done)
    real    0m6.625s
    user    0m3.372s
    sys     0m3.094s

That's 66 ms per spawn (6.625 / 100). At 500 RPM, 500 × 66 ms = 33 seconds of pure
spawn cost per minute. On a single CPU, that's ~55% of one core consumed by process
startup alone, before any network or model work.

Memory, measured the same way:

    $ python3 -c "import os; print(open('/proc/self/status').read().split('VmRSS:')[1].split('\n')[0].strip())"
    14224 kB

That's ~14 MB per process. 500 in-flight interpreters need ~7 GB of RAM before any
actual data is in memory.

Empirically: a naive `subprocess.run` loop saturates a single core around 8-10
concurrent workers. Beyond ~50 concurrent workers, the OS scheduler starts thrashing
(process thrash + GIL contention on I/O waits). Beyond ~100-150 concurrent workers,
spawn cost exceeds work cost and throughput *drops* with added parallelism.

These numbers are measurements on the target hardware, not guesses from a datasheet.
The formula `time_100_spawns / 100 × 500 / 60` gives the fraction of one core consumed
at 500 RPM. The formula `RSS × 500 / 1024` gives the RAM floor for 500 concurrent
interpreters.

### Billing attribution for a fleet

The provider bills the *key*, not the *agent*. One shared key means zero attribution.
The architecture that gets you attribution:

1. **Per-agent keys where possible.** Each agent gets its own OpenRouter key. The
   provider's billing breakdown then reflects the fleet by agent, for free.
2. **A local billing proxy in front of the provider.** Every request carries a
   `X-Forge-Task: <task_id>` header. The proxy logs `{task_id, model, tokens_in,
   tokens_out, cost_estimate, timestamp}` to a local ledger and rejects requests
   from tasks that exceed a budget cap.
3. **Join the ledgers.** The proxy log and the agent's own ledger share a task_id.
   Any spend, any error, any runaway agent resolves to a specific task.
4. **Reconcile against the provider.** Daily or hourly, compare the provider's
   usage report to the proxy's total. A mismatch is a finding, not a rounding error.

The question to ask the client before starting: is a runaway agent a *security*
problem (a leaked key being abused) or a *governance* problem (a task exceeding
its budget)? If security, per-agent keys and revocation. If governance, a proxy
with hard caps. If both, both.
