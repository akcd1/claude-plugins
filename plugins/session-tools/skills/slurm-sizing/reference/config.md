# Slurm sizing — configuration

Config lives at `~/.claude/slurm-sizing/config.json`. Both `slurm-sizing` and `slurm-digest`
read it. If it does not exist, the skills bootstrap it (below).

```json
{
  "digest_cluster": "fry",
  "digest_user":    "alice",
  "table":   "~/.claude/slurm-sizing/table.md",
  "log":     "~/.claude/slurm-sizing/jobs.tsv",
  "archive": "~/.claude/slurm-sizing/digests",
  "multi_user_digest": false,
  "policy": {
    "margin_multiplier": 2,
    "mem_floor_gb":      8,
    "mem_round_gb":      4,
    "cpu_headroom":      1.5,
    "cpu_floor":         2
  }
}
```

| Key | Default | Meaning |
|---|---|---|
| `digest_cluster` | **none — must be set** | The Slurm cluster whose weekly usage digest feeds this system. Compared against the local cluster before anything is consulted or logged. |
| `digest_user` | the invoking `$USER` | Which user's rows to merge. The digest carries a `User` column and may contain several people's jobs. |
| `table` | `~/.claude/slurm-sizing/table.md` | The sizing table. |
| `log` | `~/.claude/slurm-sizing/jobs.tsv` | The submission log where scope is recorded. |
| `archive` | `~/.claude/slurm-sizing/digests` | Directory of raw archived digests, one per date. |
| `multi_user_digest` | `false` | This digest is known to contain other users' rows and should be filtered without halting. |
| `policy.margin_multiplier` | `2` | `rec_mem` = this × peak. |
| `policy.mem_floor_gb` | `8` | Never recommend less than this. |
| `policy.mem_round_gb` | `4` | Round `rec_mem` UP to a multiple of this. |
| `policy.cpu_headroom` | `1.5` | `rec_cpu` = ceil(ReqCPU × CPUPct/100 × this), capped at `ReqCPU`. |
| `policy.cpu_floor` | `2` | Never recommend fewer CPUs than this. |

**The `policy` defaults are the reviewed values and changing them changes the safety properties.**
In particular `mem_round_gb` rounds **UP**, never to nearest — a safety margin that rounds down is
not a margin. A site that lowers `margin_multiplier` is choosing more OOM risk in exchange for
queue priority, and should say so to itself explicitly.

**The `ReqCPU` cap is not optional.** Full rule:
`rec_cpu = min(ReqCPU, max(cpu_floor, ceil(ReqCPU × CPUPct/100 × cpu_headroom)))`.
Without the cap, a job at 4 CPUs and 90% utilisation computes `ceil(4 × 0.9 × 1.5) = 6` —
recommending more CPUs than were requested. The cap is present in the working system and must
survive the port.

**`digest_user` and multi-user digests.** Merge only rows whose `User` matches `digest_user`.
Another person's peak for a same-named job would corrupt this user's recommendation with no error
anywhere. A lab-wide digest is a legitimate input; guessing whose rows to keep is not.

Precisely what to do when other users' rows appear — **ask once, then remember**:

- **First time:** HALT before merging anything. Report which other users are present and how many
  rows each has, and ask whether this is a shared digest that should be filtered to `digest_user`.
- **On confirmation:** write `"multi_user_digest": true` into the config, then proceed, filtering
  to `digest_user` and reporting the filtered-out count each week thereafter. Do not halt again.
- **If `multi_user_digest` is already `true`:** filter and report, never halt.
- **If the user declines** (this is not a shared digest to filter): merge nothing and stop. Do NOT
  write `multi_user_digest`. Report the mismatch and suggest checking that `digest_user` names the
  right account and that the pasted digest is the intended one. This condition re-halts on the next
  run by design — nothing was resolved, so proceeding silently would be worse than asking again.
- **Never** filter silently on a first encounter, and never merge another user's rows at all.

`digest_user` defaults to the invoking `$USER`, but the two can legitimately differ — a site where
the Slurm account name is not the login name is exactly why this is a config key and not an
inference. If no row matches `digest_user`, say so rather than merging nothing silently.

**`digest_cluster` deliberately has no default.** Guessing it is the one error that silently
corrupts results: sizing advice from the wrong cluster's data is worse than no advice, because job
IDs are not unique across clusters. If it is unset, ask the user and write their answer to the
config — never infer it from the local hostname, which is exactly the case where the answer may
differ.

**Determining the local cluster:** `sacctmgr -n -P list cluster format=Cluster`, falling back to
`scontrol show config | grep ClusterName`. If neither works, treat the local cluster as unknown and
say so rather than proceeding.

**Path handling.** JSON values (`table`, `log`, `archive`) may use `~` and must be expanded before
use; create parent directories as needed when bootstrapping.

## Bootstrap (first run, nothing exists yet)

- **Config missing:** ask the user for `digest_cluster`, write the file with that value and the
  defaults above. **Before writing defaults, check for a pre-plugin layout** — `~/.claude/slurm-sizing.md`,
  `~/.claude/slurm-jobs.tsv`, `~/.claude/slurm-digests/`. Those are where an early hand-rolled
  version kept its data. If any exist, offer to point the config at them instead of creating empty
  files beside them. Silently starting fresh next to a populated table would strand real history.
- **Table missing:** create it with the header and the "How to read a row" prose, and zero data
  rows. An empty table is valid — it means every job is unknown, and an unknown job is a reason to
  measure, not to size from habit.
- **Log missing:** create it with the `#` comment header explaining the `(cluster, jobid)` join key,
  then the column header `cluster	jobid	submitted	job_name	scope	cwd` (tab-separated).
- **Archive dir missing:** create it.

Never bootstrap silently — report each file created, because a user who expected existing data
needs to know the skill did not find it.
