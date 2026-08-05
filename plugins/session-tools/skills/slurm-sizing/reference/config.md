# Slurm sizing — configuration

Config lives at `~/.claude/slurm-sizing/config.json`. Both `slurm-sizing` and `slurm-digest`
read it. If it does not exist, the skills bootstrap it (below).

```json
{
  "enabled": true,
  "digest_cluster": "<ClusterName>@<short submit hostname>",
  "digest_user":    "<your-slurm-account>",
  "table":   "~/.claude/slurm-sizing/table.md",
  "log":     "~/.claude/slurm-sizing/jobs.tsv",
  "archive": "~/.claude/slurm-sizing/digests",
  "multi_user_digest": false,
  "policy": {
    "headroom_frac":     0.30,
    "mem_floor_gb":      8,
    "mem_round_gb":      4,
    "cpu_headroom":      1.5,
    "cpu_floor":         2
  }
}
```

Every `<...>` above is a placeholder — fill them in with your own. `digest_cluster` is the composed
identity `<ClusterName>@<short submit hostname>`: the cluster name from `scontrol show config`
(with a documented fallback chain, below) and the short hostname of the machine you **submit** from
— which is not always the machine you are typing on, see below. Do not copy a cluster identity from
anyone else's config, and do not shorten it to the bare `ClusterName`.
A wrong `digest_cluster` is the one setting that corrupts every recommendation silently.

| Key | Default | Meaning |
|---|---|---|
| `enabled` | `true` | Master switch. When `false`, both skills stay silent and do nothing (see "Declining"). |
| `digest_cluster` | **none — must be set** | The cluster whose weekly usage digest feeds this system, as the composed identity `<ClusterName>@<short submit hostname>` (e.g. `alpha@login-1`). Compared against the local identity before anything is consulted or logged. A bare `ClusterName` is **not** sufficient — see "Why the identity carries a host suffix". |
| `digest_user` | the invoking `$USER` | Which user's rows to merge. The digest carries a `User` column and may contain several people's jobs. |
| `table` | `~/.claude/slurm-sizing/table.md` | The sizing table. |
| `log` | `~/.claude/slurm-sizing/jobs.tsv` | The submission log where scope is recorded. |
| `archive` | `~/.claude/slurm-sizing/digests` | Directory of raw archived digests, one per week-ending date. |
| `multi_user_digest` | `false` | This digest is known to contain other users' rows and should be filtered without halting. |
| `policy.headroom_frac` | `0.30` | `rec_mem` = `peak_GB × (1 + this)`, floored and rounded up (below). |
| `policy.mem_floor_gb` | `8` | Never recommend less than this. |
| `policy.mem_round_gb` | `4` | Round `rec_mem` UP to a multiple of this. |
| `policy.cpu_headroom` | `1.5` | `rec_cpu` = ceil(ReqCPU × CPUPct/100 × this), capped at `ReqCPU`. When a job name has several rows in one digest, `CPUPct` and `ReqCPU` are taken from that name's **maximum-`CPUPct`** row (below). |
| `policy.cpu_floor` | `2` | Never recommend fewer CPUs than this. |

**The `policy` values are tunable defaults, not invariants — but they are the reviewed defaults,
and changing them changes the safety properties.** In particular `mem_round_gb` rounds **UP**,
never to nearest — a safety margin that rounds down is not a margin. A site that lowers
`headroom_frac` is choosing more OOM risk in exchange for queue priority, and should say so to
itself explicitly, in config, rather than by editing the skills.

**Why `headroom_frac` and not a multiplier — read this before changing it back.** `peak_GB × 1.30`
**is** arithmetically a 1.3x multiplier; the two forms are not different in kind, only in size.
The margin used to be `2x`, and that was calibrated against jobs running at roughly 3–15% memory
utilisation, where doubling a small peak is cheap. Measured against a digest whose jobs ran near
45% utilisation, `2x` recommended **more memory than the job had requested for 6 of 27 job names** —
e.g. a job that requested 512 G, peaked at 341.8 G and was therefore well sized got a 684 G
recommendation. Since an unknown-scope row "may justify raising a request", those are actionable,
so the tool would have pushed already-well-sized jobs *upward*, costing queue time and per-user
memory cap: the exact harm it exists to prevent, inverted. At `headroom_frac = 0.30` that same job
gets 448 G, and raises-above-request fall to 1 of 27 on the high-utilisation digest while staying
unchanged on the low-utilisation one. **Do not "restore" `2x` on the belief that a fractional
headroom is a categorically safer construction — it is the same construction with a smaller
number, and the smaller number is the point.**

**A recommendation above the request is still possible, and still intended.** The change removes a
*systematic* overshoot on well-utilised jobs; it does not remove the tool's ability to say that a
job genuinely needs more than it asked for. If a peak plus 30% exceeds the request, that is a real
signal about under-provisioning, not a bug to clamp away.

**Which row supplies `CPUPct` when a job name appears several times in one digest: the row with
the MAXIMUM `CPUPct`, paired with that same row's `ReqCPU`.** Not the peak-memory row — the
peak-memory run and the peak-CPU run are generally different runs, and pairing across them mixes
two measurements. Under-provisioning CPU is the harmful direction (a serialised job, not merely a
wasteful reservation), so this takes a running max exactly as the memory side does. A row with a
**blank** `CPUPct` contributes no CPU evidence and is excluded from that maximum; blank is absence
of evidence, not `0`.

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

## Determining the local cluster identity — one procedure, used everywhere

`slurm-sizing` (its cluster gate), `slurm-digest` (its `sacct` enrichment guard) and bootstrap all
need the **local cluster identity**. They all use exactly this procedure — do not substitute
another:

1. **Get the `ClusterName`** — a three-step chain, taken in order, so that an unreadable
   `scontrol` degrades the identity instead of disabling the skill:

   ```bash
   scontrol show config | grep -E '^ClusterName'      # 1. preferred
   sacctmgr -n -P list cluster format=Cluster         # 2. only if step 1 gave nothing
   ```

   - **Step 1** — take the value to the right of the `=`, trimmed of whitespace.
   - **Step 2** — if `scontrol` is unavailable, errors, or prints no `ClusterName`, ask the
     accounting layer instead. Take the single field, trimmed. If it returns several lines, this
     host is configured against more than one cluster and the name is ambiguous: treat that as
     step 2 having failed.
   - **Step 3 — neither yields a name:** use the **bare submit host as the whole identity** —
     resolved exactly as in step 2 below, not by a bare `hostname` — and **say so explicitly in the
     run's report**. This is the documented degraded mode, and it must be announced every time it
     is used: the user is otherwise silently keyed on something less specific than usual, with no
     way to tell from the output. It is still better than going inert, which is what an underivable
     identity used to cause.

2. **Get the short submit hostname** — the machine the command was typed on, with any domain suffix
   stripped:

   ```bash
   echo "$SLURM_SUBMIT_HOST"     # prefer this
   hostname -s                   # only if the above printed nothing
   ```

   **Prefer `SLURM_SUBMIT_HOST`; fall back to `hostname -s`. Do not "simplify" this back to a bare
   `hostname`.** Inside an allocation — an `srun --pty bash` session, or anything running under
   `sbatch` — `hostname` names the **compute node**, not the machine the user submitted from. Slurm
   sets `SLURM_SUBMIT_HOST` exactly in those contexts, and leaves it unset on a login node, so this
   one expression resolves to the submit host in both. Measured on a real cluster: on the login node
   `hostname -s` gives `login-1` and `SLURM_SUBMIT_HOST` is unset; inside `srun --pty` on the same
   cluster `hostname -s` gives `node-17` while `SLURM_SUBMIT_HOST` is still `login-1`. **The identity
   must not change just because the user is working in an interactive session** — a bare `hostname`
   would mint `alpha@node-17` for the whole session, which matches no configured `digest_cluster`,
   so the skill would stand down for that entire session and look broken. This is not a corner case:
   interactive `srun --pty` sessions are ordinary working practice.

   **Strip any domain suffix from whichever source supplied the value — take the first
   dot-separated field.** `SLURM_SUBMIT_HOST=login-1.example.edu` and `hostname -s` = `login-1` must
   both yield `login-1`. Some sites record an FQDN in `SLURM_SUBMIT_HOST` while `hostname -s`
   returns the short name; without this, the login-node identity and the in-allocation identity
   would differ by a domain suffix and the gate would stand down inside every allocation — the exact
   failure `SLURM_SUBMIT_HOST` was introduced to remove. Normalise both sources the same way, always.

   If `hostname -s` is unsupported, apply the same rule to plain `hostname`. If neither
   `SLURM_SUBMIT_HOST` nor a hostname can be read at all, the local identity is **unknown**: say so
   and stop. Do **not** fall back to the bare `ClusterName` and treat it as the identity — that is
   precisely the unsafe key this composition exists to replace. "Unknown" is never treated as a
   match.

3. **Compose the identity as `<ClusterName>@<short submit hostname>`** — e.g. `alpha@login-1`,
   `alpha@login-2`. This composed string *is* the cluster identity everywhere in this system: it is
   what `digest_cluster` holds, what goes in the log's `cluster` column, and what the local-vs-digest
   gate compares. Under the step-3 fallback it is the bare submit hostname alone.

**The host is the suffix, never the whole identity** — outside the step-3 fallback above. A hostname
on its own does not name a cluster; substituting one for the composed identity would fail the
comparison against `digest_cluster` on essentially every site and silently disable whatever it was
guarding.

### Why the identity carries a host suffix

**A Slurm `ClusterName` is not guaranteed to be unique across the clusters one person can reach.**
It is a locally chosen label, and nothing stops two independent clusters from carrying the same one.
This is not hypothetical: **two clusters reachable from one shared home directory were found
reporting the same `ClusterName` while having entirely separate controllers, separate accounting
databases, and independent job-ID spaces.** The same job number named two different jobs, submitted
a year apart, depending on which host you asked.

That matters here because `~/.claude` is often on shared/NFS storage, so **one submission log
receives rows from every cluster the user touches** — and under a bare-name key they are all tagged
identically. A scope recorded on one cluster then attaches to an unrelated job with the same number
on the other, which is exactly the corruption the `(cluster, jobid)` key was introduced to prevent.
A bare cluster name does not prevent it.

Adding a host suffix separates them, because the two clusters are reached from different submit
hosts. **At a site where `ClusterName` is already unique, this changes nothing** — the suffix just
makes the identity explicit rather than implied.

**Why the short hostname and not the controller — and what that costs.** The controller
(`SlurmctldHost`) identifies the *cluster*; the short hostname identifies *where the command was
typed*. The hostname is used because it is the name the people using this system actually say, and
the name that already appears in their existing logs, whereas a controller name is invisible to
them. That choice has a real cost, stated here rather than buried: **a cluster with several submit
hosts acquires several identities** — `alpha@login-1` and `alpha@login-2` are two identities for
one cluster.

**What happens then is benign, and visible.** The second host's identity will not equal the
configured `digest_cluster`, so the gate **stands down with a message** (`slurm-sizing` §1) instead
of silently mis-keying rows. Nothing is written under the wrong key; the run just declines to use
the table and says why. A site with several login nodes should either set `digest_cluster` per host,
or standardise on a single submit host for the jobs it wants sized. Note that a shared `~/.claude`
holds a single config, so "per host" is only available where the home directory is not shared —
otherwise, standardising on one submit host is the workable answer.

### Legacy log rows written before this change

Existing logs contain **bare** cluster names. Handle them as follows, and do not silently rewrite
them:

- A bare value that matches the `ClusterName` part of the composed identity is **unverified**: it
  may have come from any cluster carrying that name, including a different one.
- **An unverified row must not be used to satisfy a scope match.** Treat it as no match — the row
  stays `>=`. A scope that might belong to another cluster's job is worse than no scope.
- **Do not upgrade a bare name to a composed identity.** There is no evidence available after the
  fact about which cluster wrote it; writing one in would manufacture provenance.
- Say in the report how many log rows were skipped as unverified-legacy, so the user can see the
  transition happening rather than wondering why matches dropped. New rows carry the composed
  identity from now on, so this fades on its own.

## `digest_cluster` must be the exact composed identity

`digest_cluster` holds a **`<ClusterName>@<short submit hostname>`** string and is **exact-matched** —
against the local identity from the procedure above, and against the `cluster` column of every log
row. A colloquial or approximate answer ("the cluster", "our HPC", a bare hostname where a
`ClusterName` was available, a capitalised variant, or a bare `ClusterName`) is not a failure that
announces itself: it makes `slurm-sizing`'s gate never match, so the skill goes inert forever, or it
makes the digest lookup never match, so every recommendation carries `>=` forever.

Therefore **bootstrap does not ask for it in free text.** It runs the chain above, composes the
string, shows it to the user, and offers it as the answer (see Bootstrap). And the `cluster` column
written into every log row must carry that same exact composed string — it is what the digest merge
filters log rows on.

**`digest_cluster` deliberately has no default.** Guessing it is the one error that silently
corrupts results: sizing advice from the wrong cluster's data is worse than no advice, because job
IDs are not unique across clusters — and, as the finding above shows, cluster *names* are not
either. The local hostname supplies the identity's **suffix** and nothing more: it is never, on its
own, an inference about which cluster the user's digest covers. (The one exception is the step-3
fallback above, where no `ClusterName` could be read at all — and that is disclosed to the user at
bootstrap and reported on every run that uses it, not inferred silently.)

## `digest_user` and multi-user digests

Merge only rows whose `User` matches `digest_user`. Another person's peak for a same-named job
would corrupt this user's recommendation with no error anywhere. A lab-wide digest is a legitimate
input; guessing whose rows to keep is not.

Precisely what to do when other users' rows appear — **ask once, then remember**:

- **First time:** **HARD STOP** — halt before merging anything, and halt even when running
  unattended; do not filter on a default. Report which other users are present and how many
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

**Config missing:** every question in this branch is a **HARD STOP** — `digest_cluster`,
`digest_user`, and the decline option all require an answer from the user. **Running unattended is
not permission to pick one**: stop and report that bootstrap could not complete. A guessed cluster
name makes `slurm-sizing` inert or wrong forever; a guessed account name filters out every one of
the user's own rows. Neither failure announces itself.

1. **Check for a pre-plugin layout first** — two files and one **directory**, where an early
   hand-rolled version kept its data. Probe all three with `test`, which answers for a directory as
   cleanly as for a file (a `Read` cannot: it fails on a missing path and on a directory alike):

   ```bash
   test -f ~/.claude/slurm-sizing.md
   test -f ~/.claude/slurm-jobs.tsv
   test -d ~/.claude/slurm-digests
   ```

   **Three bare commands, one per path — read the exit status (`0` = present).** Do not join them
   with `&&`/`||`: a chain collapses three independent answers into a single exit status, so a
   non-zero result never tells you *which* of the three paths is missing — and `&&` short-circuits,
   so the later probes never run at all. Issue them separately and read each result.

   **On compound commands generally — the rule is about command substitution, not pipes.** A simple
   pipe whose both halves are declared is fine: `scontrol show config | grep -E '^ClusterName'` is
   covered by `Bash(scontrol *)` together with `Bash(grep *)`. What these skills must avoid is
   **command substitution** — `$(...)` cannot be checked before it runs, so a probe written that way
   may be refused outright. That is why the submit-host probe above is two bare commands rather than
   the more compact `echo "${SLURM_SUBMIT_HOST:-$(hostname -s)}"`.

   If any exist, offer to point the config at them instead of creating empty files beside them.
   Silently starting fresh next to a populated table would strand real history.
2. **Derive `digest_cluster` by running the chain and composing the identity — never ask for it in
   free text.** Run the "Determining the local cluster identity" procedure above, **compose
   `<ClusterName>@<short submit hostname>`**, display that exact composed string, and offer it as
   the answer — e.g. "This host reports `ClusterName = alpha` and submit host `login-1`, so its
   cluster identity is `alpha@login-1`; is that the cluster your weekly digest covers? [use it /
   enter a different identity / not applicable]". Show both source values, not just the result, so
   the user can tell which machine they are on — the whole point of the suffix is that two hosts
   can report the same `ClusterName`.
   - **Say which step of the chain supplied the `ClusterName`** when it was not step 1, and say
     plainly if the identity is a **host-only fallback** (step 3): "neither `scontrol` nor
     `sacctmgr` reported a `ClusterName` here, so the identity is just this submit host's name,
     `login-1` — less specific than usual". The user is deciding what to record; they need to know
     the value is degraded before they approve it.
   - **If the value came from `SLURM_SUBMIT_HOST` rather than `hostname -s`, say so** — the user is
     inside an allocation, and seeing the login node's name offered while `hostname` reports a
     compute node would otherwise look like a mistake.
   - Mention the multi-submit-host consequence when offering the value: if they submit from more
     than one login node, this identity covers **this one**, and `slurm-sizing` will stand down on
     the others (see "Why the identity carries a host suffix").
   - If the hostname cannot be read at all, say so and ask the user for the exact composed identity
     of the cluster their digest covers, warning that it is exact-matched. Do not accept or record
     a bare `ClusterName` as the identity.
   - If the user's digest covers a *different* cluster than the local one, take their answer — but
     record it verbatim, and note that `slurm-sizing` will then be inert on this machine by design.
3. **Confirm `digest_user` — show it, source it, let them correct it. Do not just write it.**
   The default is the invoking `$USER`, but this key exists *because* a Slurm account name and a
   login name can legitimately differ; a user for whom they differ gets a silently wrong value that
   then filters out **all of their own rows** — the digest merges nothing and nothing says why.
   So state the intended value, where it came from, and what it has to match — e.g. "I'll filter
   the digest to `User = <$USER>` (taken from your `$USER` login name). This must match the `User`
   column in the digest exactly. [use it / enter a different account name]". If the user is
   already holding a digest, invite them to check the `User` column against it.
4. **Offer the decline branch** in the same question as step 2 ("not applicable / I have no weekly
   digest"). Declining writes `{"enabled": false}` and stops — see "Declining".
5. Write the file with `enabled: true`, that `digest_cluster`, that `digest_user`, and the defaults
   above.

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

**Row order is part of the format: data rows are sorted by `job_name` ascending.** Every write
re-sorts. An unspecified order lets two merges of the same data produce different files, making
diffs noisy and hiding real changes during review; ascending `job_name` is stable as rows are
added.

**How to read a row**

- **`job_name`** — the Slurm `JobName` exactly as it appears in the digest. This is the lookup key;
  `slurm-sizing` matches the job it is about to submit against this column.
- **`peak_MB`** — the running maximum observed memory in **megabytes, at full precision**, across
  every digest ever merged. **This is the authoritative memory number**; everything else about
  memory is derived from it. **It never falls through merging** — a week with a smaller input
  leaves it untouched. The single exception is a human setting the `IO-BOUND` flag on in-process
  evidence (below), which may lower it: that is a deliberate correction of an inflated `sacct`
  figure, not a merge.
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

  **The scope that governs a row is the scope of the run that set `peak_MB`** — not any sibling
  run's. A job name usually has several runs in a digest and only one set the peak; the
  recommendation is derived from the peak, so the peak's provenance is what governs it. If the
  peak-setting run has no declared scope the row stays `>=`, whatever other runs of the same name
  declared, and a displaced scope is preserved in `notes` rather than discarded.

  **A recorded scope makes the row *checkable*, not automatically *actionable*.** It says what the
  measurement covered; it does not say the number is safe for the next run. Whoever uses the number
  must compare this scope against the run they are about to submit — see `slurm-sizing` §2, "The
  scope check". A `36G` row measured at "5 of 100 units" is not a `36G` recommendation for 100.
- **`last_seen`** — the week-ending date of the most recent digest that contained this job.
- **`rec_mem`** — the recommended `--mem`, computed **from `peak_MB`**. A `>=` prefix means the
  peak-setting run had no declared scope: the value may justify *raising* a request, never
  *lowering* one. **No prefix does not mean "use it unconditionally"** — it means the scope is
  known and must be checked against the intended run.
- **`rec_cpu`** — the recommended `--cpus-per-task`, capped at the job's own `ReqCPU`.
- **`notes`** — free text, **additive**: appending a note must preserve what is already there.
  `IO-BOUND` in this column is a load-bearing flag, not a comment — see below.

**The memory columns and how they change**

On every merge, for each job name:

```
peak_MB = max(existing peak_MB, this week's max UsedMem_MB)      # full precision, MB; a merge never lowers it
peak_G  = round(peak_MB / 1024, 1)                               # display only
peak_GB = peak_MB / 1024
rec_mem = round_up_to(mem_round_gb,
                      max(mem_floor_gb, peak_GB * (1 + headroom_frac)))    # headroom_frac default 0.30
```

`rec_mem` is a **pure function of the stored `peak_MB`** and the `policy` values. It is **not**
computed from this week's digest rows, and **not** from `peak_G`.

**The margin applies to a running max, not to a single observation.** `peak_MB` is the maximum
across *every* digest ever merged for that job name, so `rec_mem` is 30% above the **worst reading
ever seen** — not 30% above a typical week. That is what makes a margin this small defensible: the
number it multiplies has already absorbed every bad week in the row's history.

**`rec_mem` must never decrease while `peak_MB` is unchanged.** This is the property the whole
table exists to guarantee, and it is the reason `peak_MB` is stored at all. Without it, a job whose
peak was 14963.82 MB one week (→ `peak_G 14.6`, `rec_mem 20G`) and which runs a smaller input the
next week (5000 MB) would keep `peak_G 14.6` but recompute `rec_mem` down to `8G` — the floor —
from this week's smaller number, a recommendation less than half the row's own recorded peak and an
OOM on the next real run. Recomputing from the stored `peak_MB` makes that arithmetically
impossible.

If a legacy table has no `peak_MB` column, add it and seed it as **`(peak_G + 0.05) * 1024`**. The
`+ 0.05` is not padding, it is the rounding correction: `peak_G` is rounded to **nearest**, so a
bare `peak_G * 1024` can sit up to 51.2 MB *below* the true peak. Whether that 51.2 MB changes
`rec_mem` depends on where it falls against the `mem_round_gb` step — but when it does cross a
boundary it costs a full 4 G, in exactly the direction this column exists to prevent. Worked case:
a true peak of 9462 MB displays as `peak_G 9.2`; its true `rec_mem` is 16 G, its bare-seeded
`rec_mem` is 12 G, and the corrected seed `(9.2 + 0.05) * 1024 = 9472 MB` gives 16 G again. Seeding
at the top of the rounding interval cannot go low. Say so in the report,
and treat every such row's `rec_mem` as a lower bound until the next digest refreshes it.

**The `IO-BOUND` flag: how it is set, and what it does**

`sacct`'s `MaxRSS` on a job that reads or writes a lot of data includes kernel page cache charged
to that job's cgroup. It is not memory the job demanded, and sizing from it can overstate the real
requirement by a large multiple. `IO-BOUND` in `notes` marks a row whose numbers are hand-pinned
from a real in-process measurement instead.

*Setting it* — this cannot be done from digest data, by construction:

1. Obtain **in-process** evidence of the job's real peak RSS: `resource.getrusage(...).ru_maxrss`
   inside the job, a `psutil`-style RSS sampler, or a memory profiler. `sacct`/digest numbers are
   exactly what the flag exists to distrust, so they can never justify setting it.
2. Set the row's `peak_MB` to that measured peak, and `peak_G` to its display rounding.
3. Set `rec_mem` **by hand**, from the same in-process evidence — see the reconciliation note below.
4. **Append** `IO-BOUND` to the `notes` column, keeping any existing note text, and say where the
   measurement came from — e.g. `IO-BOUND (getrusage peak 3.2G, 2026-08-03); CPUPct 45`.

*What it does* — `slurm-digest` leaves an `IO-BOUND` row alone: its peak is not merged, its
`rec_mem`/`rec_cpu` are not recomputed, and its `notes` are not rewritten. `slurm-sizing` trusts
the pinned row over any `sacct`-derived figure for the same job.

**A pinned row's `rec_mem` does not reconcile with the formula, by design.** Every other row
satisfies `rec_mem = round_up_to(mem_round_gb, max(mem_floor_gb, peak_GB * (1 + headroom_frac)))`.
An `IO-BOUND` row does **not**: its `rec_mem` is **hand-set from in-process evidence and is
deliberately not derived from `peak_MB`**, so recomputing the formula against that row's `peak_MB`
can give a different number in either direction. **That mismatch is expected and must not be
"corrected."** It is not a data-integrity gap, and anything that "fixes" it silently throws away
the in-process measurement that is the row's entire justification. Validate a pinned row by its
recorded evidence, never by re-deriving it.

*Removing it* requires the same class of evidence that set it — never a digest, never `sacct`.
