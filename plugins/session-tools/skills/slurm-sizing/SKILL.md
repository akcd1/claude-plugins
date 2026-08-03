---
name: slurm-sizing
description: >-
  Size Slurm jobs from measured usage instead of habit. TRIGGER — load this BEFORE writing or
  running anything that submits a cluster job, and immediately AFTER submitting one (to record
  what workload it ran). "Anything that submits a job" means: a literal `sbatch`/`srun`/`salloc`
  command; ANY wrapper, launcher, Makefile target or shell script that submits on your behalf
  (e.g. `./flow.sh --backend slurm`, `submit_*.sh`, any script containing `#SBATCH` directives,
  anything whose output includes "Submitted batch job"); and any choice of
  `--mem`/`--cpus-per-task`/`--time`. Also triggers on "run this on the cluster", "how much
  memory should this job get", "why did my job OOM", "am I over-requesting", or any question
  about Slurm resource sizing. SKIP only when no cluster job is involved at any remove.
user-invocable: true
allowed-tools: Read, Write, Edit, Bash(cat *), Bash(ls *), Bash(mkdir *), Bash(sacctmgr *), Bash(scontrol *), Bash(grep *)
---

# Slurm Job Sizing

Full config contract (keys, defaults, bootstrap file formats): `reference/config.md`. Consult it
whenever any of the files below don't exist yet.

## 1. Cluster gate — check this before anything else

1. Read `digest_cluster` from `~/.claude/slurm-sizing/config.json` (path may be overridden in the
   config itself).
   - **Config file does not exist:** do not guess anything. Go to **Bootstrap** (§6) first.
2. Determine the local cluster: `sacctmgr -n -P list cluster format=Cluster`, falling back to
   `scontrol show config | grep ClusterName`.
   - **Both commands fail or return nothing:** the local cluster is undeterminable. Say so
     explicitly and STOP — do not consult the table, do not append to the log. "Unknown" is never
     treated as a match.
3. Compare the two values.
   - **They match:** proceed to §2.
   - **They differ:** say so explicitly (name both clusters) and STOP. Do not consult the table
     and do not append to the log. This system is inert on any cluster other than the one the
     digest was built from — job IDs collide across clusters (see §5), so numbers from the wrong
     cluster are worse than no numbers at all.

## 2. Before sizing any job — read the table first

Read the table at the configured `table` path (default `~/.claude/slurm-sizing/table.md`).

- **Table file does not exist:** go to **Bootstrap** (§6).
- **Job name is present in the table:** use that row's `rec_mem` / `rec_cpu` and state which row
  you used (e.g. "using the `align_star` row: rec_mem=32G, rec_cpu=8").
  - **The value carries a `>=` prefix:** that's a lower bound from a run whose scope was never
    recorded, so the true peak could be higher. It may justify *raising* the request above the
    table value; it must never justify *lowering* the request below it.
- **Job name is absent from the table:** say so explicitly, then either ask what scope to expect
  or start small and measure. Never fall back to a remembered/habitual number ("jobs like this
  usually need 64G") — an absent row is a reason to measure, not a reason to guess.

## 3. Never size from raw `sacct` MaxRSS on an I/O-heavy job

`MaxRSS` on a job that reads or writes a lot of data includes kernel page cache charged to that
job's cgroup — it is not memory the job actually demanded, and sizing from it can overstate the
real requirement by a large multiple. Rows marked `IO-BOUND` in the table are hand-pinned from
in-process measurement (e.g. `resource.getrusage`, a memory profiler) instead — trust those over
any `sacct`-derived figure for the same job, and don't recompute an `IO-BOUND` row from `sacct`.

## 4. After submitting — log what it actually ran

Append one tab-separated row to the log (default `~/.claude/slurm-sizing/jobs.tsv`):

```
cluster	jobid	submitted	job_name	scope	cwd
```

- **Log file does not exist:** go to **Bootstrap** (§6), then append the row.
- `scope` is a short, concrete description of the workload this run did — e.g.
  "12 tasks x 1565 tiles (FULL)", "5 of 100 perturbations" — not a repeat of the job name.
- A peak logged with no `scope` is only ever a lower bound: once the job has finished, `sacct` has
  no way to recover what it actually processed. Write the scope now, while it's known, even before
  the peak usage is known.

## 5. Why `cluster` is the first column

Job IDs are not unique across clusters — two different clusters can each have a job `123456`. The
join key for this whole system is the pair `(cluster, jobid)`, never `jobid` alone. Putting
`cluster` first keeps that join key visible and stops one cluster's workload scope from silently
attaching to another cluster's job.

## 6. Bootstrap — config, table, or log missing

Full file formats and rationale: `reference/config.md` → "Bootstrap" section. The branches, so
none gets skipped:

- **Config missing:** ask the user for `digest_cluster` — there is no safe default (§1). Before
  writing a fresh file, check for a pre-plugin layout (`~/.claude/slurm-sizing.md`,
  `~/.claude/slurm-jobs.tsv`, `~/.claude/slurm-digests/`); if any exist, offer to point the new
  config at them instead of starting empty files beside real history.
- **Table missing:** create it with the header and explanatory prose, zero data rows. An empty
  table is a valid, complete state — every job is unknown, which is a reason to measure, not a
  reason to size from habit.
- **Log missing:** create it with the `(cluster, jobid)` join-key comment header, then the
  tab-separated column header `cluster	jobid	submitted	job_name	scope	cwd`.
- **Archive dir missing:** create it (empty).

Report every file created during bootstrap — a user who expected existing data needs to know the
skill didn't find it, rather than silently starting fresh next to it.

## 7. Weekly maintenance

This table is only as good as the digests merged into it. Use `/session-tools:slurm-digest` to
merge a new weekly usage digest into the table.
