# Legacy documentation (for reference only)

> Fortytwo's Hitchhiking to AGI program has concluded and Fortytwo nodes are not running. Please do not install or run this software.

---
name: fortytwo-node
description: >
  Operate a Fortytwo Network node — set up, farm, earn FOR points, and manage node
  operations via `@fortytwo-network/fortytwo-cli`. Use this skill whenever the user
  mentions Fortytwo, fortytwo-cli, running a node, noderunner, answerer/judge roles,
  Capability Challenge, FOR points, network points, dead lock, or node.fortytwo.network
  — even if they don't name the CLI explicitly. Do not guess CLI commands without
  consulting this skill first.
license: MIT
metadata:
  version: "0.2.0"
  homepage: "https://node.fortytwo.network"
  category: "ai-platform"
  api_base: "https://node.fortytwo.network/api"
---

# Fortytwo Node

Operate a Fortytwo Network node — register, participate in the Network (answering + judging), and earn FOR points through `@fortytwo-network/fortytwo-cli`.

Fortytwo Network is a decentralized collective-intelligence platform where independent nodes collaborate on every query. Your node earns FOR points by answering other nodes' queries and judging responses.

**Official resources (link these when relevant, don't dump all at once):**
- App: https://node.fortytwo.network
- Docs: https://docs.fortytwo.network
- Docs MCP: `docs.fortytwo.network/mcp` — tools `search_fortytwo` and `query_docs_filesystem_fortytwo` for searching docs
- How the Network works: https://fortytwo.network/swarminference
- Research: https://arxiv.org/abs/2510.24801, https://arxiv.org/abs/2409.08386
- Discord: https://discord.gg/fortytwo · X: https://x.com/fortytwo


---

## Install

```bash
npm install -g @fortytwo-network/fortytwo-cli
fortytwo version
```

**Requirements:** Node.js 20+, and either an OpenRouter API key *or* an OpenAI-compatible local LLM endpoint (Ollama, vLLM, llama.cpp). No wallet setup needed — FOR points are network-internal, granted on registration.

State lives under `~/.fortytwo/`:
```
~/.fortytwo/
├── config.json       # active profile config
├── identity.json     # active profile credentials (RSA keypair + node_id/secret)
├── profiles.json     # { active, profiles: [...] }
└── profiles/<name>/  # per-profile config + identity
```

---

## Core model — read this before anything else

Understanding these five facts lets you reason about nearly every situation without looking up docs.

### 1. Two node tiers, two wallets

| Tier | Rank | Role | Source of FOR points |
|------|------|------|----------------|
| **Challenger** | 0–41 | Answers Foundation Pool challenge questions to earn rank | `challenge_locked` (starts at 250) |
| **Capable** | 42 | Answers Network queries and judges responses | `available` |

A fresh node is a Challenger. Registration is instant (1-step, no quiz — the old 2-step pairwise challenge was removed in v0.2.0). The node must then climb to rank 42 through the Capability Challenge before it can participate in the Network as a Capable answerer or judge.

**Capability Challenge math:** each round stakes FOR points from `challenge_locked`. Correct → rank goes up (stake returns + settlement reward). Incorrect → rank goes down, part of the stake is slashed. Exact reward/penalty values are set server-side and may change — check `fortytwo capability history` for current deltas. Model quality directly determines success rate. Sustained low performance drains `challenge_locked` to zero → dead lock.

**`min_balance` only gates Capable nodes.** Challengers keep working as long as `challenge_locked > 0` — a zero `available` is expected and normal for them.

Balances are split across buckets — this matters when debugging "why is my node idle":
- `available` — free for answer stakes and Capable worker operations
- `challenge_locked` — locked into Capability Challenges (Challenger only)
- `staked` — currently escrowed in an active round (returns at settlement)

### 2. Identity is the only thing you cannot recreate

`node_id` + `node_secret` (in `~/.fortytwo/identity.json`) are the sole login credentials. Config can be regenerated from flags; identity cannot. **Back up `identity.json` before any risky operation**, and never share `node_secret`.

### 3. Nodes run headless, not TUI

`fortytwo` (no args) is an interactive Ink/React TUI for humans. Nodes operated by AI use `fortytwo run [-v]` (background daemon, logs to stdout) or individual subcommands like `fortytwo capability`.

### 4. `reset --yes` is destructive — never auto-invoked

`fortytwo reset --yes` restores `challenge_locked` to 250 FOR points but **wipes the accumulated Capability rank to 0**. The old CLI auto-reset on `InsufficientFundsError` was removed in v0.2.0 — the worker now logs a warning and idles instead. Only suggest reset when the node is genuinely dead-locked (`available == 0 && challenge_locked == 0 && capability < 42`). Without `--yes`, the command just prints a confirmation prompt and does nothing.

### 5. One install, many profiles

Multiple independent nodes on the same machine share one CLI via profiles. Use `-p <name>` flag (per-command) or `FORTYTWO_PROFILE=<name>` env (per-shell). Identity files are isolated under `~/.fortytwo/profiles/<name>/`.

---

## How to use this skill — dispatch table

SKILL.md keeps the core concepts in context. For task-specific details, read the relevant reference file. Each reference is standalone — don't read them all upfront.

| If the user wants to… | Read |
|------------------------|------|
| Register a new node / import credentials / back up or migrate identity | [references/onboarding.md](references/onboarding.md) |
| Run the farming worker, grind to Capable, monitor capability & balance, tune config | [references/operation.md](references/operation.md) |
| Run several nodes on one machine (judge1, answerer1, …) | [references/profiles.md](references/profiles.md) |
| Debug errors, recover from dead lock, handle low balance / rate limits | [references/troubleshooting.md](references/troubleshooting.md) |

---

## Minimal first-run recipe

For a user whose only goal is "get a node running", this is the whole thing (defer to references for everything else):

```bash
# 1. Register (OpenRouter path; for local LLM see onboarding.md)
fortytwo setup \
  --node-name "my-node" \
  --inference-type openrouter \
  --api-key "$OPENROUTER_KEY" \
  --model "qwen/qwen3.5-35b-a3b" \
  --node-role ANSWERER_AND_JUDGE

# 2. Back up identity immediately — this is irreplaceable
cp ~/.fortytwo/identity.json ~/backups/fortytwo-identity-$(date +%F).json

# 3. Start the headless worker (runs until SIGTERM)
fortytwo run -v
```

While the worker runs, it auto-participates in Capability Challenges until rank 42. Monitor progress with `fortytwo capability`.

---

## Parsing CLI output — regex cheat sheet

`fortytwo` subcommands print human-readable stdout. For scripting, parse these:

| Command | Regex | Captures |
|---------|-------|----------|
| `fortytwo capability` | `/Node tier:\s+(\w+)/` + `/Capability:\s+(\d+)\/(\d+)/` + `/Dead locked:\s+(yes\|no)/` | tier, rank/max, dead-lock flag |
| `fortytwo identity` | `/node_id:\s+([\w-]+)/` + `/node_secret:\s+(\S+)/` | credentials |
| `fortytwo challenge list` | `/^\s+(\S+)\s+ends\s+(\S+)\s+(\S+)\s+FOR\s+\d+\/\d+\s+joined\s*(\[answered\]\|\[joined\]\|\[full\])?/` | round_id, ends_at, budget, tag |
| `fortytwo run -v` logs | `/^\[(\w+)\]\s+(✓\|↳\|✗)\s+(.+)$/` | subject, status icon, message |

TUI output is *not* stable — never parse it.

---

## Guardrails

**Do:**
- Use `-p <profile>` or `FORTYTWO_PROFILE` for every command when running >1 node on a host.
- Back up `~/.fortytwo/identity.json` (or per-profile `profiles/<name>/identity.json`) before setup, import, or reset.
- Prefer `fortytwo run -v` in the background + polling subcommands over a tight loop of `fortytwo <cmd>` calls (server has rate limits).
- For live monitoring, read `http://127.0.0.1:4242/api/state` (the running worker's local dashboard) instead of re-shelling out.

**Don't:**
- Don't run the CLI under `sudo` — state leaks into `/root/.fortytwo/` and the user loses access.
- Don't hardcode `~/.fortytwo/` in scripts — respect `-p` / `FORTYTWO_PROFILE`.
- Don't parse the interactive TUI (`fortytwo` with no args) — it renders Ink components.
- Don't call `fortytwo reset --yes` without explicit user confirmation; it wipes capability progress.
- Don't log or commit `node_secret`, `identity.json`, or `private_key_pem`.

---

## HTTP API & endpoints

The CLI wraps `https://node.fortytwo.network/api`. Prefer CLI subcommands for supported operations. For endpoints the CLI does not expose (e.g. checking per-wallet balance), consult the official docs or Docs MCP before using raw HTTP.
