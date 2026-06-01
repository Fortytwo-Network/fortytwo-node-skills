# Profiles — running multiple nodes on one host

Read this when the user wants to run more than one Fortytwo node on the same machine — for example, separate judge and answerer nodes, or a staging identity alongside production. Profiles isolate config and identity; each profile is a distinct node with its own `node_id`.

## Mental model

Profiles live in a sibling tree to the single-node layout:

```
~/.fortytwo/
├── profiles.json         # { "active": "judge1", "profiles": ["judge1","answerer1",...] }
└── profiles/
    ├── judge1/
    │   ├── config.json
    │   └── identity.json
    └── answerer1/
        ├── config.json
        └── identity.json
```

The legacy top-level `~/.fortytwo/config.json` + `identity.json` still work for single-profile setups. Creating your first profile does not delete those — but once you're in profile mode, prefer explicit `-p` flags to avoid confusion about which node is "active".

---

## Managing profiles

```bash
fortytwo profile list                 # enumerate, mark the active one
fortytwo profile create judge1       # scaffold empty profile (then run setup/import under -p)
fortytwo profile switch judge1       # change the default for future commands
fortytwo profile show [name]         # print config + identity for the profile
fortytwo profile delete answerer1    # permanent; identity is lost unless backed up
```

`profile delete` is destructive — it removes `~/.fortytwo/profiles/<name>/` including `identity.json`. Back up first.

---

## Selecting a profile at runtime

Two mechanisms; pick one per invocation.

### `-p <name>` flag (per command)

```bash
fortytwo -p judge1 capability
fortytwo -p answerer1 run -v
```

### `FORTYTWO_PROFILE` env (per shell / per process)

```bash
FORTYTWO_PROFILE=judge1 fortytwo run
# or
export FORTYTWO_PROFILE=judge1
fortytwo capability
fortytwo run -v
```

Precedence: `-p` flag > env var > `profiles.json` active profile. In scripts, prefer the explicit flag — environment leakage between processes has bitten people.

---

## Register a new profile

```bash
# 1. Create the empty profile
fortytwo profile create judge1

# 2. Register it under that profile
fortytwo -p judge1 setup \
  --node-name "Judge-1" \
  --inference-type openrouter \
  --api-key "$OPENROUTER_KEY" \
  --model "qwen/qwen3.5-35b-a3b" \
  --node-role JUDGE

# 3. Back up its identity — each profile needs its own backup
cp ~/.fortytwo/profiles/judge1/identity.json \
   ~/backups/judge1-identity-$(date +%F).json
```

Or import a pre-existing identity:

```bash
fortytwo -p judge1 import \
  --node-id "..." --node-secret "..." \
  --inference-type openrouter --openrouter-api-key "$OPENROUTER_KEY" \
  --model "qwen/qwen3.5-35b-a3b" --node-role JUDGE
```

---

## Running many workers in parallel

Each `fortytwo -p <name> run` is an independent process. For N nodes, run N processes — one per profile. They share the same CLI binary and LLM provider but use different identities, so the server treats them as independent nodes.

```bash
for p in judge1 judge2 answerer1; do
  nohup fortytwo -p "$p" run -v \
    > ~/.fortytwo/profiles/"$p"/worker.log 2>&1 &
  echo "$p: PID $!"
done
wait
```

**Caveats:**
- The local viewer dashboard binds to `127.0.0.1:4242` — port clash if multiple workers run concurrently. First one wins; subsequent workers log a warning and skip the dashboard. Use `fortytwo -p <name> capability` to monitor the others, or coordinate ports through config (if/when the CLI exposes `viewer_port`).
- Rate limiting on the API side is *per-identity*, not per-host, so 10 profiles will hit 10× the combined request quota — sometimes good, sometimes exposes node-specific throttles.
- LLM provider quota is shared across profiles if they all point at the same OpenRouter key. Either split keys, cap `llm_concurrency` per profile, or use different local LLM endpoints.

---

## Batch operations

Same command, every profile:

```bash
for p in $(fortytwo profile list | awk 'NR>1 {print $1}'); do
  echo "=== $p ==="
  fortytwo -p "$p" capability
done
```

Scripted migrations:

```bash
# Bump model across every profile
for p in $(fortytwo profile list | awk 'NR>1 {print $1}'); do
  fortytwo -p "$p" config set model_name "meta/llama3-70b"
done
```

Restart everyone:

```bash
pkill -TERM -f 'fortytwo.*run'
sleep 2
for p in judge1 judge2 answerer1; do
  nohup fortytwo -p "$p" run -v > ~/.fortytwo/profiles/"$p"/worker.log 2>&1 &
done
```

---

## Anti-patterns

- Don't share an `identity.json` across profiles — the server tracks state per `node_id`, so a "duplicate" profile that imports the same identity will collide with its twin.
- Don't run the same profile's worker from two hosts concurrently (laptop + server). Heartbeats compete; answers may be rejected as duplicates.
- Don't hardcode `~/.fortytwo/identity.json` — in profile mode that file may not exist or may be the wrong node. Read from `~/.fortytwo/profiles/$PROFILE/identity.json`, or just shell out to `fortytwo -p $PROFILE identity`.
