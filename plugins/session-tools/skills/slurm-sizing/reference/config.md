# Slurm sizing — configuration

Config lives at `~/.claude/slurm-sizing/config.json`. Both `slurm-sizing` and `slurm-digest`
read it. If it does not exist, the skills bootstrap it (below).

```json
{
  "enabled": true,
  "digest_cluster": "<your-cluster-name>",
  "digest_user":    "<your-slurm-account>",
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

The two `<...>` values above are placeholders — fill them in with your own. Do not copy them
verbatim, and in particular do not copy a cluster name from anyone else's config: a wrong
`digest_cluster` is the one setting that corrupts every recommendation silently (below).

| Key | Default | Meaning |
|---|---|---|
| `enabled` | `true` | Master switch. When `false`, both skills stay silent and do nothing (see "Declining"). |
| `digest_cluster` | **none — must be set** | The Slurm cluster whose weekly usage digest feeds this system. Compared against the local cluster before anything is consulted or logged. Must be the cluster's exact Slurm name (below). |
| `digest_user` | the invoking `$USER` | Which user's rows to merge. The digest carries a `User` column and may contain several people's jobs. |
| `table` | `~/.claude/slurm-sizing/table.md` | The sizing table. |
| `log` | `~/.claude/slurm-sizing/jobs.tsv` | The submission log where scope is recorded. |
| `archive` | `~/.claude/slurm-sizing/digests` | Directory of raw archived digests, one per week-ending date. |
| `multi_user_digest` | `false` | This digest is known to contain other users' rows and should be filtered without halting. |
| `policy.margin_multiplier` | `2` | `rec_mem` = this × peak. |
| `policy.mem_floor_gb` | `8` | Never recommend less than this. |
| `policy.mem_round_gb` | `4` | Round `rec_mem` UP to a multiple of this. |
| `policy.cpu_headroom` | `1.5` | `rec_cpu` = ceil(ReqCPU × CPUPct/100 × this), capped at `ReqCPU`. |
| `policy.cpu_floor` | `2` | Never recommend fewer CPUs than this. |

**The `policy` values are tunable defaults, not invariants — but they are the reviewed defaults,
and changing them changes the safety properties.** In particular `mem_round_gb` rounds **UP**,
never to nearest — a safety margin that rounds down is not a margin. A site that lowers
`margin_multiplier` is choosing more OOM risk in exchange for queue priority, and should say so to
itself explicitly, in config, rather than by editing the skills.

**The `ReqCPU` cap is not optional.** Full rule:
`rec_cpu = min(ReqCPU, max(cpu_floor, ceil(ReqCPU × CPUPct/100 × cpu_headroom)))`.
Without the cap, a job at 4 CPUs and 90% utilisation computes `ceil(4 × 0.9 × 1.5) = 6` —
recommending more CPUs than were requested. The cap is present in the working system and must
survive the port.

## Where the weekly digest comes from

The digest is **not** something these skills generate. At Whitehead it arrives on its own: users
who ran Slurm jobs are sent a usage digest **by email, once a week**, listing each job's requested
versus used memory and CPU. `slurm-digest` is the merge step for that email — you paste its table
in, along with the week-ending date, and it folds the numbers into your sizing table.

The practical consequence: **both skills are inert until the first digest is merged.** A fresh
install has an empty table, so every job is "unknown" — which `slurm-sizing` reports as a reason to
measure, never as licence to guess. If your site does not send such a digest, the skills have no
input and there is nothing to configure; see "Declining" below.

## Determining the local cluster — one procedure, used everywhere

`slurm-sizing` (its cluster gate), `slurm-digest` (its `sacct` enrichment guard) and bootstrap all
need the **local** Slurm cluster's name. They all use exactly this procedure — do not substitute
another:

1. `sacctmgr -n -P list cluster format=Cluster` — take the single name it prints.
2. If that prints nothing or fails: `scontrol show config | grep ClusterName` and take the value
   to the right of the `=`, trimmed of whitespace.
3. If neither yields a name, the local cluster is **unknown**. Say so and stop. "Unknown" is never
   treated as a match.

**Never use `hostname` for this.** It returns a node's name, not a Slurm cluster's name; the two
are different strings on essentially every site, so comparing a hostname against `digest_cluster`
fails silently and permanently disables whatever it was guarding.

## `digest_cluster` must be the exact Slurm cluster name

`digest_cluster` is **exact-matched** — against the local cluster name from the procedure above,
and against the `cluster` column of every log row. A colloquial or approximate answer ("the
cluster", "our HPC", a hostname, a capitalised variant) is not a failure that announces itself: it
makes `slurm-sizing`'s gate never match, so the skill goes inert forever, or it makes the digest
join never match, so every recommendation carries `>=` forever.

Therefore **bootstrap does not ask for it in free text.** It runs the query above, shows the user
the exact string, and offers that string as the answer (see Bootstrap). And the `cluster` column
written into every log row must carry that same exact string — it is what the digest merge
filters log rows on.

**`digest_cluster` deliberately has no default.** Guessing it is the one error that silently
corrupts results: sizing advice from the wrong cluster's data is worse than no advice, because job
IDs are not unique across clusters. Never infer it from the local hostname.

## `digest_user` and multi-user digests

Merge only rows whose `User` matches `digest_user`. Another person's peak for a same-named job
would corrupt this user's recommendation with no error anywhere. A lab-wide digest is a legitimate
input; guessing whose rows to keep is not.

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

## Declining — `"enabled": false`

Not everyone has a digest, and not every machine is a cluster. A user must be able to say "this
does not apply to me" **once** and never be asked again.

- Bootstrap always offers a decline branch ("not applicable / I have no weekly digest").
- Declining writes `{"enabled": false}` to `~/.claude/slurm-sizing/config.json` and creates
  nothing else — no table, no log, no archive directory.
- **`slurm-sizing` honours it by doing nothing**: when `enabled` is `false` it does not consult,
  does not log, and does **not** prompt. It is silent, permanently, until the user changes the
  file.
- **`slurm-digest` honours it too**, but it is only reachable by an explicit user invocation, so
  it reports that the system is disabled and offers to re-enable (set `enabled` to `true` and run
  bootstrap). It merges nothing unless the user confirms.

Re-enabling is editing one word in the config, and either skill will do it on request.

## Path handling

JSON values (`table`, `log`, `archive`) may use `~` and must be expanded before use; create parent
directories as needed when bootstrapping. The config file's own location is fixed at
`~/.claude/slurm-sizing/config.json` — only the data paths are configurable.

## Bootstrap (first run, nothing exists yet)

**Config missing:**

1. **Check for a pre-plugin layout first** — `~/.claude/slurm-sizing.md`, `~/.claude/slurm-jobs.tsv`,
   `~/.claude/slurm-digests/`. Those are where an early hand-rolled version kept its data. If any
   exist, offer to point the config at them instead of creating empty files beside them. Silently
   starting fresh next to a populated table would strand real history.
2. **Determine `digest_cluster` by running the query, not by asking in free text.** Run the
   "Determining the local cluster" procedure above, show the user the exact string it returned, and
   offer it as the answer — e.g. "Slurm reports this cluster's name as `<name>`; is that the
   cluster your weekly digest covers? [use it / enter a different name / not applicable]".
   - If the query returns nothing, say so and ask the user for the exact Slurm cluster name (the
     value `sacctmgr` would print on that cluster), warning that it is exact-matched.
   - If the user's digest covers a *different* cluster than the local one, take their name — but
     record it verbatim, and note that `slurm-sizing` will then be inert on this machine by design.
3. **Offer the decline branch** in the same question ("not applicable / I have no weekly digest").
   Declining writes `{"enabled": false}` and stops — see "Declining".
4. Write the file with `enabled: true`, that `digest_cluster`, and the defaults above.

**Table missing:** create it with the header row and the "How to read a row" prose below, and zero
data rows. An empty table is valid — it means every job is unknown, and an unknown job is a reason
to measure, not to size from habit.

**Log missing:** create it with the `#` comment header explaining the `(cluster, jobid)` join key,
then the column header `cluster	jobid	submitted	job_name	scope	cwd`
(tab-separated). The `cluster` value written into each row must be the exact `digest_cluster`
string (see above).

**Archive dir missing:** create it.

Never bootstrap silently — report each file created, because a user who expected existing data
needs to know the skill did not find it.

### Table file format (`table.md`) — write exactly this on bootstrap

The table is a GitHub-flavoured markdown table with **nine** columns, in this order. Both skills
read and write these column names; inventing a different header breaks the merge silently.

```markdown
# Slurm sizing table

Measured memory and CPU usage per job name, merged from weekly usage digests by
`/session-tools:slurm-digest` and read by `slurm-sizing` before a job is submitted.

| job_name | peak_MB | peak_G | n | scope | last_seen | rec_mem | rec_cpu | notes |
|---|---|---|---|---|---|---|---|---|
```

**How to read a row**

- **`job_name`** — the Slurm `JobName` exactly as it appears in the digest. This is the lookup key;
  `slurm-sizing` matches the job it is about to submit against this column.
- **`peak_MB`** — the running maximum observed memory in **megabytes, at full precision**, across
  every digest ever merged. **This is the authoritative memory number**; everything else about
  memory is derived from it. It only ever rises (see `peak_G`).
- **`peak_G`** — `peak_MB / 1024`, rounded to 1 decimal, **for human reading only**. Never compute
  a recommendation from this column: it has already lost precision, and re-deriving `peak_MB` from
  it is not possible.
- **`n`** — how many weekly digests have contributed to this row. The confidence signal: `n = 1` is
  one observation, not a characterisation. Re-merging a digest inflates it, which is why an
  already-archived date is refused.
- **`scope`** — a short, concrete description of the workload the peak was measured at (e.g.
  "12 tasks x 1565 tiles (FULL)"). Empty or `unknown` means the peak is only a **lower bound** —
  nothing is known about what it was processing. A `derived: <submit line>` value is evidence
  recovered from `sacct`, not a human-declared scope, and still counts as unknown until confirmed.
- **`last_seen`** — the week-ending date of the most recent digest that contained this job.
- **`rec_mem`** — the recommended `--mem`, computed **from `peak_MB`**. A `>=` prefix means the
  scope was unknown: the value may justify *raising* a request, never *lowering* one.
- **`rec_cpu`** — the recommended `--cpus-per-task`, capped at the job's own `ReqCPU`.
- **`notes`** — free text, **additive**: appending a note must preserve what is already there.
  `IO-BOUND` in this column is a load-bearing flag, not a comment — see below.

**The memory columns and how they change**

On every merge, for each job name:

```
peak_MB = max(existing peak_MB, this week's max UsedMem_MB)      # full precision, MB, never lowered
peak_G  = round(peak_MB / 1024, 1)                               # display only
rec_mem = round_up_to(mem_round_gb,
                      max(mem_floor_gb, margin_multiplier * peak_MB / 1024))
```

`rec_mem` is a **pure function of the stored `peak_MB`** and the `policy` values. It is **not**
computed from this week's digest rows, and **not** from `peak_G`.

**`rec_mem` must never decrease while `peak_MB` is unchanged.** This is the property the whole
table exists to guarantee, and it is the reason `peak_MB` is stored at all. Without it, a job whose
peak was 14963.82 MB one week (→ `peak_G 14.6`, `rec_mem 32G`) and which runs a smaller input the
next week (5000 MB) would keep `peak_G 14.6` but recompute `rec_mem` down to `12G` from this week's
smaller number — a recommendation below the row's own recorded peak, and an OOM on the next real
run. Recomputing from the stored `peak_MB` makes that arithmetically impossible.

If a legacy table has no `peak_MB` column, add it and seed it as `peak_G * 1024` — a lossy but
never-lower starting point — say so in the report, and treat every such row's `rec_mem` as a lower
bound until the next digest refreshes it.

**The `IO-BOUND` flag: how it is set, and what it does**

`sacct`'s `MaxRSS` on a job that reads or writes a lot of data includes kernel page cache charged
to that job's cgroup. It is not memory the job demanded, and sizing from it can overstate the real
requirement by a large multiple. `IO-BOUND` in `notes` marks a row whose numbers are hand-pinned
from a real in-process measurement instead.

*Setting it* — this cannot be done from digest data, by construction:

1. Obtain **in-process** evidence of the job's real peak RSS: `resource.getrusage(...).ru_maxrss`
   inside the job, a `psutil`-style RSS sampler, or a memory profiler. `sacct`/digest numbers are
   exactly what the flag exists to distrust, so they can never justify setting it.
2. Set the row's `peak_MB` to that measured peak, and let `peak_G` / `rec_mem` follow from it by
   the formulas above.
3. **Append** `IO-BOUND` to the `notes` column, keeping any existing note text, and say where the
   measurement came from — e.g. `IO-BOUND (getrusage peak 3.2G, 2026-08-03); CPUPct 45`.

*What it does* — `slurm-digest` leaves an `IO-BOUND` row alone: its peak is not merged, its
`rec_mem`/`rec_cpu` are not recomputed, and its `notes` are not rewritten. `slurm-sizing` trusts
the pinned row over any `sacct`-derived figure for the same job.

*Removing it* requires the same class of evidence that set it — never a digest, never `sacct`.
