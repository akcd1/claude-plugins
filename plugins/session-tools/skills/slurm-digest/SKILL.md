---
name: slurm-digest
description: >-
  Merge a weekly Slurm usage digest into the sizing table used by slurm-sizing. User-invoked via
  /session-tools:slurm-digest — paste the week's usage digest as the argument AND give the
  week-ending date it covers, then run it explicitly; this skill does not trigger on its own the
  way slurm-sizing does.
user-invocable: true
allowed-tools: Read, Write, Edit, Bash(mkdir *), Bash(grep *), Bash(awk *), Bash(sacct *), Bash(sacctmgr *), Bash(scontrol *)
---

# Slurm Digest Merge

## Where this digest comes from

This skill does **not** produce a digest; it merges one you already have. At Whitehead, users who
ran Slurm jobs are sent a usage digest **by email, once a week** — a table of each job's requested
versus used memory and CPU. That email is the input: paste its table below and give the
week-ending date it covers.

Consequently the whole system is **inert until the first digest is merged**. A fresh install has an
empty sizing table, so `slurm-sizing` reports every job as unknown — which is a reason to measure,
not licence to guess. There is no substitute input; do not invent an `sacct` recipe to synthesise
one (`sacct` is used later only to *annotate* rows this digest already contains).

## Config

Full config contract — keys, defaults, the table and log file formats, the shared "determine the
local cluster" procedure, and the multi-user filtering rules referenced in Step 3 below — is in
[../slurm-sizing/reference/config.md](../slurm-sizing/reference/config.md). Read `enabled`,
`digest_cluster`, `digest_user`, `table`, `log`, `archive`, `multi_user_digest`, and `policy` from
`~/.claude/slurm-sizing/config.json` before anything else below; if that config doesn't exist yet,
follow its "Bootstrap" section first rather than guessing any of these values.

**If `enabled` is `false`**, the user has declined this system. Merge nothing. Report that it is
turned off and offer to re-enable it (set `"enabled": true` and run bootstrap); proceed only if
they confirm.

Merge the Slurm digest pasted below into the sizing table at the configured `table` path
(default `~/.claude/slurm-sizing/table.md`). The table's nine columns are `job_name`, `peak_MB`,
`peak_G`, `n`, `scope`, `last_seen`, `rec_mem`, `rec_cpu`, `notes` — see
[../slurm-sizing/reference/config.md](../slurm-sizing/reference/config.md) → "Table file format"
for what each one means and the exact header to write if the table has to be created.

## Procedure

1. **Establish the week-ending date, then refuse duplicates (precondition — check this FIRST,
   before parsing or merging anything).** The digest's date is **a required argument**, supplied
   alongside the pasted table (e.g. `/session-tools:slurm-digest 2026-08-01 <pasted digest>`). The
   digest body itself contains no date column, so it cannot be recovered from the paste.
   **If the date is missing, ASK for it — never assume today's date.** Defaulting to today lets the
   same digest pasted on two different days archive under two filenames, pass this duplicate check
   both times, and double `n` — exactly the corruption this step exists to prevent.
   With the date in hand, check whether `<archive>/YYYY-MM-DD.tsv` already exists for it (`Read`
   it; a "no such file" error is the not-archived case), where `<archive>` is the configured
   `archive` path (default `~/.claude/slurm-sizing/digests`).
   **If it exists, STOP here: change nothing** — do not parse, join, merge, or write anything —
   report that the digest is already archived for that date, and ask whether to force. Re-merging
   an already-counted digest would inflate `n` (nine `score` runs would read as eighteen), and `n`
   is the signal that says whether a recommendation is trustworthy. Only proceed past this step
   if no file exists for that date, or the user has explicitly instructed you to force a
   re-merge.

2. **Parse.** Read the pasted table. Columns are, in order: `User`, `JobID`, `JobName`, `ReqMem`,
   `UsedMem`, `MemPct`, `ReqCPU`, `UsedCPU`, `CPUPct`, `Elapsed`. Memory columns are in
   **megabytes**. Where a `(M)` suffix appears on those column names, it states the unit and is
   **not** part of the header text: do not require the literal `(M)` to be present, and do not
   reject a digest over it. Skip the `OVERALL` line, skip `bash`, `python3`, and any bare shell
   or interpreter name.

3. **Filter to `digest_user` — a consent-gated filtering procedure, not part of parsing.**
   Merge only rows whose `User` matches the configured `digest_user` (default: the invoking
   `$USER`). Another person's peak for a same-named job would corrupt this user's recommendation
   with no error anywhere, so this filter runs before anything downstream sees a row. Precisely
   what to do when other users' rows appear — ask once, then remember:
   - **First time** (`multi_user_digest` is unset or `false` in config) **and other users are
     present:** HALT before merging anything. Report which other users are present and how many
     rows each has, and ask whether this is a shared digest that should be filtered to
     `digest_user`.
   - **On confirmation:** write `"multi_user_digest": true` into the config, then proceed,
     filtering to `digest_user` and reporting the filtered-out count each week thereafter. Do not
     halt again.
   - **If `multi_user_digest` is already `true`:** filter and report, never halt.
   - **If the user declines** (this is not a shared digest to filter): merge nothing and stop. Do
     NOT write `multi_user_digest`. Report the mismatch and suggest checking that `digest_user`
     names the right account and that the pasted digest is the intended one. This condition
     re-halts on the next run by design — nothing was resolved, so proceeding silently would be
     worse than asking again.
   - **Never** filter silently on a first encounter, and **never** merge another user's rows at
     all, under any branch above.
   - If no row matches `digest_user`, say so explicitly rather than silently merging nothing.

4. **Exclude capped runs.** Drop any row with `MemPct >= 99`. Its MaxRSS is a ceiling imposed
   by `--mem`, not a measured peak. Note in the report how many rows were dropped this way.

5. **Look up scope in the log, keyed on `(cluster, jobid)` — never `jobid` alone.** The log at the
   configured `log` path (default `~/.claude/slurm-sizing/jobs.tsv`) is tab-separated and begins
   with `#`-prefixed comment lines before its header row
   `cluster	jobid	submitted	job_name	scope	cwd`.
   **Digests come from the configured `digest_cluster` only; the log's rows carry their own
   `cluster` value, which may name a different cluster for jobs submitted elsewhere.**
   `digest_cluster` and other clusters are separate Slurm ID namespaces — the same `jobid` number
   can label unrelated jobs on each one — so a bare `jobid` lookup is a correctness bug: it will
   silently splice an unrelated cluster's job onto this digest's scope and produce a confidently
   wrong recommendation. Joining on `jobid` alone is never acceptable, even as a shortcut.

   **Do this with a real shell command, not by reasoning over `Read` output**, so the composite key
   and the tab delimiter are actually enforced. One `awk` lookup does all of it — comment
   stripping, the `cluster` equality test, and the `jobid` membership test — in a single pass:

   ```bash
   grep -v '^#' "$LOG" | awk -F'\t' -v c="$DIGEST_CLUSTER" -v ids="$JOBIDS" '
     BEGIN { n = split(ids, a, ","); for (i = 1; i <= n; i++) want[a[i]] = 1 }
     $1 == c && ($2 in want) { print $2 "\t" $5 }
   '
   ```

   where `$JOBIDS` is the comma-separated list of `JobID`s from the surviving digest rows. Each
   output line is `jobid<TAB>scope` for a row that matched **both** halves of the key.

   Three properties are load-bearing and must not be dropped:
   - **`-F'\t'`** — the file is tab-separated and `scope` is free text containing spaces. Default
     whitespace splitting shatters `scope` across fields and corrupts the result.
   - **`$1 == c`** — log rows from any other cluster are excluded before `jobid` is even consulted.
     This is the composite key; `awk` is used here precisely because `join` takes a single key
     field and needs both inputs pre-sorted, while this log is append-ordered.
   - **`grep -v '^#'`** — the comment header is not data.

   A digest row with no output line gets `scope: unknown`. **If zero rows match, say so
   prominently** — a silently failing lookup makes every recommendation a lower bound and the
   system quietly stops working. Because `slurm-sizing` refuses to log on any cluster other than
   `digest_cluster`, a populated log should contain `digest_cluster` rows; a zero match against a
   non-empty log means the lookup is broken, not that there was nothing to find.

6. **Enrich unmatched rows from `sacct` — a conditional recovery procedure, not part of the
   lookup.** For every digest row that did NOT match a submission-log entry in Step 5, attempt to
   recover a *derived* scope directly from Slurm before falling back to `scope: unknown`. The six
   bullets below are sequential and order-dependent — cluster guard, then batch query, then
   discard step rows, then collapse array rows, then the miss guard, then record — treat them as
   an ordered procedure to follow in full, not as loose elaboration on the lookup.
   - **Cluster guard, checked first.** `sacct` only sees the LOCAL cluster's accounting
     database — **verified in practice**: a job ID from the digest's cluster does not resolve
     via `sacct` run against a different cluster's accounting database (`sacct -L -j <jobid>`
     returns nothing for it there), and `sacctmgr -n -P list cluster format=Cluster` lists only
     the local cluster's name. Determine the local cluster with the **shared procedure** in
     [../slurm-sizing/reference/config.md](../slurm-sizing/reference/config.md) → "Determining the
     local cluster" (`sacctmgr -n -P list cluster format=Cluster`, falling back to
     `scontrol show config | grep ClusterName`, taking the value after the `=`) — **never
     `hostname`**, which returns a node name, not a cluster name, and would therefore fail this
     comparison on essentially every site while reporting the result as a deliberate skip. Compare
     it to the digest's cluster (`digest_cluster` from config). If they differ, **skip enrichment
     entirely** for the whole digest — do not query `sacct` — and say so plainly in the report
     (Step 12). If the local cluster is undeterminable, treat that the same way: skip and say so.
     Never fabricate a scope to compensate.
   - **Batch query, one field per call — never a combined multi-field query.** Query once for
     every unmatched row, not per row, but query `SubmitLine` and `WorkDir` in **separate** calls:
     `sacct -j <comma-separated jobids> --format=JobID,SubmitLine --parsable2 --noheader` and, only
     if `WorkDir` is wanted, `sacct -j <comma-separated jobids> --format=JobID,WorkDir --parsable2
     --noheader`. **Do not combine fields into one `--format=JobID,SubmitLine,WorkDir` call and do
     not "fix" this with `--delimiter`.** `--parsable2` separates fields with `|`, and a submit
     line can itself contain a `|` (e.g. an ordinary `--wrap="zcat x | seqkit stats"`), which
     silently turns one 3-field row into a 5-field row no fixed-column parse recovers — verified
     live: a combined `JobID,SubmitLine,WorkDir` query interleaves `WorkDir` into the wrong
     position whenever a wrapped command uses one or more pipes. No replacement delimiter is safe
     either, since any character you pick could itself appear in a submit line. Querying one field
     at a time makes each row exactly `JobID|<value>` — a `JobID` can never contain `|`, so
     splitting on the **first** `|` only is exact regardless of what the field holds. Keep this as
     two calls; do not re-merge them into one query.
   - **Discard step rows before treating a hit as real.** A base job (and every array task within
     it) returns extra rows for its steps — `<jobid>.batch`, `<jobid>.extern`, and any other
     `.`-suffixed step — with empty `SubmitLine`/`WorkDir`. Drop any returned row whose `JobID`
     contains a `.` before doing anything else with the result; only bare (`<jobid>`) or
     array-task (`<jobid>_<task>`) rows carry real data.
   - **Array jobs: map task rows back to the digest's base job, then collapse to one scope.** A
     digest row for an array job has a bare base `JobID` (e.g. `7932156`); `sacct` returns one row
     per task (`7932156_0` … `7932156_11`) plus that task's step rows. After discarding step rows,
     strip the `_<task>` suffix to correlate each surviving row back to the base job. The
     surviving task rows for one base job carry an identical `SubmitLine` (it's the same `sbatch`
     invocation) — take one as the scope. If they somehow differ, record the first and add a note
     flagging the discrepancy rather than inventing a merge. This case is exactly where the
     feature earns its keep: a 12-task array charged 32G/task is the scale signal enrichment
     exists to recover, and it is silently lost if task/step rows aren't collapsed first.
   - **Empty-result / retention / absent-jobid guard.** Three things all count as the same miss —
     record nothing and leave the row `scope: unknown`: (a) `sacct` errors or is unavailable;
     (b) a jobid returns a row with an empty `SubmitLine`; (c) a jobid returns **no row at all**
     (purged by retention, or never existed on this cluster). Do not treat any of the three as
     evidence of anything. Note in the report (Step 12) how many rows missed this way.
   - **Record the full `SubmitLine` verbatim, prefixed `derived:`, never bare.** Store the whole
     recovered submit line as-is — do not trim it to "just the interesting flags"; the untrimmed
     line is the evidence, and no extraction rule can pick out what mattered without silently
     discarding information (e.g. a `--mem=` token is still worth seeing at the Step 10 prompt).
     Write it into the row's `scope` column as `derived: <full SubmitLine>` — e.g.
     `derived: sbatch --dependency=afterok:7932155 --array=0-11%4 --mem=32G
     scripts/08_screen_zarr.sh` — visibly distinct from a declared scope (no prefix) that came
     from the lookup. **A `derived:` scope does NOT clear the `>=` prefix on `rec_mem`** — a submit
     line records what the measurement WAS, not that the next run shares that scope — so Step 8's
     asymmetric scope rule still treats the row as unknown-scope for sizing purposes until a
     human promotes it (Step 10).

7. **Merge the peak.** For each job name, update the two memory columns:
   ```
   peak_MB = max(existing peak_MB, this week's max UsedMem_MB)     # full precision, MB
   peak_G  = round(peak_MB / 1024, 1)                              # display only
   ```
   `peak_MB` is the **authoritative, full-precision** peak in megabytes and is what every later
   step computes from; `peak_G` is a display rounding of it and is never an input to anything.
   **The `/1024` is not optional, and it belongs only in the `peak_G` line.** `UsedMem` in the
   digest is in MB, so the `max` is MB-vs-MB with no conversion; dividing there, or *failing* to
   divide when writing `peak_G`, corrupts the column by roughly 1000x (a 4794.71 MB reading would
   display `peak_G = 4794` where `4.7` belongs). Add to `n`, set `last_seen` to the digest's
   week-ending date. **Never lower `peak_MB`** — it is a running maximum across all weeks ever
   merged, so a week in which the job ran a smaller input leaves it untouched.
   If a row predates the `peak_MB` column, seed it as **`(peak_G + 0.05) * 1024`** — `peak_G` is
   rounded to *nearest*, so a bare `peak_G * 1024` can sit up to 51.2 MB below the true peak and
   would lower the recommendation on migration (true peak 16394 MB shows as `peak_G 16.0`: true
   `rec_mem` 36 G, bare-seeded `rec_mem` 32 G). Say so in the report, and treat its `rec_mem` as a
   lower bound until the next digest.
   **Skip rows flagged `IO-BOUND`.** That flag lives in the `notes` column of the job's
   **existing row in the sizing table** (the configured `table` path) — never in the incoming
   digest, which carries no such marker. Check the current table row before merging, not the
   pasted digest; a digest row for an IO-BOUND job looks like any other row and its MaxRSS is
   page cache, not demand, so it must not enter the max. An `IO-BOUND` row is left **entirely
   untouched** by Steps 7, 8 and 9 — peak, recommendations and notes alike.

8. **Recompute `rec_mem` from the stored `peak_MB`.**
   `rec_mem = policy.margin_multiplier x (peak_MB / 1024)` (default `margin_multiplier = 2`),
   where `peak_MB` is **the row's stored full-precision peak after Step 7** — **not** this week's
   digest rows, and **not** the 1-decimal `peak_G` displayed in the table. Floor the result at
   `policy.mem_floor_gb` (default 8 G), then round UP to the next multiple of `policy.mem_round_gb`
   (default 4 G) — never down; this is a safety margin, so a peak that lands exactly on a multiple
   of `mem_round_gb` stays there, and any remainder pushes to the next multiple up. Worked example
   at the default values: a stored peak of 13.2 G gives `2 x 13.2 = 26.4`, which rounds up to
   28 G, not down to 24 G.

   **`rec_mem` must never decrease while `peak_MB` is unchanged.** Because it is a pure function of
   the stored `peak_MB` and the `policy` values, this holds automatically — but it is the property
   to check if the arithmetic is ever changed. Computing it from *this week's* peak instead would
   break it: a job that peaked at 14963.82 MB (`peak_G 14.6`, `rec_mem 32G`) and then ran a smaller
   input peaking at 5000 MB would keep `peak_G 14.6` but drop to `rec_mem 12G` — a recommendation
   below the row's own recorded peak, and an OOM on the next full-size run.

   **These are the reviewed defaults, not hardcoded law** — a site that lowers `margin_multiplier`,
   or any other `policy` value, in config is choosing more OOM risk in exchange for queue priority,
   and should make that choice deliberately, in config, not by editing this file.

   Then apply the asymmetric scope rule, comparing this week's `scope` (Step 5/6) against the one
   recorded in the row's `scope` column:
   - scope matches the recorded scope -> plain recommendation, no prefix
   - scope is larger than recorded -> do NOT lower; scale by the known ratio or keep the prior
     request and treat this run as a fresh measurement
   - scope is smaller than recorded -> a smaller scope is not evidence about the larger one; do
     NOT lower anything, treat it like unknown scope for recommendation purposes, and note the
     scope it was actually measured at
   - scope unknown (empty, or a `derived:` value not yet confirmed by a human) -> prefix `>=`; the
     row may justify raising a request, never lowering one

9. **CPU.** One formula across the whole range — no bands, no carve-outs:
   `rec_cpu = min(ReqCPU, max(policy.cpu_floor, ceil(ReqCPU x CPUPct / 100 x
   policy.cpu_headroom)))` — at the default values (`cpu_floor = 2`, `cpu_headroom = 1.5`),
   `rec_cpu = min(ReqCPU, max(2, ceil(ReqCPU x CPUPct/100 x 1.5)))`. **The `ReqCPU` cap is not
   optional:** without the outer `min(ReqCPU, ...)`, a job at 4 requested CPUs and 90%
   utilisation computes `ceil(4 x 0.9 x 1.5) = 6` — recommending more CPUs than were requested —
   so the cap brings it back down to 4. The `cpu_headroom` multiplier (default 1.5x) is
   deliberate headroom over the observed average, not a serial/parallel classification — e.g.
   a job at 25% of 16 requested CPUs used ~4 cores on average, so `ceil(16 x 0.25 x 1.5) = 6`.

   Record the observed `CPUPct` in `notes` **additively**: `notes` is one shared free-text column
   that also holds the load-bearing `IO-BOUND` flag and any human annotation. Update or append the
   `CPUPct` fragment while **preserving every other fragment already in the cell** — never
   overwrite the column with a bare `CPUPct <n>`. Use a separated form such as
   `IO-BOUND (getrusage peak 3.2G); CPUPct 45` so fragments stay individually editable.
   **`IO-BOUND` rows are exempt from this step**, as from Steps 7 and 8: do not recompute their
   `rec_cpu` and do not touch their `notes`. An overwritten note erases the flag, and the next
   week's page-cache MaxRSS then enters the peak — the exact failure the flag exists to prevent.

10. **Prompt for scope.** List unmatched job names ranked by wasted reservation
    `(ReqMem - rec_mem) x n`, largest first, capped at 10. Ask the user to supply scope for those.
    Rank by waste, not recency: scope on a 9-run 200 G job is worth ~1,584 G; scope on a one-off
    8 G job is worth nothing. Names left unanswered stay `unknown` and carry forward.
    **Show derived evidence alongside the prompt.** For any unmatched name Step 6 enriched with a
    `derived:` scope, display that derived submit line next to it so the human can confirm or
    correct it. **Both confirmation and correction promote it identically:** whether the human
    accepts the derived submit line as-is or supplies a corrected scope in its place, the result is
    a declared scope — drop the `derived:` prefix, and it is now governed by Step 8's normal
    asymmetric rule (eligible to have `>=` removed). It is the human's confirmation or correction
    that authorizes this, never the derived string by itself. Names with no derived evidence are
    prompted exactly as before.

11. **Persist, then archive.** This is the point the on-disk table changes: write the fully
    merged and recomputed table (Steps 7-10 applied) back to the configured `table` path now,
    preserving the nine-column header. Only after that write succeeds, write the raw pasted digest
    to `<archive>/YYYY-MM-DD.tsv` under the week-ending date from Step 1 (creating the configured
    `archive` directory with `mkdir -p` if it does not exist) — the duplicate check already
    happened in Step 1, so this write should never collide with an existing file under normal
    operation.

12. **Report.** State: the week-ending date merged, rows parsed, rows dropped as capped, scope
    matches from the log, job names updated, job names new, peaks that rose, and the total
    reservation delta if the recommendations were applied. Be explicit that `>=` rows are not
    actionable for sizing down.
    Also state: how many unmatched rows were enriched with a `derived:` scope from `sacct`, how
    many attempted enrichments missed (empty `SubmitLine` / retention gap), and whether
    enrichment was skipped entirely because the digest's cluster differs from the local cluster,
    or because the local cluster was undeterminable (the Step 6 cluster guard) — if skipped, say
    so explicitly rather than silently omitting the counts.
    Also state: how many rows were filtered out because their `User` did not match
    `digest_user` (Step 3), whenever that filtering applied.

## Rules

- Advisory only. Do not edit job scripts, launcher scripts, or any pipeline code. Do not submit,
  cancel, or modify jobs.
- Never lower `peak_MB` **by merging** — it is a running maximum, and a smaller week leaves it
  untouched. The one exception is a human setting the `IO-BOUND` flag on in-process evidence
  (a deliberate correction of an inflated `sacct` figure, not a merge), which may lower it.
  Never remove an `IO-BOUND` flag without in-process evidence
  (`getrusage`/`psutil`-style RSS sampling), because `sacct` MaxRSS cannot distinguish demand from
  page cache.
- If the digest's column layout differs from Step 2's order, stop and report rather than guessing.
  A missing `(M)` unit annotation in the header text is **not** a layout difference.

## Digest

Week-ending date and pasted digest:

$ARGUMENTS
