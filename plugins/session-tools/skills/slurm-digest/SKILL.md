---
name: slurm-digest
description: >-
  Merge a weekly Slurm usage digest into the sizing table used by slurm-sizing. User-invoked via
  /session-tools:slurm-digest — paste the week's usage digest as the argument and run it
  explicitly; this skill does not trigger on its own the way slurm-sizing does.
user-invocable: true
allowed-tools: Read, Write, Edit, Bash(cat *), Bash(ls *), Bash(mkdir *), Bash(grep *), Bash(awk *), Bash(join *), Bash(sacct *), Bash(sacctmgr *), Bash(hostname)
---

# Slurm Digest Merge

Full config contract (keys, defaults, bootstrap file formats, and the multi-user filtering rules
referenced in Step 3 below): `../slurm-sizing/reference/config.md`. Read `digest_cluster`,
`digest_user`, `table`, `log`, `archive`, `multi_user_digest`, and `policy` from
`~/.claude/slurm-sizing/config.json` before anything else below; if that config doesn't exist
yet, follow its "Bootstrap" section first rather than guessing any of these values.

Merge the Slurm digest pasted below into the sizing table at the configured `table` path
(default `~/.claude/slurm-sizing/table.md`).

## Procedure

1. **Refuse duplicates (precondition — check this FIRST, before parsing or merging anything).**
   Determine the digest's date and check whether `<archive>/YYYY-MM-DD.tsv` already exists for it,
   where `<archive>` is the configured `archive` path (default `~/.claude/slurm-sizing/digests`).
   **If it does, STOP here: change nothing** — do not parse, join, merge, or write anything —
   report that the digest is already archived for that date, and ask whether to force. Re-merging
   an already-counted digest would inflate `n` (nine `score` runs would read as eighteen), and `n`
   is the signal that says whether a recommendation is trustworthy. Only proceed past this step
   if no file exists for that date, or the user has explicitly instructed you to force a
   re-merge.

2. **Parse.** Read the pasted table. Columns are `User JobID JobName ReqMem(M) UsedMem(M)
   MemPct ReqCPU UsedCPU CPUPct Elapsed`. Skip the `OVERALL` line, skip `bash`, `python3`, and
   any bare shell or interpreter name.

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

5. **Join for scope, on `(cluster, jobid)` — never `jobid` alone.** The log at the configured
   `log` path (default `~/.claude/slurm-sizing/jobs.tsv`) begins with `#`-prefixed comment lines
   before its header row — **filter them first**: run `grep -v '^#' <log path>` before running
   `awk`/`join` on that file. **This join is carried out with real shell commands, not by
   reasoning over `Read` output** — the failure mode below is what `join` itself does, and only
   manifests when it is actually invoked: treating the comments as data feeds `join` unsorted
   input (it will warn "is not sorted") and can produce false non-matches. The filtered log's
   header is
   `cluster jobid submitted job_name scope cwd`.
   **Digests come from the configured `digest_cluster` only; the log's rows carry their own
   `cluster` value, which may name a different cluster (e.g. `cluster-b`) for jobs submitted
   elsewhere.** `digest_cluster` and other clusters are separate Slurm ID namespaces — the same
   `jobid` number can label unrelated jobs on each one — so a bare `jobid` join is a correctness
   bug: it will silently splice an unrelated cluster's job onto this digest's scope and produce a
   confidently wrong recommendation. Joining on `jobid` alone is never acceptable, even as a
   shortcut. Before matching, **exclude every log row whose `cluster` is not the digest's own
   `digest_cluster`** — those rows are not eligible matches regardless of `jobid` equality. For
   each remaining digest row, look up `JobID` among the rows that survived that exclusion. A
   match supplies the workload scope. No match means `scope: unknown`. **If zero rows match
   against `digest_cluster` log rows and `digest_cluster` rows exist in the log, say so
   prominently** — a silently failing join makes every recommendation a lower bound and the
   system quietly stops working. (A zero-match result is *expected*, not a defect, when the
   filtered log contains no `digest_cluster` rows at all — e.g. it currently holds only rows
   whose provenance is some other cluster.)

6. **Enrich unmatched rows from `sacct` — a conditional recovery procedure, not part of the
   join.** For every digest row that did NOT match a submission-log entry in Step 5, attempt to
   recover a *derived* scope directly from Slurm before falling back to `scope: unknown`. The six
   bullets below are sequential and order-dependent — cluster guard, then batch query, then
   discard step rows, then collapse array rows, then the miss guard, then record — treat them as
   an ordered procedure to follow in full, not as loose elaboration on the join.
   - **Cluster guard, checked first.** `sacct` only sees the LOCAL cluster's accounting
     database — **verified in practice**: a job ID from the digest's cluster does not resolve
     via `sacct` run against a different cluster's accounting database (`sacct -L -j <jobid>`
     returns nothing for it there), and `sacctmgr -n -P list cluster format=Cluster` lists only
     the local cluster's name. Determine the local cluster (e.g. `sacctmgr -n -P list cluster
     format=Cluster` or `hostname`) and compare it to the digest's cluster (`digest_cluster` from
     config). If they differ, **skip enrichment entirely** for the whole digest — do not query
     `sacct` — and say so plainly in the report (Step 12). Never fabricate a scope to compensate.
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
     from the join. **A `derived:` scope does NOT clear the `>=` prefix on `rec_mem`** — a submit
     line records what the measurement WAS, not that the next run shares that scope — so Step 8's
     asymmetric scope rule still treats the row as unknown-scope for sizing purposes until a
     human promotes it (Step 10).

7. **Merge.** For each job name:
   `peak_G = max(existing_peak_G, this week's max UsedMem_MB / 1024)`.
   **The `/1024` is not optional.** `existing_peak_G` (the table's `peak_G` column) is already in
   GB; `UsedMem` in the digest is in MB (`UsedMem(M)`, per Step 2's column header) — skip the
   conversion and you corrupt `peak_G` by roughly 1000x (e.g. the first merge of a 4794.71 MB
   reading would write `peak_G = 4794` where `4.7` belongs). Add to `n`, update `last_seen`.
   Never lower a peak.
   **Skip rows flagged `IO-BOUND`.** That flag lives in the `notes` column of the job's
   **existing row in the sizing table** (the configured `table` path) — never in the incoming
   digest, which carries no such marker. Check the current table row before merging, not the
   pasted digest; a digest row for an IO-BOUND job looks like any other row and its MaxRSS is
   page cache, not demand, so it must not enter the max.

8. **Recompute `rec_mem`.** `rec_mem = policy.margin_multiplier x full-precision peak` (default
   `margin_multiplier = 2`), where full-precision peak is `max(UsedMem_MB) / 1024` — **not** the
   1-decimal `peak_G` value displayed in the table. Floor the result at `policy.mem_floor_gb`
   (default 8 G), then round UP to the next multiple of `policy.mem_round_gb` (default 4 G) —
   never down; this is a safety margin, so a peak that lands exactly on a multiple of
   `mem_round_gb` stays there, and any remainder pushes to the next multiple up. Worked example at
   the default values: a full-precision peak of 13.2 G gives `2 x 13.2 = 26.4`, which rounds up to
   28 G, not down to 24 G. **These are the reviewed defaults, not hardcoded law** — a site that
   lowers `margin_multiplier`, or any other `policy` value, in config is choosing more OOM risk in
   exchange for queue priority, and should make that choice deliberately, in config, not by
   editing this file. Then apply the asymmetric scope rule:
   - scope matches the recorded scope -> plain recommendation, no prefix
   - scope is larger than recorded -> do NOT lower; scale by the known ratio or keep the prior
     request and treat this run as a fresh measurement
   - scope is smaller than recorded -> a smaller scope is not evidence about the larger one; do
     NOT lower anything, treat it like unknown scope for recommendation purposes, and note the
     scope it was actually measured at
   - scope unknown -> prefix `>=`; the row may justify raising a request, never lowering one

9. **CPU.** One formula across the whole range — no bands, no carve-outs:
   `rec_cpu = min(ReqCPU, max(policy.cpu_floor, ceil(ReqCPU x CPUPct / 100 x
   policy.cpu_headroom)))` — at the default values (`cpu_floor = 2`, `cpu_headroom = 1.5`),
   `rec_cpu = min(ReqCPU, max(2, ceil(ReqCPU x CPUPct/100 x 1.5)))`. **The `ReqCPU` cap is not
   optional:** without the outer `min(ReqCPU, ...)`, a job at 4 requested CPUs and 90%
   utilisation computes `ceil(4 x 0.9 x 1.5) = 6` — recommending more CPUs than were requested —
   so the cap brings it back down to 4. The `cpu_headroom` multiplier (default 1.5x) is
   deliberate headroom over the observed average, not a serial/parallel classification — e.g.
   a job at 25% of 16 requested CPUs used ~4 cores on average, so `ceil(16 x 0.25 x 1.5) = 6`.
   Record the observed `CPUPct` in `notes`.

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
    merged and recomputed table (Steps 7-10 applied) back to the configured `table` path now.
    Only after that write succeeds, write the raw pasted digest to `<archive>/YYYY-MM-DD.tsv`
    (the configured `archive` path) — the duplicate check already happened in Step 1, so this
    write should never collide with an existing file under normal operation.

12. **Report.** State: rows parsed, rows dropped as capped, join matches, job names updated,
    job names new, peaks that rose, and the total reservation delta if the recommendations were
    applied. Be explicit that `>=` rows are not actionable for sizing down.
    Also state: how many unmatched rows were enriched with a `derived:` scope from `sacct`, how
    many attempted enrichments missed (empty `SubmitLine` / retention gap), and whether
    enrichment was skipped entirely because the digest's cluster differs from the local cluster
    (the Step 6 cluster guard) — if skipped, say so explicitly rather than silently omitting the
    counts.
    Also state: how many rows were filtered out because their `User` did not match
    `digest_user` (Step 3), whenever that filtering applied.

## Rules

- Advisory only. Do not edit job scripts, launcher scripts, or any pipeline code. Do not submit,
  cancel, or modify jobs.
- Never lower a peak; never remove an `IO-BOUND` flag without in-process evidence
  (`psutil`-style RSS sampling), because `sacct` MaxRSS cannot distinguish demand from page cache.
- If the digest's column layout differs from the above, stop and report rather than guessing.

## Digest

$ARGUMENTS
