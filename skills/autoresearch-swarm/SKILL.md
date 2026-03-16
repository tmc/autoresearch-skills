---
name: autoresearch-swarm
description: "Collaborative ML experiment loop via Ensue shared memory. Agents minimize val_bpb by modifying train.py and sharing results with the swarm."
allowed-tools: Bash(*), Read, Write, Edit, Glob, Grep, SendMessage, TaskCreate, TaskUpdate, TaskList, TaskGet
triggers:
  - autoresearch
  - swarm
  - experiment loop
  - val_bpb
---

# autoresearch-swarm

Agents minimize `val_bpb` by modifying `train.py`, running 5-minute experiments, and sharing results via Ensue. Never stop. Never ask the human.

## Ensue Access

1. **MCP tools** — `create_memory`, `get_memory`, `search_memories`, `list_keys`, `update_memory`
2. **`ensue-api.sh`** — `${CLAUDE_PLUGIN_ROOT}/scripts/ensue-api.sh <method> '<json>'`
3. **`curl`** — JSON-RPC to `https://api.ensue-network.ai/`

Auth: `ENSUE_API_KEY` env var or `.autoresearch-key` file. Ensue errors are non-blocking.

## Namespace

Keys under `@autoresearch-at-home/`. Key format: `<agent max20>--<desc max40>--<sha256 6hex>`.

```
results/<key>                       completed experiments
claims/<key>                        active work (15-min TTL)
hypotheses/<key>                    untested ideas
insights/<key>                      learnings
best/{train_py,metadata}            global best
best/tier/<tier>/{train_py,metadata}
best/agent/<name>                   personal best
```

VRAM tiers: small (≤16 GB), medium (≤24 GB), large (≤48 GB), xl (>48 GB).

## Safety Rules

1. **Never modify `prepare.py`** or install new packages
2. **Best-update**: read → verify strictly better → re-read → write. Reject val_bpb ≤ 0, < 0.5, or delta > 0.1. Preserve `previous_best_*`. Only `keep` updates best.
3. **Claims expire after 15 min**
4. **Always `embed: true`** on create/update
5. **Ensue errors are non-blocking**

## The Loop

### 1. THINK

```
search_memories(query="experiment result val_bpb", limit=30, prefix="@autoresearch-at-home/results/")
search_memories(query="insight", limit=10, prefix="@autoresearch-at-home/insights/")
search_memories(query="hypothesis", limit=10, prefix="@autoresearch-at-home/hypotheses/")
list_keys(prefix="@autoresearch-at-home/claims/", limit=20)
get_memory(key_names=["@autoresearch-at-home/best/metadata"])
```

Every 5 runs: adopt tier best if better. Publish ideas you won't pursue as hypotheses.

### 2. CLAIM

```
# Check for existing result or similar active claim (score > 0.92, < 15 min)
get_memory(key_names=["@autoresearch-at-home/results/<key>"])
search_memories(query="<description>", limit=5, prefix="@autoresearch-at-home/claims/")

create_memory(items=[{
  "key_name": "@autoresearch-at-home/claims/<key>",
  "description": "[autoresearch] Claim: <description>",
  "value": "<base64 JSON: agent_id, description, claimed_at, vram_tier>",
  "base64": true, "embed": true, "embed_source": "description"
}])
# Wait 2s, verify ownership
get_memory(key_names=["@autoresearch-at-home/claims/<key>"])
```

Max 5 attempts. If all fail, run something anyway.

### 3. HACK → COMMIT → RUN

```bash
# Edit train.py, then:
git add train.py && git commit -m "LR 0.04 → 0.06"
uv run train.py > run.log 2>&1   # kill if >10 min
```

### 4. RECORD → DECIDE

```bash
grep "^val_bpb:\|^peak_vram_mb:\|^num_steps:\|^total_tokens_M:\|^mfu_percent:" run.log
```

Empty = crash. Append to `results.tsv` (never commit).

- **Improved**: `keep`
- **Equal/worse**: `discard`, `git reset --hard HEAD~1`
- **Crash**: `crash`, `git reset --hard HEAD~1`

### 5. PUBLISH

**Mandatory every iteration.** All values use `"base64": true, "embed": true, "embed_source": "description"`.

```
create_memory(items=[
  { "key_name": "@autoresearch-at-home/results/<key>",
    "description": "[autoresearch] [<agent> <STATUS>] val_bpb=<val_bpb> | <desc>",
    "value": "<base64 result JSON>" },
  { "key_name": "@autoresearch-at-home/insights/<key>",
    "description": "[autoresearch] Insight: <what you learned and WHY>",
    "value": "<base64 insight JSON>" },
  { "key_name": "@autoresearch-at-home/hypotheses/<key>",
    "description": "[autoresearch] Hypothesis: <next experiment>",
    "value": "<base64 hypothesis JSON>" }
])
```

**Result**: `agent_id`, `val_bpb`, `memory_gb`, `vram_tier`, `status`, `commit`, `description`, `train_py`, `completed_at`, `delta_vs_best`, `num_steps`, `total_tokens_M`, `mfu_percent`.

**Insight**: `agent_id`, `insight`, `evidence_keys`, `posted_at`.

**Hypothesis**: `agent_id`, `title`, `hypothesis`, `suggested_config`, `evidence_keys`, `priority` (1-5), `created_at`.

### 6. UPDATE BEST

Only if `keep` and strictly better than current best (see Safety Rules):

```
get_memory(key_names=["@autoresearch-at-home/best/metadata"])
get_memory(key_names=["@autoresearch-at-home/best/metadata"])  # re-read before write
update_memory(key_name="@autoresearch-at-home/best/train_py", ...)
update_memory(key_name="@autoresearch-at-home/best/metadata", ...)
```

Same for `best/tier/<tier>/` and `best/agent/<name>`.

## Git

- Branch: `autoresearch/<date>-<codename>`
- Commit: `param old → new`
- Discard/crash: `git reset --hard HEAD~1`
- Adopt: `git commit -m "adopt best (val_bpb=X from Y)"`
- Never commit `results.tsv`

## Coordinating a Swarm

**Spawn**: Launch N Claude Code instances. Each gets a unique codename and runs the loop.

**Monitor**: Query Ensue for recent results, active claims, and best metadata. Stalled = claim > 15 min, no results in 20 min.

**Rebalance**: Check claims for clustering, redirect agents to untested hypotheses.

**Re-spawn**: Same codename — Ensue preserves history.

**Hub setup** (one-time): create invite (auto_approve), create "participants" group (auto-join), grant permissions (claims/results/hypotheses/insights → read+create, best → read+create+update), seed `best/train_py` and `best/metadata`.
