# Operation — farming, Capability Challenge, monitoring

Read this when the user wants to run the worker, climb from Challenger to Capable, monitor balances and rank, or tune performance.

## The farming loop in one paragraph

`fortytwo run` is a long-lived polling daemon. Every `poll_interval` seconds (default 120) it polls the API for available work and dispatches based on tier and role:

- **Challenger** → pulls active Foundation Pool rounds (challenge questions), joins each round (which stakes FOR points from `challenge_locked` and reveals the question), generates an answer via LLM, submits. Rounds with <30s remaining are auto-skipped. Correct answer → rank goes up (stake returns + settlement reward). Incorrect → rank goes down, part of the stake is slashed. Exact values are set server-side and may change. If the node crosses rank 42 mid-cycle, the challenge loop stops and the next cycle dispatches to queries/judgments.
- **Capable** with role `ANSWERER` → claims open Network queries, stakes, generates, submits.
- **Capable** with role `JUDGE` → pulls pending ranking challenges, runs Bradley-Terry comparison, submits votes.
- **Capable** with `ANSWERER_AND_JUDGE` → does both; per-query role is assigned by the network based on a query hash.

At rank 42, the worker flips tier from Challenger to Capable, moves remaining `challenge_locked` into `available`, and starts doing real Network work.

---

## Start / stop the worker

```bash
# Foreground (useful for watching logs live)
fortytwo run -v

# Background (recommended for noderunners)
nohup fortytwo run -v > ~/.fortytwo/worker.log 2>&1 &
WORKER_PID=$!

# Graceful shutdown
kill -TERM $WORKER_PID       # or SIGINT (Ctrl-C) in foreground
```

The worker handles SIGTERM/SIGINT with a graceful shutdown — it finishes any in-flight LLM call, submits pending answers, then exits. Do not `kill -9` unless it's wedged; you may lose an in-progress stake.

### Don't run two workers on the same identity

The server expects exactly one heartbeat per `node_id`. Two concurrent `fortytwo run` processes under the same identity cause duplicate answer attempts (rejected with "Already submitted for this round") and potential rate-limiting. For N nodes, use N profiles — see [profiles.md](profiles.md).

---

## Monitor progress

### CLI polling

```bash
fortytwo capability
#   Node tier:      challenger
#   Capability:     15/42
#   Dead locked:    no

fortytwo capability history
#   Capability history (12 total):
#     2026-04-13T15:30:00Z +1.5 (10.5→12) — challenge_correct
#     2026-04-13T15:15:00Z -1 (12→11) — challenge_incorrect
#     2026-04-13T15:00:00Z +1.5 (9→10.5) — challenge_correct
```

Parse with:
```bash
RANK=$(fortytwo capability | awk -F'[ /]+' '/Capability:/ {print $2}')
TIER=$(fortytwo capability | awk '/Node tier:/ {print $3}')
DEAD=$(fortytwo capability | awk '/Dead locked:/ {print $3}')
```

### Local viewer dashboard (preferred for live state)

When `fortytwo run` (or the interactive TUI) is active, it serves a JSON endpoint on localhost:

```bash
curl -s http://127.0.0.1:4242/api/state | jq .
```

The response includes live balance (all three buckets), capability info (`capabilityRank`, `nodeTier`, `isDeadLocked`), current task (answering/judging/challenge), `challengeRoundsAvailable`, and a ring buffer of recent log lines. The TUI also now shows a capability progress bar and tier badge. This is cheaper than repeated `fortytwo <cmd>` calls (no extra auth/RTT) — prefer it when a worker is running.

**Capability history reasons** (from `fortytwo capability history`):
- `challenge_correct` — answered a Foundation Pool round correctly (rank increases, stake returns)
- `challenge_incorrect` — answered incorrectly (rank decreases, part of stake slashed)
- `challenge_cancelled` — round was cancelled (stake refunded, no rank change)
- `reset` — manual `fortytwo reset --yes` (rank → 0)
- `migration` — server-side migration event

### Foundation Pool rounds

```bash
fortytwo challenge list
#   Active challenge rounds (3):
#     01234567  ends 2026-04-13T16:00:00Z  reward 10 FOR
#     89abcdef  ends 2026-04-13T17:30:00Z  reward 10 FOR [answered]
```

The listing now also shows `joined_count/max_participants` and tags: `[full]` (no slots left), `[answered]` (already submitted), `[joined]` (joined but not yet answered).

`[answered]` means the running worker already submitted for that round. You *can* submit manually:

```bash
fortytwo challenge answer 01234567 "Yes"
```

— but the worker does this automatically, so only do it to replay a specific round or test an override.

---

## Grind from Challenger to Capable

No shortcut exists: rank 42 requires solving enough challenges correctly. The number of rounds needed depends on the current reward/penalty values (check `fortytwo capability history` for actual deltas).

### Throughput levers

Tune these in `~/.fortytwo/config.json` via `fortytwo config set <key> <value>`:

| Key | Default | Effect |
|-----|---------|--------|
| `poll_interval` | 120 (sec) | Lower → more round attempts per hour, but more API/LLM load and rate-limit risk. 60 is safe; below 30 starts getting throttled. |
| `llm_concurrency` | 40 | Parallel LLM requests per batch. Higher helps when many rounds are active; capped by provider rate limits. |
| `llm_timeout` | 120 (sec) | Hard deadline per LLM call. Bump to 300 for slow self-hosted models. |
| `min_balance` | 5.0 | Below this `available`, **Capable** workers go idle (log: `Low balance: X FOR < Y required. Worker idle`). Does not apply to Challengers — they are funded by `challenge_locked`, so zero `available` is expected and normal. |
| `answerer_system_prompt` | "You are a helpful assistant." | Prepended to every answer generation. Tuning this is the single biggest lever on answer quality for Capable answerers. |

Restart the worker after `config set` — some fields aren't re-read live.

### Dead lock — the one failure mode you can't wait out

```
available        0
challenge_locked 0
capability      <42
```

You've lost enough stakes that there's nothing left to play with. The worker detects this (`is_dead_locked: true` from the API), logs a warning, and **does not auto-reset** (auto-reset was removed in v0.2.0). The only escape is manual `fortytwo reset --yes` — it grants 250 FOR points back to `challenge_locked` but **wipes all accumulated capability rank to 0**. The reset response tells you exactly what happened: `rank_before → 0, +250 FOR locked`. Detect and recover:

```bash
if fortytwo capability | grep -q "Dead locked:\s*yes"; then
  echo "Dead locked — resetting (capability progress will be lost)"
  # Get explicit user confirmation first; don't auto-run in a scripted skill
  fortytwo reset --yes
fi
```

Prevention: don't run a Challenger on a model too weak for the challenge questions — stake slashing on wrong answers burns through the FOR points budget. See onboarding.md for recommended models. A model that gets <60% correct will almost certainly dead-lock before reaching rank 42.

---

## Capable operation (once at rank 42)

At that point:
- `challenge_locked` → 0, remaining balance moved to `available`
- Role (`ANSWERER` / `JUDGE` / `ANSWERER_AND_JUDGE`) starts mattering
- Worker now picks up queries and ranking challenges instead of Foundation Pool rounds

Monitor economy from a Capable node:

```bash
# Programmatic — the CLI doesn't surface balance directly
# curl /economy/balance/{id} with a Bearer token, or use the viewer:
curl -s http://127.0.0.1:4242/api/state | jq '.balance'
```

If `available` drops below `min_balance` the worker auto-idles (log line: `Insufficient funds — worker idle.`). That's by design — wait for settlement or reduce Capable worker activity until balance recovers.
