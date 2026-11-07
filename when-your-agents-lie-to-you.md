# When your agents lie to you

### Acceptance lessons from a 4-machine, 3-runtime coding pipeline (one node since retired)

*Draft v3 — 2026-11-07. This is a field write-up, not a framework announcement.*
*v2 (2026-09-24) applied the pre-publication review (4 × P1, 8 × P2): honest implementation status for
the read-only gate, the fixture-scale lesson, terminology and structure fixes.*
*v3 adds group F — four rules the team already knew and never enforced — plus §4.5 on moving a rule from
prose into the execution path, and the spec-driven tooling comparison in §7. Items A1–E3 are unchanged.*

---

## TL;DR

- **Verify against what changed on disk, not against what the agent said it did.**
- An exit code tells you whether the process finished, not whether the work landed.
- This is a list of failure modes, not a product pitch.
- If your setup is *one machine, one coding CLI, a PR flow* — use [4x](https://github.com/ggwhite/4x) or [Liza](https://github.com/liza-mas/liza). You do not need this.
- **Every item below was a rule we already knew.** The failure was never ignorance; it was the absence of a surface that enforces the rule (§4.5).

---

## 1. The shape we're in, and why the existing projects don't fit

Our pipeline wrote code across **four machines and three different agent runtimes** — a WSL host (Windows Subsystem for Linux), two Linux servers, and a Windows box — running three separate agent frameworks, one of them a Windows-packaged build of another. (One correction since this was written: the Windows node was decommissioned. That is *not* a footnote — it is X1 happening to us. The machine that used to make Windows changes provable is gone, so the only Windows side left is the one under the WSL host, reached through the Windows/Linux boundary on a single box. A heterogeneous test surface does not stay still; plan for yours to shrink.) They are all publicly available tools, but they share no session semantics, no streaming format and no output conventions.

Four assumptions that hold in every open-source multi-agent orchestrator we looked at do **not** hold here:

| Assumption in existing tools | Our reality |
|---|---|
| Single machine | Four machines, and the coder is never on the same box as the reviewer |
| One coding CLI | Three runtimes with different session, streaming and output semantics |
| Shared git worktree | No shared filesystem, no worktrees across the boundary |
| PR flow as the review artifact | No PRs — acceptance happens on the target machine |

Those tools solve *"run several agents on one machine in parallel."* Our problem was different: **how do you know the remote box actually did the work?**

That distinction matters, because it moves the interesting engineering from *distribution* to *verification*.

## 1.5 Three extra constraints that only appear across machines

These three are specific to the multi-machine shape. Each one cost us a wasted round.

- **X1 — Heterogeneous nodes mean your test surface is incomplete.** The orchestration box has no Windows environment. Any change that touches Windows can only be proven on a different machine, so *"it passes here"* is not *"it passes."* We wrote that down as a principle; we then watched it happen — the Windows node went away, taking with it the only environment where Windows-side work could be proven. A pipeline that treats its own host as the world will ship unverified work.
- **X2 — Cross-machine channels disappear on their own.** Routes and tunnels between nodes get reclaimed between two calls. The same command succeeds, then times out, with no change on either end. Any orchestration that assumes a persistent link will produce phantom failures.
- **X3 — Path semantics differ across OS boundaries.** A path that exists on one side does not exist under the same spelling on the other. A task file with a hard-coded single-side path is an instruction that silently becomes invalid on the other side. (The encoding version of the same boundary is C5 — and it is nastier, because the bytes agree and only the *reading* differs.)

> The first real problem in cross-machine orchestration is not *how to dispatch*. It is *how to get comparable evidence back within the same run.*

---

## 2. Exit codes don't tell you whether the work got done

Three observed cases, all in the same pipeline.

**Case A — the process failed; the work is finished.** A coding step ended with `rc=1` and a log line reading `Agent run failed`. The archived round record shows the same step with four target files modified, complete, and passing the change gate. We have this more than once: the model run dies (transport, timeout, provider hiccup) *after* the edits have already been written.

**Case B — the process succeeded; nothing was executed.** A step ended with `rc=0` and its output contained *tool-call syntax as literal text* — the agent described the call instead of making it. Nothing on disk changed, and the process was perfectly healthy.

**Case C — the process failed, the work is finished, and the gate agrees — and the round is still marked failed.** *(found the day we wrote this, while drafting this section)* A coding step timed out twice: `rc=124` on both attempts. The CLI had finished the work and still did not exit; the harness killed it at the timeout cap. The step's own gate ran on the result and returned `10/10 pass` — every declared deliverable present, every assertion green. The round was still reported as failed, with the reason line reading `1 step did not pass (exit code / leak)`. The gate said pass and the accounting said fail, in the same round, about the same step.
*Cost:* a round that is substantively complete gets escalated to a human, or silently re-dispatched — the second one costs real money and re-does finished work.
*Fix:* the exit code and the gate verdict are two independent signals, and the **gate is the one that decides**. A non-zero exit alongside a passing gate is an *anomaly to record*, not a failure to declare: mark the step passed, annotate the line (`rc=124, gate=pass — non-zero exit but the delivery passed, recorded`), keep the original `rc` in the record, and add a counter the summary prints. The counter is what makes it honest: you can see how often your processes are lying about work that is already done.
Put together: **any pipeline that treats the exit code or the agent's self-report as its acceptance signal will produce both false failures and false passes.** The false failures cost money (retries that redo finished work). The false passes cost correctness (unverified code moving forward).

The rule we extracted:

> A gate must be able to answer two questions from the machine's point of view:
> **What changed on disk?** and **what did a command actually print when we ran it?**

Everything below is a way that rule gets violated — including, repeatedly, by our own tooling.

---

## 3. Twenty-three ways the verification itself lies

**A word on the title.** The twenty-three items below are your *verification* lying to you, which is the more dangerous half: a lie from an agent still leaves a disk you can inspect, whereas a lie from your own gate tells you the disk is fine. The title stays because it is what people search for; the content is about the second half.

The agent's own lie sits upstream of all of them, in §2 — Case B, where the process succeeds and nothing is executed — and D1 below is the detector we built to catch it, which then produced a false failure of its own.

Format for each: **symptom → cost → the test that catches it → the fix.**
Items marked *(our choice)* are decisions we made rather than bugs we hit; items marked *(rule)* are rules we adopted after a failure. Everything else is something we hit and reproduced.

### Group A — Gate semantics

**A1 · `skip` gets promoted to `pass` downstream.**
*Symptom:* the gate has three outcomes — pass / fail / **not-evaluated** — but the orchestrator only looked at the exit code. A step whose assertions were all not-evaluated was recorded as *gate passed*, and the run announced "all steps passed (delivery gate)".
*Cost:* "we couldn't test it" silently becomes "we tested it". This is the exact inversion the "don't block on untestable things" design was meant to avoid.
*Test:* run a step whose only assertion cannot be evaluated; check what the archived step record says.
*Fix:* carry the three states end to end — gate line → step record → run summary wording. A summary line may not contain the phrase *delivery gate passed* if any assertion was not evaluated.

**A2 · Unknown or misspelled assertion names are dropped in silence.**
*Symptom:* a typo (`min_diff_line` for `min_diff_lines`) matched no branch, so the line was skipped. No pass line, no fail line, no not-evaluated line. The gate had one fewer assertion than its author thought.
*Cost:* the gate can vanish entirely, with nothing on the judging surface to show it.
*Test:* write a typo on purpose and check that the gate goes **red**, not quiet.
*Fix:* an unrecognised directive is a hard failure. Whitespace, comments and non-directive lines are still ignored.

**A3 · `skip` triggers a retry, and stays invisible in the summary.** *(our choice)*
*Symptom:* a step with nothing to assert got re-dispatched, and the summary reported `1/1 pass` with no mention of the two assertions that were never evaluated.
*Cost:* retries with nothing to fail burn model calls, and readers over-estimate coverage.
*Test:* count the dispatches for an all-skip step (must be one), and read the summary as if you were a stranger.
*Fix:* not-evaluated assertions never trigger a retry, and every count must be visible at step level, run level and in the archive.

**A4 · A read-only gate's command screening can be bypassed.**
*Symptom:* the gate is allowed to execute commands, so it screens them against a keyword blocklist of write operations. Equivalent write commands were not on the list — and **executing them really did modify files** during review.
*Cost:* "read-only" is an intention, not an enforced property.
*Test:* take your blocklist, and try to write a file without using any of its words.
*Fix (what we actually built, not what we wish we had built):* we did two things, and only the second one is a boundary. First we widened the blocklist (`sed -i`, `perl -pi`, `rsync`, `split`, `git apply`, `dd of=`, and redirection to anything but `/dev/null`). Second — and this is the part that matters — **the gate now snapshots the declared scope before the command runs and re-checks the disk after it finishes**: a new file, a deleted file, or a changed hash fails that assertion and names the offending path. The blocklist is a cheap first filter; the disk check is what makes "nothing was written" an observable claim rather than a promise. A true allowlist with a filesystem sandbox is the stronger design and we have not built it.
*Bounded scope is part of the fix, not an optimization:* our first version of the disk check fell back to the process working directory, which on our orchestration host means the whole home tree — 191,803 files. See B3.

> **Self-check for group A:** how many outcomes does your gate have, and are all of them visible in all three places — step, run, archive?

### Group B — Your own assertions lie

**B1 · Declaration order changes semantics.**
*Symptom:* the parser is single-pass. A directive that sets context (which repository, which root) only affects the lines *after* it. Moving one line silently neutered an assertion: the assertion did not fail, it disappeared.
*Cost:* a task file looks right and checks nothing. In our case the missing baseline turned a git-based assertion into a no-op for two consecutive rounds.
*Test:* write the same assertion twice, once before and once after the context directive; the outcomes must match.
*Fix:* pre-scan the file for context directives before evaluating anything.

**B2 · A grep assertion that isn't anchored to the line it means.** *(rule)*
*Symptom:* the same keyword appears on several lines — a per-attempt detail line and the run-level summary. An assertion looking for `skip=1` matched the detail line while the real summary said `skip=2`.
*Cost:* you read "already correct" and stop looking. In our case, this false green survived a full review pass.
*Test:* make the assertion print *which line* it matched.
*Fix:* assertions must be anchored to the target line, and must report the matched content, not just success.

**B3 · Assertions that were never smoke-tested — including at the right scale.**
*Symptom (two variants, both ours):*
1. An assertion compared "lines of git diff" inside a directory that is not a git repository. The comparison is structurally unable to pass; it can only fail. It failed for two consecutive rounds on otherwise correct deliveries.
2. A fixture that is *too small* to expose the failure it is meant to catch. Our worst case: the disk-verification above was validated on a two-file repository and passed in milliseconds. On the real host, the same code path tried to walk 191,803 files (an 18 MB path list) and did not return in 90 seconds. It was caught only when a live round hung — for **two hours and forty-four minutes** — with the gate spinning at 26% CPU. The fixed version (bounded scope: declared paths only, `SCOPE_MAX_FILES` default 2000, `SCOPE_MAXDEPTH` default 4, explicit skip with a visible reason) then took 289 ms. The regression test that now guards it uses a 4,000-file directory, because the old fixture — two files — could never have failed.
*Cost:* good work is rejected, and each rejection triggers retries (see D2) — straight into the model bill. Or, worse, a check that proves nothing while your test suite stays green.
*Test:* every assertion must have been seen **both red and green** on the target machine before it is allowed into a gate. If you cannot make it go red, you cannot trust it going green. And "the target machine" means the *real* one: test at production scale, or state the scale you tested at.
*Fix:* smoke each assertion in isolation, on the machine that will run it, with fixtures sized like the real thing.

**B4 · Denominators that mean different things.**
*Symptom:* `1/1 pass` and `1 pass + 2 not-evaluated` are both rendered as "passed".
*Cost:* a percentage with an implicit denominator is not a measurement.
*Test:* count the assertions the file declares, and compare with the number the summary implies.
*Fix:* always print the denominator explicitly, and say out loud what it excludes.

> **Self-check for group B:** name every assertion currently in your gate, then say when each of them last went red — and at what scale you tested it.

### Group C — Time and content identity

**C1 · Baseline and file timestamps with mismatched precision.**
*Symptom:* "did this file change?" was implemented by comparing file mtimes (sub-second, `%.9Y`) against a run baseline taken in whole seconds. A file created in the same second as the baseline, and never touched since, compares as *newer* — a false pass.
*Cost:* unchanged artifacts sail through a change gate. We found this only because an older self-test — one we had stopped paying attention to — went permanently red.
*Test:* create a file, take the baseline a moment later, assert "must be unchanged" — it must go red.
*Fix:* both sides must use the same clock. The mirror-image bug then shows up on the other side of the same boundary: a high-precision wall-clock baseline compared against a filesystem whose mtime granularity is coarse (our volume lags the clock by 4–7 ms) will occasionally declare a file *older than the run*, producing a false failure. Measure both sides and leave an explicit tolerance; "compare two different clocks exactly" is not a specification.
*Status:* **closed** — and it took two rounds, which is itself the lesson. Sub-second baseline and strict comparison landed first; then an explicit tolerance (default 50 ms, operator-configurable, `=0` restores strict behaviour), **bounded** so a file modified before the baseline still fails, and **visible**: every time-based verdict line prints the tolerance it used and the signed delta (`require_grep` and `run_check` lines have no clock in them). A reviewer then broke the first version — the value check matched *any* line, so a tolerance of `0\n9999` slipped past the bound — the check is now whole-string. The harness pins all of it: the tolerance works, can be switched off, cannot be widened past the bound, and cannot be injected into the evidence line. The fixture that proves it is deterministic rather than probabilistic — a reviewer's ten-run stub is what found it, but a single crafted case is what keeps it fixed.

**C2 · `>` and `>=` both lie, in opposite directions.**
*Symptom:* with a whole-second baseline, "strictly newer" misses writes that land inside the same clock tick; "newer-or-equal" then accepts a file that merely shares a tick with the baseline.
*Cost:* whichever you pick, one failure mode remains; picking by intuition guarantees neither is documented.
*Test:* construct both boundary cases explicitly and keep them as regression cases.
*Fix:* fix the precision problem (C1) and then use strict inequality, because "unchanged" files are always *older*, never *equal*.

**C3 · mtime and size are not identity.**
*Symptom:* early evidence records hashed only timestamps and sizes.
*Cost:* content edited while preserving size (and often mtime) is invisible to the record.
*Test:* change content, preserve length, and re-verify.
*Fix:* hash content (sha256). Size and mtime are metadata for humans; they are not evidence.

**C4 · Tampering must be detectable, and the failure must point somewhere.**
*Symptom:* (this is the good case) a record binding the task file, the produced artifacts and the per-step conclusions with an HMAC — a keyed hash that detects changes the record was not rebuilt for.
*Test:* flip one byte in a recorded artifact, then re-verify: it must fail **and name the item** — stored hash, recomputed hash, and which entry in the record disagrees.
*Fix:* verify must recompute content, not just validate the signature over the record. A signature over stale metadata proves nothing.

**C5 · The same bytes are not the same text.** *(found the day we wrote this)*
*Symptom:* two independent platforms agreed on a file's SHA-256 and the content assertion still failed — the file was UTF-16LE, written by a Windows shell's default `Out-File`, so `grep` for an ASCII word found nothing while the hash matched byte for byte. In the same week the mirror image: the remote OS name *was in the captured output*, and our field extractor returned silence — PowerShell emitted `gb2312`, GNU `grep` hit invalid UTF-8, took its "binary file matches" path and printed no line at all. The assertion then reported "remote OS missing" on a machine whose OS was plainly visible two lines above.
*Cost:* a hash proves the *bytes* travelled intact. It says nothing about whether the two sides agree on what those bytes *mean*. Content assertions are where that shows up — and when they fail, the failure text lies about the cause ("missing", not "unreadable").
*Test:* write a text artifact from the far side using its **default** encoding, then assert on its content from the near side. Both checks must be present: hash equal **and** content matched. One passing while the other fails is the whole lesson.
*Fix:* treat "decoded to the empty string" as its own outcome — never as "absent". Force UTF-8 on the producing side where you can, mark the value `unreadable` where you cannot (our artifact path tags such values `UNREADABLE`, which is a different verdict from "missing"), and where a field cannot tolerate ambiguity, fail closed: an empty identity string aborts the check instead of passing it. Keep both sides' text semantics inside the assertion rather than in your head.

> **Self-check for group C:** what exactly do you compare when you decide "this changed", and do both sides of that comparison have identical precision?

### Group D — Detectors and orchestration

**D1 · A detector that matches bare words means quoting it is self-harm.**
*Symptom:* we detect "the agent printed a tool call instead of making one" by pattern-matching the output. The patterns were broad enough that *discussing the mechanism in prose* — as a reviewer does when explaining a finding — tripped the detector. The reviewer's report was flagged, the step was retried with a fresh session, and the run was marked failed.
*Cost:* two wasted model rounds, plus a whole run reported as failed for a correct delivery. Both times the flagged text was a *description* of the marker, not the marker in its operational form.
*Test:* keep two fixtures. One true positive (a real occurrence) must still match; one prose mention must not.
*Fix:* match the *syntax*, not the vocabulary. And never echo the literal token in task descriptions or test output — otherwise your own instructions trip your own detector.

**D2 · Retries multiply — and a retry can be structurally unable to pass.**
*Symptom:* retries exist at two levels — inside the dispatcher (rerun the agent) and above it (the gate failed, re-dispatch the step). Uncapped: *2 × 2 = 4* model attempts for one step. After capping the gate-level re-dispatch: *2 + 1 = 3*.
*Second failure mode:* a re-dispatch reset the "changed since" baseline, so every assertion meaning *"this file must have been modified during this run"* became unsatisfiable on the retry. A step whose work was already finished got retried, then failed — the retry was structurally incapable of passing. In our case the step's code was already complete and had been verified independently (38/38 checks); the round still burned two more model attempts and then reported failure.
*Cost:* a single flaky step can silently consume several times its budget; and a finished step can be failed forever by its own retry.
*Test:* write the worst case as one sentence, then assert it in a test. If you cannot state it, it is not bounded.
*Fix:* cap explicitly at one level, document the arithmetic in the code comment — ours said "2" while the code produced 4 — and evaluate "what changed" against the **run's** start, not the latest attempt's. We verified the fix with a before/after control: against the pre-fix backup, a stub round still fails on attempt 2; against the fixed orchestrator, attempt 2 passes.

**D3 · Editing an orchestration script while it is running.**
*Symptom:* bash reads a script incrementally. Editing `orchestrator.sh` mid-run shifted its read offsets and **the entire trailing block — the attestation step — was swallowed**. The run finished "successfully" with no evidence record at all. Re-running the same task with the same parameters by hand produced a normal record.
*Second occurrence:* a later round died with `unexpected EOF while looking for matching quote` at a line number that no longer existed — the coder had edited the orchestrator *in place*, and the running interpreter read a truncated intermediate state. By the time we looked, the file on disk was perfectly valid, which is exactly what makes this one confusing.
*Cost:* the artifact that proves what happened is the thing that silently disappears.
*Test:* check that the last thing a run prints is actually the last thing the script contains.
*Fix:* move run teardown into a separate script that the orchestrator merely invokes, and always edit the orchestrator last. When a live script really must change, **replace it atomically** — write the new version elsewhere and `mv` it over the old path: the running interpreter keeps reading the old inode, so nothing breaks. In-place editing of a script a running round depends on is not a fast lane, it is a corruption.

> **Self-check for group D:** can your detector distinguish *a real occurrence*, *a discussion of it*, and *a test of it*?

### Group E — Attestation, isolation and observability

**E1 · Attestation scope is a source of both over- and under-collection.**
*Symptom:* one record pulled in dozens of files that had nothing to do with the change, while a *relatively* referenced path was dropped from the record — and the record still verified as passed.
*Cost:* noise, plus the risk that the one file that mattered is not covered.
*Test:* declare the scope, then assert that (a) nothing outside it is collected and (b) anything missing inside it fails.
*Fix:* the record must state what it covers, and coverage gaps must be failures, not omissions.

**E2 · Sandboxes and evidence logs get polluted accidentally.**
*Symptom:* while testing, records were written into the **real** log store — a few dozen lines appended to the live evidence stream, and in one case a sandbox attestation landed in the production store and had to be surgically removed.
*Cost:* your evidence trail becomes untrustworthy exactly when you need it to prove something.
*Test:* re-run your test suite after pointing the output root at a temporary directory. Then check whether the real store grew anyway.
*Fix:* every output path must be overridable by an environment variable, and the default must be the sandbox when running under a test harness.

**E3 · No incremental observation.**
*Symptom:* conclusions existed only in the run's stdout and in a single result file written at the end. If a run dies before teardown, everything is gone — which is exactly what happened in D3.
*Cost:* an interrupted run leaves no usable forensics.
*Test:* kill a run mid-flight and see what you can still read.
*Fix:* append one structured line per judgement as it happens; write the run-level result file as a summary, not as the only copy.

> **Self-check for group E:** after redirecting your output root, does anything still write to the real store?

### Group F — Rules that were never enforced

Every item below was a rule someone on the team already knew. The failure was not ignorance; it was the
absence of a surface that enforces the rule. That distinction is the whole group.

**F1 · A rule the agent knows is not a rule the agent follows.**
*Symptom:* "never print a secret" existed in prose, in the system prompt and in the task template. It
still failed **seven times in one engagement** — each time through a *different* command nobody had
enumerated: `diff` on a config file, `cat -n`, `od`, `head` of an `.env`, `git show` of a commit that
contained a token, and an inline `mysql -p…` one-liner. None of these were leaks in transit. The secret
was already on disk; the tool call copied it into the transcript.
*Cost:* the transcript is the leak surface — logs, the model's context window, screenshots, and any
downstream summariser that reads them. One `diff` of a config file can put six credentials somewhere you
never audited.
*Test:* take every command in your toolchain that can print a file and run it against a canary file
containing a fake secret. If the secret appears in the tool result, that command is a hole — regardless
of what the rule says.
*Fix (two mechanisms, neither of which is a prompt):* (a) **pre-execution interception** — refuse the
command *shape* before it runs, so an enumeration of dangerous commands is enforced rather than
remembered; (b) **post-execution transformation** — rewrite every tool result so a secret becomes
`key=<name> len=<n> shape=<class>`, keeping no first or last characters and no hash (a hash is still an
oracle). We shipped (b) on the tool boundary, not as advice, and the rule now holds for an agent that
has never heard of it (it is a small MIT plugin, [credential-guard](https://github.com/pzmmy/credential-guard)). The honest gap: an allowlist plus a filesystem sandbox is the stronger design; we
have the two mechanisms above, not the sandbox.
*Why it belongs here:* it is A4 in a different costume — "read-only" and "never print secrets" are both
*intentions* until something on the execution path makes them properties.

**F2 · Spec, plan and tasks drift apart while each one stays internally consistent.**
*Symptom:* after a few rounds, `spec.md` described a behaviour the `plan.md` no longer implemented, and
`tasks.md` still scheduled work for the behaviour that had been removed. Every document was readable, and
every document was self-consistent. The *set* was not. Nothing in the gate looked at the set.
*Cost:* you implement the plan, verify against the tasks, and ship something the specification does not
describe — with every step green, because no check compares documents to each other.
*Test:* pick one requirement and trace it forward (spec → plan → task → test), then pick one task and
trace it backward. Every dangling edge is a finding, and the **count of dangling edges** is the
measurement.
*Fix:* a mechanical cross-artifact pass — build a semantic model of each document, run four detection
channels (duplication, contradiction, coverage gap, terminology drift), and grade each finding
CRITICAL / HIGH / MEDIUM / LOW. Report the edges, not a summary sentence. A consistency check that only
prints "looks fine" is the same false green as B2.

**F3 · "Done" is an opinion until it is computed.**
*Symptom:* after implementation, "are we done?" was answered by reading the diff and feeling satisfied.
Two of three rounds called finished still had declared intent with no code behind it. The missing part
was never the hard part; it was the part nobody had written down as an item.
*Cost:* the last ten percent lives in the gap between *what the artefacts asked for* and *what the code
does*. A human reading a diff cannot enumerate that gap, and neither can an agent asked to "check your
work" — the same actor that skipped the item is the one auditing for it.
*Test:* write the **intent inventory before looking at the code** — every declared behaviour, constraint
and acceptance criterion, one line each — then mark each item `present / partial / absent` by pointing
at the code that satisfies it. If you cannot point, it is not present.
*Fix:* compute the verdict. Build the inventory from the artefacts, assess the codebase against it,
classify the findings, then either report `Converged` with an empty remainder, or append the remainder as
tasks. **A round may not be called complete while the remainder is non-empty.** This is A1's rule —
three states, not two — applied to the project instead of to a step.

**F4 · Three different things called "memory".**
*Symptom:* "you told me X last month" was sometimes a full-text hit in the session archive, sometimes a
derived conclusion from a memory service, and sometimes a line injected into every prompt. All three were
called *memory*, and all three could be wrong in different ways: the archive hit can be a different
project that used the same word; the derived conclusion can be a stale inference that was never revised;
the injected profile can be a fact that stopped being true. In one review all three were cited in the
same paragraph as if they were one source.
*Cost:* you cannot audit a claim you cannot attribute. A wrong memory is worse than no memory, because it
arrives with the confidence of a fact.
*Test:* take one recalled fact and answer three questions — which store produced it, when was it written,
and what would have to change on disk for it to become false?
*Fix:* name the layers and give them an authority order — injected profile (highest authority, smallest
scope), derived memory (medium; must carry provenance and a timestamp), history search (lowest: it is a
*quote*, not a conclusion). Never let a quote be presented as a conclusion, and never let a derived
inference be presented as something the user said.

> **Self-check for group F:** for each rule you believe your pipeline enforces, name the surface that
> enforces it — then try to break the rule without touching that surface.


---

## 4. The architecture we ended up with

Five layers, each one a direct answer to the failure modes above. Nothing here is exotic; the point is that each layer is *mechanically checkable*.

1. **Evidence-based gate** — an assertion may demand *output*, not promises. The gate re-runs the command itself, stores the output, fails if the evidence is missing, and re-checks the disk afterwards to catch writes the command screening missed. *Catches: A4, B2, B3.*
2. **Three-state propagation** — pass / fail / not-evaluated travels unchanged from the gate to the step record, the run summary and the archive. *Catches: A1, A2, A3, B4.*
3. **Content-fingerprint attestation** — every run binds the task file, the produced artifacts and the per-step conclusions into one verifiable record. *Catches: C3, C4, E1.*
4. **Incremental observation** — one structured result per run plus an append-only stream of individual judgements, so a crash still leaves a trail. *Catches: E2, E3, D3.*
5. **Explicit cost ceiling** — the worst-case number of model attempts per step is *written down in one sentence*, and the code comment matches the code. *Catches: D2.*

The one-line version: **every layer converts a claim into an observation.**

## 4.5 From lessons to mechanisms: three executable gates

The five layers above **accept** work. They do not **prevent** it; they judge it after the fact. The four
failures in group F sit earlier in the path, and the fix for all of them is one move: take the rule out of
prose and put it on the execution path.

A rule can live at three levels, and only one of them survives an agent that is tired, rushed, or working
from a stale instruction:

| Level | Where the rule lives | What enforces it | Failure mode |
|---|---|---|---|
| Prose | a document, a README, a checklist | human memory and attention | forgotten under load |
| Prompt | the system prompt, a task template | the model's attention | diluted by context, overridden by a stronger instruction |
| **Mechanism** | the execution path itself | the runtime | costs engineering — and cannot be talked out of |

Three gates, one per enforcement failure:

1. **Credential gate — before and after every tool call.** Before: refuse the command shape (F1a). After:
   transform the result (F1b). Neither asks the agent for cooperation. The check to keep: a canary secret
   that must never appear in any tool result.
2. **Consistency gate — between artefacts, before implementation.** Model each document, run the four
   detection channels, grade the findings (F2). The check to keep: every requirement traces forward and
   every task traces back.
3. **Convergence gate — against the artefacts, after implementation.** Build the intent inventory,
   assess, and produce either `Converged` or a remainder that becomes tasks (F3). The check to keep: a
   round cannot be marked complete with a non-empty remainder.

The pattern is identical in all three: **the rule stops being something the agent must remember and
becomes something the system does.** That is also why they are cheap — each is a few hundred lines. The
expensive part was noticing that the rule had been "known" for months while the failures kept happening.

> **The generalisation:** if a rule has failed more than twice, stop rewriting the rule and start building
> the surface that enforces it.


---

## 5. Migration checklist

If you already have a pipeline, add these in order. Each row gives you a precondition, a machine-checkable criterion, an observable failure signal, and a fallback if you cannot build the full thing.

| Step | Precondition | Machine-checkable criterion | Observable failure signal | Fallback / rollback |
|---|---|---|---|---|
| ① Evidence-based gate | The gate can run commands on the target machine | Every assertion has been seen both red and green there | Deliverables exist, no command output anywhere | Degrade to "file set + content sha256 comparison" |
| ② Three-state propagation | The gate distinguishes pass / fail / not-evaluated | The not-evaluated count is identical at step, run and archive level | Summary says "all passed" while not-evaluated assertions exist | At minimum, persist the not-evaluated count in the archive |
| ③ Content fingerprint | You can hash content | Flipping one byte makes verification fail and names the item | A record exists but verification never recomputes content | Degrade to "hash manifest + manual recomputation" |
| ④ Incremental observation | Output root is overridable | After a mid-run kill, per-judgement conclusions are still readable | An interrupted run leaves only a single "failed" line | Degrade to "one result file per run" |
| ⑤ Cost ceiling | You know the retry hierarchy | Worst-case attempt count can be stated in one sentence | One delivery consumes several times its expected model calls | Turn off all inner retries; keep only gate-level re-dispatch |

### A minimal `expect` skeleton

The smallest version of ①–③ that we would hand to someone starting today — six lines, no framework:

```yaml
expect: |
  require_changed: <path>                     # the fix actually touched this
  require_grep: <pattern> in <path>           # it really contains the new branch
  run_check: <your real test command>         # the gate re-runs it and stores the output
  require_evidence: <output file> cmd=<token> # the claimed run produced this token
  require_nonempty: <output file>             # ...and it is not empty
  require_changed: <path that must NOT change> # the negative case: this must go red
```

The last line is deliberate: an assertion set with no red case is not a test.

### Three knobs to build in from day one

Every one of these exists in ours because its absence cost us a round:

- `RUN_CHECK_TIMEOUT` — wall-clock cap per gate command, so one slow check cannot hold a round open. Default 120 s.
- `SCOPE_MAX_FILES` / `SCOPE_MAXDEPTH` — hard bounds on any disk scan, with a *visible* "skipped, scope too large" line instead of a silent all-tree walk. Defaults 2000 / 4.
- `PIPELINE_SANDBOX_ROOT` — one variable that redirects every output path (logs, evidence, attestations) into a scratch tree, so tests cannot pollute the real record.

To run ours: the orchestrator takes a task file (`orchestrator.sh tasks/<name>.yaml`), each step declares an agent and an `expect` block, and the round ends by writing an HMAC-bound attestation plus one result file. The scripts are ~1,500 lines of bash in total, which is the honest size of this problem: small enough to read, boring enough to trust.

---

## 6. Anti-patterns

- **Replacing the framework before naming the failure.** Our weaknesses were not architectural. They were things like "a gate that says pass when it means not-evaluated." A new framework would have reproduced them.
- **Unbounded retries.** Retries multiply across levels (D2) and hide the fact that a step is structurally unable to pass (B3).
- **Letting the producer judge its own output.** The same actor that wrote the change is the worst-placed actor to certify it.
- **Gates written as prompts.** A gate that lives in a prompt is a suggestion. A gate that runs a command and reads the exit code and output is a gate.
- **Writing the rule again, louder.** If a rule has failed twice, the problem is the enforcement surface, not the wording. Repeating it in the prompt is how you get the seventh credential leak.
- **Declaring done from the diff.** The diff shows what changed, not what the specification asked for. Only an inventory can answer "is it all there".

---

## 7. Relationship to existing projects

We surveyed the field before building. Each of these does something better than we did, and each of them assumes a shape we do not have:

- **[4x](https://github.com/ggwhite/4x)** — deterministic gates enforced by a CLI rather than by prompts; the tester must hand over a `verify.json` containing command output. Closest to our ①.
- **[Liza](https://github.com/liza-mas/liza)** — per-step doer/reviewer binding with mechanically enforced role boundaries and behavioural contracts.
- **[Metaswarm](https://github.com/dsifry/metaswarm)** — the orchestrator verifies independently and never trusts a subagent's self-report; retries get a *fresh* reviewer.
- **[Multi-Agent Evolution Framework](https://github.com/CJ-coding-apps/Multi-Agent-Evolution-Framework)** — signed in-toto attestations per role, plus per-role tool allowlists. Closest to our ③.
- **[Emdash](https://github.com/generalaction/emdash)** — natively targets remote machines; the only one pointing the same direction as us on cross-machine work.
- **[spec-kit](https://github.com/github/spec-kit)** — artefact templates plus clarify / checklist / analyze gates and a converge loop; the closest public work to our F2 and F3, and the first mainstream toolchain to make the artefacts first-class. Two limits matter for this write-up: it assumes a single repository where the agent implements, and it has no notion of acceptance on a different machine. Its gates are also **skills the agent runs**, so by §4.5's ladder they are still prompts — better prompts, but not yet a surface.
- **[A2A](https://a2a-protocol.org/)** — the emerging standard for agent-to-agent interop across hosts (Linux Foundation; v1.0.1, current as of May 2026). **We read the spec rather than implementing it, and "an Agent Card plus JSON-RPC per node" understates it in the direction that matters**: a conformant node is an Agent Card's required field groups, a resident HTTPS JSON-RPC service serving eleven methods, a task store with cursor pagination, and per-request authorization scoping. What it does *not* give you is anything our pipeline is actually built around — assertion-level gates, three-state verdicts, content hashes, signed attestations: the A2A data model has no verdict object, and `Artifact` has no digest field. For four statically known nodes behind one orchestrator, A2A's discovery layer buys nothing (its own spec points you at direct configuration), and its SSRF guidance steers push notifications away from all four private ranges (`127.0.0.0/8`, `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`) — and ours is one of them. **Adopting it would be additive, not substitutive: every script in our acceptance layer would stay.**

What we added is narrow and specific: **cross-machine, heterogeneous runtimes, no PR flow, acceptance on the machine that ran the change.**

Three caveats on our own evidence, since this whole piece is about evidence. First, all of it comes from **one pipeline** over a few months — twenty-three failure modes from one system is a sample, not a survey, and a different stack will meet a different twenty-three. Second, the attestation layer detects **tampering and corruption, not a malicious operator**: the key lives on the orchestration host, so anyone who can read the key can forge a record. It is an integrity check for a cooperative team, not a defence against an adversary who already has your shell.

Third — and this is the one we had to learn by building it — **verification only means something if the verifier's own working set is out of the verified party's write scope.** We wrote a cross-machine evidence tool whose whole promise was "never a silent pass". A reviewer took it apart three times, and each time the false pass came from the same direction: not a broken comparison, but the *inputs* to that comparison being writable. The tool printed `missing: 2` and still concluded PASS, because the summary counters were recomputed from an on-disk file that another process had overwritten. Isolate the run directory, read the inputs once, and keep the verdict in memory — and then say out loud what you are *not* defending against. If the other side can rewrite your inputs, it can rewrite your gate script too, and no local file check survives that. State the boundary instead of pretending the tool covers it.

---

## Appendix A — the twenty-three, in one table

| # | Pitfall | One-line test | One-line fix |
|---|---|---|---|
| A1 | `skip` promoted to `pass` | Archive a step with only not-evaluated assertions | Carry three states end to end |
| A2 | Unknown assertion names dropped silently | Introduce a typo; the gate must go red | Unrecognised directive = failure |
| A3 | `skip` retries and hides in the summary | Count dispatches for an all-skip step | No retry on not-evaluated; print every count |
| A4 | Read-only screening is bypassable | Write a file without using any blocked word | Blocklist + **post-command disk check**; allowlist + sandbox is the stronger design |
| B1 | Declaration order changes meaning | Same assertion before/after the context line | Pre-scan context before evaluating |
| B2 | Grep assertion not anchored | Make it print which line it matched | Anchor to the target line |
| B3 | Assertion never smoke-tested — or tested too small | Make it go red on purpose; test at production scale | Red-and-green before it ships |
| B4 | Denominators that disagree | Count declared assertions vs implied | Print the denominator and its exclusions |
| C1 | Baseline/mtime precision mismatch | Baseline after creating the file; must fail | Same clock on both sides, plus a bounded, configurable, visible tolerance (whole-string validated) |
| C2 | `>` vs `>=` | Build both boundary cases | Fix precision, then use strict `>` |
| C3 | mtime/size as identity | Change content, keep length | Hash content |
| C4 | Tampering undetected | Flip one byte, re-verify | Recompute content; name the mismatch |
| C5 | Bytes identical, text not | Write a text artifact with the far side's default encoding; assert content from the near side | Hash *and* content; treat "decoded empty" as its own outcome, never as "absent" |
| D1 | Detector matches bare words | Keep a prose fixture and a real fixture | Match syntax; never echo the token |
| D2 | Retries multiply; retry cannot pass | State the worst case in one sentence | Cap one level; baseline against the run, not the attempt |
| D3 | Editing a running script | Check the run's last output line | Externalise teardown; replace live scripts atomically |
| E1 | Attestation scope too wide / too narrow | Declare scope; assert both directions | Coverage gaps must fail |
| E2 | Sandbox writes to the real store | Redirect output root, then re-check | Every path overridable; sandbox by default |
| E3 | No incremental observation | Kill a run mid-flight | Append per judgement; summarise at the end |
| F1 | A known rule, never enforced | Run every file-printing command against a canary secret | Intercept before; transform after — not a prompt |
| F2 | Artefacts drift while each stays consistent | Trace one requirement forward and one task back | Cross-artefact pass with graded findings |
| F3 | "Done" is an opinion | Write the intent inventory before reading the code | Compute `Converged`, or append the remainder |
| F4 | Three things called "memory" | Attribute one recalled fact to a store and a date | Name the layers; give them an authority order |

## Appendix B — the three cross-machine constraints

| # | Constraint | Observable symptom |
|---|---|---|
| X1 | Heterogeneous nodes → incomplete test surface | "Passes here" ≠ "passes", because one OS is missing from the orchestrator's host — and the missing OS can disappear entirely (our Windows node was retired) |
| X2 | Cross-machine channels vanish | The same command succeeds, then times out, with no change on either side |
| X3 | Path semantics differ across OS | A task file with a single-side path is silently invalid on the other side |

---

## License

Text: CC BY 4.0. Any scripts extracted later: MIT.

*Corrections welcome — every one of the twenty-three above was found the expensive way, and we assume there are more.*
