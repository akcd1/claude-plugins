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
  about Slurm resource sizing. SKIP only when no cluster job is involved at any remove, or when
  the user has turned this system off (`"enabled": false` in its config).
user-invocable: true
allowed-tools: Read, Write, Edit, Bash(test *), Bash(mkdir *), Bash(sacctmgr *), Bash(scontrol *), Bash(grep *)
---

# Slurm Job Sizing

Full config contract — keys, defaults, the table and log file formats, and the shared
"determine the local cluster" procedure — is in
[reference/config.md](reference/config.md). Consult it whenever any of the files below
don't exist yet, and whenever you need the exact column layout of the table or the log.

The table this skill reads is filled in by merging a weekly Slurm usage digest with
`/session-tools:slurm-digest`. Where that digest comes from, and the fact that this skill is inert
until the first one is merged, are documented in
[reference/config.md](reference/config.md) → "Where the weekly digest comes from".

## 0. Off switch — check this before the cluster gate

Read `~/.claude/slurm-sizing/config.json` (this path is fixed; only the `table`/`log`/`archive`
data paths inside it are configurable).

- **`"enabled": false`** — the user has declined this system. Do nothing: do not consult the
  table, do not log, and do **not** prompt or offer to re-enable. Stay silent.
- **Config file does not exist:** do not guess anything. Go to **Bootstrap** (§6) first.
- Otherwise (`enabled` absent or `true`): continue to §1.

## 1. Cluster gate — check this before anything else

1. Read `digest_cluster` from the config.
2. Determine the local cluster using the shared procedure in
   [reference/config.md](reference/config.md) → "Determining the local cluster":
   `sacctmgr -n -P list cluster format=Cluster`, falling back to
   `scontrol show config | grep ClusterName` (take the value after the `=`, trimmed). **Never use
   `hostname`** — it names a node, not a cluster.
   - **Both commands fail or return nothing:** the local cluster is undeterminable. Say so
     explicitly and STOP — do not consult the table, do not append to the log. "Unknown" is never
     treated as a match.
3. Compare the two values — an exact string match.
   - **They match:** proceed to §2.
   - **They differ:** say so explicitly (name both clusters) and STOP. Do not consult the table
     and do not append to the log. This system is inert on any cluster other than the one the
     digest was built from — job IDs collide across clusters (see §5), so numbers from the wrong
     cluster are worse than no numbers at all.

## 2. Before sizing any job — read the table first

Read the table at the configured `table` path (default `~/.claude/slurm-sizing/table.md`). Its
nine columns (`job_name`, `peak_MB`, `peak_G`, `n`, `scope`, `last_seen`, `rec_mem`, `rec_cpu`,
`notes`) and how to read each one are specified in
[reference/config.md](reference/config.md) → "Table file format".

- **Table file does not exist:** go to **Bootstrap** (§6).
- **Job name is present in the table:** state which row you used, its `rec_mem` / `rec_cpu`, its
  `n` (`n = 1` is a single observation, not a characterisation) — **and its `scope`**. Then apply
  the scope check below **before** treating any number as usable.
- **Job name is absent from the table:** say so explicitly, then either ask what scope to expect
  or start small and measure. Never fall back to a remembered/habitual number ("jobs like this
  usually need 64G") — an absent row is a reason to measure, not a reason to guess.

### The scope check — do this every time, before quoting a number as usable

**A recorded scope makes a row *checkable*. It does not make it *actionable*.** Those are
different claims, and conflating them is how this system produces a confidently wrong answer.
`scope` records what the measurement covered; whether the number is safe for *this* run depends on
what you are about to submit, which only you know — the merge step could not have known it.

State the row's `scope` prominently, then compare it to the run about to be submitted:

- **No declared scope (`rec_mem` carries `>=`, or `scope` is empty / `unknown` / a `derived:` value
  a human has not confirmed):** a lower bound from a run whose workload was never recorded, so the
  true peak could be higher. It may justify *raising* the request above the table value; it must
  never justify *lowering* the request below it.
- **Declared scope, and the intended run is the SAME scope:** the number is usable as-is. Say which
  scope you matched (e.g. "the row was measured at `1565 tiles (FULL)`, which is what you're about
  to run").
- **Declared scope, but the intended run is LARGER:** **do not use this number to size down.**
  Treat it exactly as a lower bound — `>=` in effect — and say why: the measurement covered less
  work than the run being sized, so it is evidence about a smaller job, not about this one. A row
  reading `36G` measured at "5 of 100 units" is **not** a 36G recommendation for all 100.

  **Say plainly that the table alone cannot answer this** — it holds a measurement of a
  different-sized run, and no amount of reading it turns that into a number for this one. Then
  produce an actual answer, taking the first of these three routes that applies. Do not stop at the
  refusal: leaving the researcher with no number is not a safe outcome, it just moves the guess
  somewhere nobody can see it.

  1. **A scaling parameter is known** (samples, tiles, cells, perturbations, reads — whatever the
     scope is counted in): scale the measured peak by the ratio of intended to measured units,
     apply the normal headroom, and **label the result an estimate, not a measurement**. Show the
     arithmetic and the assumption — e.g. "measured 27.5 G at 5 of 100 samples; 20x the work, so
     ~550 G plus headroom — an estimate that assumes memory scales linearly with samples, which it
     may not." If you have reason to think it is not linear, say which way you expect it to be
     wrong.
  2. **No scaling parameter, but a larger run of this job has succeeded before:** start from that
     run's request. A request that actually completed is evidence; the table's smaller-scope row is
     not evidence about this size.
  3. **Neither:** measure first. Run a small or intermediate case, merge the resulting digest with
     `/session-tools:slurm-digest`, and size the big run from the row that produces. Committing a
     large reservation on an unscaled subset measurement is exactly the guess this system exists to
     replace.

  **Never present the table's number as if it covered the larger run** — not as the recommendation,
  and not silently as a floor without saying that a floor is all it is.

  Whichever route you take, **record the scope at submission time per §4.** That is the whole
  mechanism by which the next person asking this question gets a better answer than you did.
- **Declared scope, but the intended run is SMALLER:** a smaller run is not evidence for a bigger
  one, and this row *is* the bigger one — the number is safe but likely generous. You may use it;
  note that it was measured at a larger scope, and that measuring the smaller run would tighten it.
- **Cannot determine the intended run's scope, or cannot tell how it compares: HARD STOP — ask.**
  This is a halt, not a suggestion (see "Hard stops" below). Do not assume they match. "Same scope"
  is a claim about the workload, not a default.

This check is the reason `scope` exists in the table at all. Skipping it turns a subset measurement
into a full-run recommendation, which is the specific under-provisioning failure the column was
added to prevent.

### Hard stops

A **HARD STOP** in this skill means: stop, report what you need, and **do not proceed on a
default**. It is not a prompt you may skip when no one answers.

- **Running unattended is not permission to continue.** If there is no human to answer, **stop and
  report** — state the question, what you would have needed, and that no sizing was produced. An
  agent operating without a user must not pick a value and carry on.
- **Why these hold the whole mechanism together:** proceeding silently past an unanswered scope
  question is precisely how an unsafe number gets used. Every other safeguard here — `>=`, the
  running max, the scope check — is designed so that the *conservative* answer is the automatic
  one. A hard stop marks the places where there is no conservative answer to fall back to, only a
  guess.
- The hard stops in this skill are: **the undeterminable-scope branch above**, and **every question
  in Bootstrap (§6)** — `digest_cluster`, `digest_user`, and the decline option. Never invent a
  cluster name, an account name, or a scope.
- **Not every question is a hard stop.** Where a documented conservative fallback exists and is
  applied, taking it is correct — e.g. `slurm-digest`'s Step 10 scope prompt, whose unanswered
  outcome is `unknown` plus a `>=` prefix, and whose merge completes. The distinction is whether
  continuing requires *assuming* something unverified.

## 3. Never size from raw `sacct` MaxRSS on an I/O-heavy job

`MaxRSS` on a job that reads or writes a lot of data includes kernel page cache charged to that
job's cgroup — it is not memory the job actually demanded, and sizing from it can overstate the
real requirement by a large multiple. Rows whose `notes` column contains `IO-BOUND` are hand-pinned
from in-process measurement instead — trust those over any `sacct`-derived figure for the same job,
and don't recompute an `IO-BOUND` row from `sacct`.

**Setting the flag** (this is how a row *becomes* `IO-BOUND` — nothing else sets it):

1. Get **in-process** evidence of the real peak RSS — `resource.getrusage(...).ru_maxrss` inside
   the job, a `psutil`-style RSS sampler, or a memory profiler. A digest or `sacct` number can
   never justify the flag: distrusting those numbers is the entire point of it.
2. Set the row's `peak_MB` to that measured peak, and `peak_G` to its display rounding.
3. Set `rec_mem` **by hand**, from the same in-process evidence — the measured demand plus whatever
   headroom that evidence justifies. See the next paragraph: this is deliberately not the formula.
4. **Append** `IO-BOUND` to `notes`, preserving whatever text is already there, and record where
   the measurement came from — e.g. `IO-BOUND (getrusage peak 3.2G, 2026-08-03); CPUPct 45`.

**An `IO-BOUND` row's `rec_mem` will not reconcile with the formula, and that is correct.** On every
other row `rec_mem` is `peak_GB × (1 + headroom_frac)`, floored and rounded up. On a pinned row it
is **hand-set from in-process evidence and deliberately NOT derived from `peak_MB`** — the two
numbers can come from different measurements of different things, so applying the formula to the
row's `peak_MB` may well give a different answer (a small pinned `peak_MB` can sit beside a larger
hand-set `rec_mem`, or the reverse). **A mismatch there is expected and must not be "corrected".**
Recomputing a pinned row from its `peak_MB` silently discards the in-process evidence that is the
only reason the row exists. If a pinned row looks inconsistent, that is the flag working, not a
data-integrity bug.

Removing the flag requires the same class of evidence that set it — never a digest, never `sacct`.

## 4. After submitting — log what it actually ran

Append one tab-separated row to the log (default `~/.claude/slurm-sizing/jobs.tsv`):

```
cluster	jobid	submitted	job_name	scope	cwd
```

- **Log file does not exist:** go to **Bootstrap** (§6), then append the row.
- `cluster` must be the exact `digest_cluster` string from the config — the digest merge filters
  log rows by exact match on this column, so a variant spelling makes the row invisible.
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

Full file formats, the literal table header, and the rationale are in
[reference/config.md](reference/config.md) → "Bootstrap". The branches, so none gets skipped:

**Every question in this section is a HARD STOP** (see §2, "Hard stops"): `digest_cluster`,
`digest_user`, and the decline option all require an answer from the user. Running unattended is not
permission to pick one — stop and report that bootstrap could not complete. A guessed cluster name
makes the skill inert or wrong forever, and a guessed account name filters out every one of the
user's own rows; neither failure announces itself.

- **Config missing:** check first for a pre-plugin layout — probe with `test`, since one of the
  three is a directory: `test -f ~/.claude/slurm-sizing.md`, `test -f ~/.claude/slurm-jobs.tsv`,
  `test -d ~/.claude/slurm-digests`. If any exist, offer to point the new config at them instead of
  starting empty files beside real history. Then establish
  `digest_cluster` — **do not ask for it in free text.** Run the §1 query, show the user the exact
  string it returned, and offer it as the answer, along with the option to name a different cluster
  and the option to decline ("not applicable / I have no weekly digest"). Declining writes
  `{"enabled": false}`, creates nothing else, and this skill then stays silent permanently (§0).
  Then **confirm `digest_user` rather than just writing it**: show the value you intend to use, say
  it came from `$USER`, state that it must match the digest's `User` column exactly, and let the
  user correct it — the two can legitimately differ, and a wrong value silently filters out every
  one of their own rows.
- **Table missing:** create it with the nine-column header row and the "How to read a row" prose
  given verbatim in [reference/config.md](reference/config.md) → "Table file format", and zero data
  rows. An empty table is a valid, complete state — every job is unknown, which is a reason to
  measure, not a reason to size from habit.
- **Log missing:** create it with the `(cluster, jobid)` join-key comment header, then the
  tab-separated column header `cluster	jobid	submitted	job_name	scope	cwd`.
- **Archive dir missing:** create it (empty) with `mkdir -p`.

Report every file created during bootstrap — a user who expected existing data needs to know the
skill didn't find it, rather than silently starting fresh next to it.

## 7. Weekly maintenance

This table is only as good as the digests merged into it. When the week's Slurm usage digest email
arrives, merge it with `/session-tools:slurm-digest`, passing the week-ending date along with the
pasted digest.
