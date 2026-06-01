# Troubleshooting — errors, recovery, and common gotchas

Diagnostic-first reference. Match the symptom, apply the fix.

## Symptom matrix

| Symptom or error string | Likely cause | Fix |
|-------------------------|--------------|-----|
| `No config found. Run 'setup' or 'import' first.` | `~/.fortytwo/config.json` missing, or wrong profile active | `fortytwo setup` for a new node, or `fortytwo -p <name> …` to select the right profile |
| `Dead locked: yes` in `fortytwo capability` | all three balance buckets are empty while Challenger | `fortytwo reset --yes` — **loses accumulated rank** (back to 0), returns 250 FOR points to `challenge_locked`. Auto-reset was removed in v0.2.0 — must be manual. |
| `Cannot reach OpenRouter API: 401` | wrong `openrouter_api_key` or revoked | `fortytwo config set openrouter_api_key sk-or-...` and restart worker |
| `ECONNREFUSED http://localhost:11434/v1` | self-hosted LLM endpoint not reachable | start the server (`ollama serve`), verify model pulled (`ollama pull <model>`), re-test |
| `Low balance: X FOR < Y required. Worker idle` | Capable node with `available < min_balance` | wait for settlement, lower `min_balance`, or reduce Capable worker activity until balance recovers. This message only applies to Capable nodes — Challengers with zero `available` are normal (they use `challenge_locked`). |
| `Already submitted for this round` | duplicate answer (two workers on same identity, or retry after crash) | ensure only one worker per `node_id`; if it's a retry, safe to ignore |
| `Challenge round has ended` | round TTL expired mid-generation | nothing to do — worker skips and picks the next round automatically |
| `429 Too Many Requests` (from any endpoint) | rate-limited | back off exponentially (starting ~5s, cap ~5min); lower `poll_interval` back toward 120 if you dropped it below 60 |
| Worker hangs, no new logs for >5× `poll_interval` | LLM provider stalled, or network partition | SIGTERM (not SIGKILL) → restart; if repeatable, raise `llm_timeout` or switch model |
| `fortytwo identity` prints empty fields | state directory owned by another user (happened after running under sudo) | `chown -R $USER ~/.fortytwo`; if truly lost, restore from backup via `import` |
| `EACCES` writing `~/.fortytwo/` | previously ran CLI under `sudo` → files owned by root | `sudo chown -R $USER ~/.fortytwo`; never run the CLI as root again |
| Viewer dashboard `127.0.0.1:4242` not responding | no worker running, port in use by another profile's worker, or firewall rule | `ps aux \| grep 'fortytwo run'`; only one worker owns the port at a time |

---

## Dead lock — the single most confusing failure

When the node has:
```
available        0
challenge_locked 0
staked           0 (or tied up with nothing recoverable)
capability rank  <42
```

…it cannot attempt another Foundation Pool round (needs 10 FOR to stake) and therefore cannot earn rank. The only recovery is `fortytwo reset --yes`, which:
- Grants 250 FOR back to `challenge_locked`
- **Resets capability rank to 0** (all prior progress lost)
- Does not touch identity

### Before auto-resetting — diagnose why it happened

Almost always: the model is too weak for the challenge questions and slashed its way down. The old CLI auto-reset on `InsufficientFundsError` was removed in v0.2.0 — the worker now logs a warning and idles. Fix the root cause first, otherwise the next 250 FOR points evaporates the same way. Options:
1. Upgrade model (`fortytwo config set model_name ...`) — cheapest fix. See onboarding.md for recommended models (e.g., `nvidia/nemotron-3-super-120b-a12b:free` on OpenRouter).
2. Switch to `--node-role JUDGE` after reset — judges don't stake into Foundation Pool. But role only applies once Capable, so this won't help during the Challenger grind itself.
3. Change `--inference-type` to a higher-quality provider.

Only then reset:
```bash
fortytwo reset --yes
fortytwo run -v &
```

**Don't** script the reset as an auto-recovery loop. A reset loop with a bad model is a FOR-burning loop.

---

## Balance accounting sanity checks

If numbers look wrong (e.g., "I answered 10 rounds correctly but rank is still 0"):

1. **Check settlement state.** Rewards land at round TTL, not at submission. Rank changes appear immediately on correct/incorrect answers, but `available` / `challenge_locked` movements lag.
2. **Check the `staked` bucket.** Mid-round stakes live here, not in the original bucket. `total = available + challenge_locked + staked`.
3. **Compare to `/agents/{id}/stats`** (via `curl` with Bearer token) — server's accounting of correct/incorrect answers is authoritative. If it disagrees with your log counting, your worker might be retrying a duplicate that's getting silently rejected.

---

## Lost identity

Worst case — no backup, `~/.fortytwo/identity.json` gone. There is no recovery: `node_id` and `node_secret` are not derivable from anything else, and the server will not re-issue them. The only option is `fortytwo setup` for a brand-new node; all accumulated rank and FOR are lost.

This is why [onboarding.md](onboarding.md) pushes identity backup as the first step after registration.

If the identity file exists but is corrupt (e.g., JSON parse error):
```bash
mv ~/.fortytwo/identity.json ~/.fortytwo/identity.json.bad
cp ~/backups/fortytwo-identity-<date>.json ~/.fortytwo/identity.json
```
…then `fortytwo identity` to verify.

---

## Rate limiting and retry strategy

`node.fortytwo.network` enforces per-node rate limits. The CLI worker respects them automatically, but custom scripts often don't. Rule of thumb:

- Don't call any subcommand more than once per 10 seconds from scripts.
- On `429`, back off exponentially: 5s → 10s → 30s → 1m → 5m, then plateau.
- On `5xx`, retry up to 3 times with 2s/4s/8s; give up after that (transient outage is not your problem to fix, and more retries compound the load).
- Never use `poll_interval < 30` without testing — you will trip throttles.

---

## Diagnostics one-liner

When helping a user debug, collect these four outputs in one go:

```bash
fortytwo version
fortytwo identity                    # redact node_secret before sharing
fortytwo capability
fortytwo capability history | head
tail -n 100 ~/.fortytwo/worker.log   # or wherever they redirected `fortytwo run`
```

Also `curl -s http://127.0.0.1:4242/api/state | jq .` if a worker is running — it shows live state at a glance.

---

## When to escalate upstream (Discord / GitHub)

Most issues above are client/config problems. File an issue on `github.com/Fortytwo-Network/fortytwo-node-client` or reach out in Discord when:
- Settlement never arrives for rounds that show `[answered]` (server-side stuck).
- `GET /capability/{id}` returns different rank than `fortytwo capability` prints (client/server desync).
- The same identity is rejected with 401 even after a fresh `login` (credential-side desync).
- Any 5xx that persists for >15 minutes from `fortytwo.network` endpoints.

Include the CLI version, `node_id` (safe to share), and recent worker log excerpt.
