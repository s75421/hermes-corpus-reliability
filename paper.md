# From “他 → he” to End-to-End Trustworthy Ingestion

## Reliability Engineering of a Local LLM Agent–Corpus Pipeline

**從「他 → he」到端到端可信入庫：本地 LLM Agent–Corpus Pipeline 的可靠性工程實證**

---

## Abstract

A language-model agent can eventually produce the correct result while still being unreliable at almost every layer between user input and final side effects.

This paper presents a production engineering case study of a fully local LLM Agent–Corpus ingestion pipeline built around Hermes Agent, a local Qwen 27B model, Ollama, SQLite durable state, a persisted work queue, and Hermes native Cron scheduling on a Windows workstation.

The investigation began with a seemingly minor data-integrity failure: a Chinese character in user-pasted source material was transformed from `他` into `" he"` before formal ingestion. That defect exposed a broader class of reliability problems involving input fidelity, identity propagation, tool termination, scheduler ownership, resource admission, retry semantics, durable state, and test-harness correctness.

Rather than attempting to improve reliability through larger models, additional retries, or a second queueing system, the pipeline was redesigned around explicit invariants:

- original user bytes are authoritative,
- durable state is the correctness boundary,
- wake events are latency hints rather than sources of truth,
- side effects are fenced by ownership,
- retries are bounded,
- foreground and background responsibilities are separated,
- failures and negative evidence are preserved.

Production end-to-end measurements showed that the delay between a resource-deferred native execution and its next execution was reduced from **155.273 seconds to 0.328 seconds**, a reduction of approximately **99.79%** and an improvement of approximately **473×**.

The complete background campaign duration was reduced from **199.199 seconds to 42.213 seconds**, approximately **4.72× faster**, while preserving the final correctness outcome:

```text
DELIVERABLE: 1
UNCERTAIN:   0
REJECTED:    0
FAILED:      0
```

A key finding is that the local LLM itself was not the dominant bottleneck. Actual artifact-review inference took approximately **5–7 seconds**. Most of the original latency arose from orchestration.

---

# 1. Introduction

LLM-agent evaluation is often reduced to a simple question:

> Did the agent eventually complete the task?

For production systems, this definition is insufficient.

An agent may eventually return the expected result while simultaneously:

- corrupting user-supplied data,
- executing unnecessary tools,
- duplicating side effects,
- retrying deterministic failures,
- waiting minutes for scheduler alignment,
- losing ownership of asynchronous work,
- misclassifying normal operational outcomes as failures,
- or producing misleading test evidence.

The system described in this paper exhibited several of these failure modes despite eventually completing many requests successfully.

The engineering goal therefore changed from:

> Make the Agent complete the task.

to:

> Make every state transition between user input and canonical output explainable, durable, bounded, and testable.

This case study documents that transition.

---

# 2. Research Questions

The investigation focused on five practical questions.

### RQ1 — Input Fidelity

How can a formal ingestion pipeline guarantee that user-pasted source material remains identical to the original user turn rather than a model-reconstructed approximation?

### RQ2 — Ownership

When a foreground agent hands a task to a background workflow, how should ownership transfer be represented so the foreground agent stops working?

### RQ3 — Scheduling

How can a durable recurring scheduler provide low-latency execution without replacing durable state with unreliable wake events?

### RQ4 — Resource Recovery

How should temporary local-model resource contention be retried without introducing an unbounded loop or another queueing system?

### RQ5 — Evidence

How can production regressions be distinguished from pre-existing flaky tests, timing artifacts, and failures in the test harness itself?

---

# 3. System Environment

The production system used the following environment.

| Component | Configuration |
|---|---|
| Operating system | Windows 10 Pro for Workstations 22H2 |
| CPU | AMD Ryzen 9 9950X, 16 cores / 32 threads |
| RAM | 192 GB DDR5 |
| GPU | NVIDIA RTX 5090, 32 GB VRAM |
| Local model | Qwen 27.3B, Q6_K |
| Model context | 131,072 tokens |
| Runtime | Ollama |
| Ollama version | 0.32.15 |
| Hermes Agent | 0.20.5 |
| Python | 3.11.16 |
| SQLite runtime | 3.53.1 |
| SQLite mode | WAL |
| SQLite synchronous | FULL |
| Corpus durable state | SQLite |
| Work persistence | Existing persisted queue |
| Scheduler | Hermes native Cron |
| Scheduler ticker | 60 seconds |
| Campaign schedule | `every 2m` |
| Artifact reviewer | Local-only |
| Reviewer temperature | 0 |
| Host storage | NVMe system disk + SATA Corpus disk |

No Redis, Celery, RabbitMQ, or second review queue was introduced.

This was intentional.

The existing system already contained enough durable primitives. The objective was to repair ownership and scheduling semantics rather than add another infrastructure layer.

---

# 4. Failure 1: “他” Became `" he"`

The reliability investigation began with a small textual mutation.

A user pasted source document contained:

```text
他
```

During the front-agent path, one instance was transformed into:

```text
" he"
```

The resulting artifact remained readable and semantically recognizable.

That made the defect more dangerous, not less.

A semantic evaluator could easily accept both texts as “equivalent,” but a formal corpus-ingestion system must distinguish between semantic similarity and data identity.

## 4.1 New Invariant

For formal user-paste ingestion, the source of truth became the original user turn.

The accepted artifact must be an exact contiguous substring of that original input.

Allowed normalization was intentionally limited to transport-level differences such as:

- CRLF versus LF,
- final writer newline.

Explicitly forbidden transformations included:

- fuzzy matching,
- semantic reconstruction,
- paraphrasing,
- Unicode NFKC normalization,
- whitespace collapsing,
- LLM re-copying.

The principle was simple:

> A model may analyze canonical user data, but it may not silently recreate it.

## 4.2 Identity Propagation

A native turn identity was propagated using session and turn identity.

The relevant logical boundary became:

```text
(session_id, turn_id)
```

rather than relying on the model's rewritten tool arguments.

A write-once raw-turn cache was used as the canonical reference for the formal ingestion gate.

---

# 5. Failure 2: Deterministic Errors Triggered Tool Loops

Another early failure involved deterministic ingestion errors.

Instead of terminating after receiving an exact error code, the foreground agent repeatedly:

- inspected source files,
- retried tools,
- searched state,
- and consumed the tool-iteration budget.

In one observed class of failure, this behavior reached approximately forty tool iterations.

The problem was not that the error was difficult.

The problem was that the system did not classify it as terminal.

## 5.1 Fix

Deterministic `USER_PASTE_*` failures were converted into terminal foreground outcomes.

The final response path explicitly set:

```text
tool_choice = none
tools = []
```

The exact error code was preserved.

This prevented the model from deciding whether a deterministic transport or validation error “felt retryable.”

---

# 6. Failure 3: Successful Ingest Did Not End the Foreground Turn

After a formal ingest was successfully accepted by the background system, the foreground agent continued operating.

Typical unnecessary actions included:

```text
SQLite inspection
staging inspection
event inspection
batch inspection
grep / uncertain checks
file re-reading
```

The background workflow already owned the task, but the foreground model had no explicit terminal state telling it to stop.

## 6.1 Structured Handoff

A successful background handoff was represented structurally.

The relevant state included:

```text
background_accepted = true
scheduler_health = healthy
completion_notification = scheduled_to_origin
foreground_polling_required = false
next_action = REPLY_SUBMITTED_AND_END_TURN
```

Once these conditions were met, tools were disabled and the turn ended.

The front-end response became a short acknowledgement similar to:

> Submitted. Waiting for Corpus completion notification.

Production testing confirmed that the previous post-ingest SQLite/staging exploration stopped.

---

# 7. Failure 4: The First Native Execution Took ~155 Seconds to Start

The first major latency investigation initially appeared to implicate storage, SQLite, or the local model.

Precise timestamps showed:

```text
job created   06:51:49
review start  06:54:24
```

The delay was approximately:

```text
155 seconds
```

Source inspection revealed the actual cause.

The campaign owner used:

```text
every 2m
```

while the built-in scheduler scanned approximately every:

```text
60 seconds
```

Depending on phase alignment, a newly created campaign could therefore wait roughly:

```text
120–180 seconds
```

before the first useful native opportunity.

The measured ~155 seconds fit this model.

This was the first important performance lesson:

> Do not call a system “model-bound” until scheduler latency has been separated from inference latency.

---

# 8. Durable Re-arm

The first scheduling optimization did not shorten the global Cron interval.

Instead, a fresh campaign used the scheduler's existing durable job record.

The existing trigger primitive set:

```text
next_run_at = NOW
```

This changed the job's durable schedule state before any wake notification was attempted.

The architecture therefore became:

```text
durable next_run_at
        ↓
correctness authority

wake notification
        ↓
latency optimization
```

If the wake were lost, the normal ticker would still eventually see a due job.

If the wake were duplicated, the durable scheduler and fire-claim logic remained authoritative.

---

# 9. Scheduler Wake

The native scheduler originally waited using a fixed timer.

A generation-based wake mechanism was added using a condition variable.

Healthy waits could be interrupted by job mutations.

Failure backoff remained unchanged.

The design intentionally avoided making the wake itself authoritative.

## 9.1 Production Result

After durable re-arm and scheduler wake, production measurements showed:

```text
campaign → first native
```

in approximately:

```text
2.03–2.72 seconds
```

instead of approximately:

```text
155 seconds
```

The first major scheduler bottleneck had been removed.

---

# 10. A Second Three-Minute Delay Appeared

Fixing the first scheduler delay exposed another one.

A production trace showed:

```text
campaign created       13:02:08.269
first native start     13:02:10.299
first native finish    13:02:35.095
second native start    13:05:10.368
```

The first wake was excellent:

```text
campaign → first native = 2.030 s
```

But the gap after the first execution was:

```text
first native finish
→ second native start
= 155.273 s
```

The total background campaign lasted:

```text
199.199 s
```

The first scheduler bug was fixed, but the workflow still returned to the normal recurring cadence after a resource defer.

---

# 11. Resource Admission and HEAVY_WARMUP

The first native execution had not failed.

It had been deferred by resource admission.

Observed values included approximately:

```text
native runtime               24.797 s
post-consumer review         22.813 s
status                       resource_deferred
reason                       HEAVY_WARMUP
```

The local QoS system required a sustained clean period before admitting heavy same-resident inference.

The same-resident path required approximately:

```text
18 seconds
```

of continuous clean state.

The bounded recovery window was approximately:

```text
21 seconds
```

If the first few seconds were still considered active or not clean, the remaining clean interval could fail to reach eighteen seconds before the recovery deadline.

The first invocation therefore behaved approximately like:

```text
native starts
    ↓
resource gate
    ↓
wait up to ~21 s
    ↓
HEAVY_WARMUP still not mature
    ↓
resource_deferred
```

The resource gate itself was working.

The problem was what happened next.

---

# 12. Why a Direct `trigger_job()` Was Unsafe

A tempting fix was to call the scheduler trigger directly from the running Corpus job.

Source inspection showed this would introduce a race.

Hermes Cron performed approximately the following sequence:

```text
tick()
    ↓
advance_next_runs()
    ↓
execute Corpus script
    ↓
mark_job_run()
    ↓
compute_next_run()
    ↓
save
```

If the child workload called:

```text
trigger_job()
```

during execution, it could temporarily write:

```text
next_run_at = NOW
```

but the terminal scheduler path would later execute:

```text
compute_next_run()
```

and overwrite that value.

The child workload did not own terminal scheduling state.

This became a central ownership rule:

> A workload may request a scheduling outcome, but only the scheduler's authoritative terminal path should commit it.

---

# 13. D5: Durable Post-Run Retry Intent

The final design introduced a durable retry **intent**, not another queue.

When bounded local review ended with one of the explicitly recoverable reasons:

```text
HEAVY_WARMUP
RESIDENT_RESOURCE_CONTENTION
```

Corpus stored a request associated with the current fire owner.

Conceptually:

```text
post_run_retry = {
    reason,
    requested_at,
    fire_owner
}
```

The workload did not modify `next_run_at`.

## 13.1 Terminal Consumption

The authoritative terminal scheduler path later:

1. reacquired the job under the fire-job fence,
2. verified the expected fire owner,
3. updated terminal run bookkeeping,
4. cleared the fire claim,
5. computed the normal recurring next run,
6. consumed the pending retry intent,
7. replaced `next_run_at` with `NOW`,
8. persisted the record,
9. notified the scheduler provider.

The wake remained latency-only.

The durable `next_run_at` remained the authority.

---

# 14. Bounded Retry Budget

A critical design constraint was preventing a hot loop.

The policy allowed only:

```text
one immediate post-run retry
```

A successful terminal arm incremented:

```text
post_run_retry_streak = 1
```

If the second native execution also requested another immediate retry, that request was denied.

The workflow then returned to its original recurring schedule.

The resulting policy was:

```text
bounded recovery
        ↓
one immediate fast retry
        ↓
normal recurring fallback
```

not:

```text
while deferred:
    retry()
```

This preserved a hard upper bound on retry amplification.

---

# 15. Production D6 End-to-End Result

The final production experiment produced the following trace.

```text
campaign created
14:02:25.646

first native start
14:02:28.229
```

First native execution:

```text
native runtime                    24.125 s
post-consumer review              21.922 s
status                            resource_deferred
reason                            HEAVY_WARMUP

post_run_retry.requested          true
mode                              immediate_after_terminal_mark
```

It finished at:

```text
14:02:52.355
```

The second native execution began at:

```text
14:02:52.683
```

Therefore:

```text
first native finish
→ second native start
= 0.328 s
```

The second invocation successfully entered review.

```text
review start    14:02:56.544
review finish   14:03:02.375
review runtime  5.831 s
```

The campaign stopped at:

```text
14:03:07.859
```

Total background campaign time:

```text
42.213 seconds
```

---

# 16. Quantitative Results

## 16.1 Post-Defer Continuation

Before D5:

```text
155.273 s
```

After D5:

```text
0.328 s
```

Improvement factor:

```text
≈ 473×
```

Latency reduction:

```text
≈ 99.79%
```

---

## 16.2 Complete Background Campaign

Before:

```text
199.199 s
```

After:

```text
42.213 s
```

Improvement factor:

```text
≈ 4.72×
```

Reduction:

```text
≈ 78.81%
```

---

## 16.3 LLM Runtime

Actual Qwen artifact-review inference remained approximately:

```text
5–7 seconds
```

Final measured D6 review:

```text
5.831 s
```

The model was therefore not the dominant source of the original multi-minute latency.

---

# 17. Correctness Was Preserved

The optimization did not trade correctness for latency.

Final production outcome:

```text
DELIVERABLE: 1
UNCERTAIN:   0
REJECTED:    0
FAILED:      0
```

The final artifact was committed through the same canonical acquisition and verification path.

The optimization changed orchestration timing, not validation standards.

---

# 18. Same-Resident GPU Admission

The system intentionally did not use a crude rule such as:

```text
GPU utilization < threshold
```

for local inference admission.

Instead it considered evidence including:

- exact configured model identity,
- model digest,
- runtime version,
- context length,
- VRAM size,
- measured free VRAM,
- calibrated required headroom,
- UI responsiveness,
- CPU pressure,
- disk latency,
- network conditions,
- RAM pressure,
- single-flight state,
- current local inference state.

This matters because a large resident model may occupy most VRAM while still having enough measured incremental headroom for a proven workload.

Conversely, low GPU utilization alone does not prove that another heavy inference request is safe.

---

# 19. Reviewer Safety Properties

The artifact reviewer used:

```text
temperature = 0
```

Review instructions required evidence quotes to be:

```text
one contiguous exact substring
```

The reviewer was not allowed to prove completeness by:

- paraphrasing,
- combining disjoint passages,
- reconstructing omitted material,
- or rewriting the source.

This was particularly important because the system was evaluating canonical ingestion material rather than conversational prose.

---

# 20. Cloud Escalation Policy

Normal business outcomes were not treated as infrastructure failures.

Examples that should not automatically trigger cloud escalation included:

```text
ZERO_RESULTS
normal REJECTED
normal UNCERTAIN
duplicate
resource defer
known bounded transient
known dependency outcome
```

Cloud escalation belongs to system-level failure conditions such as:

- repeated progress-invariant violation,
- exhausted local recovery,
- repeated identical failure signatures,
- planner path failure,
- unclassifiable infrastructure failure.

This avoided turning normal operational state into expensive or privacy-sensitive fallback behavior.

---

# 21. Test Methodology

The release process did not rely only on end-to-end success.

Evidence included:

- production SHA256 gates,
- AST patch validation,
- fresh-process import tests,
- persisted isolated Cron-store integration,
- fire-owner fencing tests,
- paused-job protection,
- one-shot protection,
- bounded retry-budget tests,
- scheduler shutdown tests,
- durable re-arm tests,
- production E2E timelines,
- baseline-versus-current shadow replay.

A representative final non-flaky regression gate produced:

```text
127 passed
2 skipped
3 deselected
0 failed
```

The three deselected tests were not silently ignored.

Each was independently investigated and reproduced against the pre-D5 baseline.

---

# 22. Flaky-Test Attribution

One Cron test asserted that immediately claiming a recurring job must always change its `next_run_at`.

Repeated fresh-process testing of the current code produced:

```text
8 PASS
2 FAIL
```

To determine whether D5 caused the behavior, a shadow package using the exact pre-D5 `jobs.py` was tested against the same target.

A 30×30 paired experiment showed:

```text
Baseline:
24 PASS
6 FAIL

Current:
19 PASS
11 FAIL
```

Because the baseline itself reproduced the same assertion failure, the test could not be classified as a D5-specific regression.

---

## 22.1 Heartbeat Timing Tests

Two additional heartbeat tests showed Windows timing sensitivity.

### Heartbeat refresh

```text
Baseline:
12 PASS
8 FAIL

Current:
15 PASS
5 FAIL
```

### Bounded heartbeat-error grace

```text
Baseline:
10 PASS
10 FAIL

Current:
14 PASS
6 FAIL
```

The corresponding heartbeat and claim functions were source-identical before and after D5.

These tests were therefore documented as:

```text
baseline-reproduced Windows timing flakes
```

rather than simply discarded.

---

# 23. The Test Harness Was Also a Failure Surface

Several engineering failures occurred in the diagnostic and deployment tooling itself.

These included:

### 23.1 Text Anchor Assumptions

An early patch harness assumed:

```text
import threading
```

occurred only once.

The source contained both module-level and function-local imports.

The candidate failed before construction.

The patching strategy was changed to AST-based structural targeting.

---

### 23.2 Manual PASS After Failure

At several points a PowerShell command threw an error, after which later commands were manually continued.

This could result in output such as:

```text
TEST FAILED
...
PASS
```

A printed `PASS` was therefore never treated as authoritative unless the actual test process had exited successfully.

---

### 23.3 Wrong Test Filename

A test was initially referenced as:

```text
test_recurring_again_redispatch.py
```

The actual filename was:

```text
test_recurring_eagain_redispatch.py
```

where `EAGAIN` referred to the error condition.

This produced a false release-gate failure before any tests ran.

---

### 23.4 PowerShell `$HOME`

PowerShell variable names are case-insensitive.

A test harness attempted to assign:

```powershell
$Home = ...
```

which collided with the readonly automatic variable:

```powershell
$HOME
```

The A/B experiment therefore executed zero trials.

This failure was retained as invalid evidence rather than silently rerun and forgotten.

---

### 23.5 Scheduler Sentinel Values

A timeline extractor assumed every `due` value was a normal Unix timestamp.

A value of:

```text
0.0
```

was passed into Windows timestamp conversion and caused:

```text
OSError: Invalid argument
```

The diagnostic code was changed to classify scheduler sentinel values explicitly.

---

### 23.6 Live Production Store Hashes

A regression harness initially assumed:

> If production `jobs.json` changes during tests, tests contaminated production.

That invariant was too strong because the real production scheduler remained live.

A later audit showed that the file changed at exactly the time a legitimate production job ran:

```text
Corpus aggregate incident review
status = ok
```

No test artifacts were present.

The correct invariant became:

> Search for test-origin semantic contamination, not merely byte-level mutation of a live store.

---

# 24. Reliability Invariants

The final system can be summarized by the following ownership chain.

```text
User bytes
    ↓
exact identity

Agent ingress
    ↓
structured terminal handoff

Durable campaign
    ↓
durable Cron owner

Scheduler wake
    ↓
latency hint only

Bounded resource recovery
    ↓
one fenced fast retry

Normal recurring schedule
    ↓
correctness fallback

Canonical commit
    ↓
origin notification
```

The most important architectural property is:

> Each state transition has one authoritative owner.

---

# 25. Durable State vs. Events

A recurring pattern throughout the project was separating correctness from latency.

The final design uses:

```text
Durable state = correctness authority
Wake/event    = latency hint
```

This has several useful consequences.

A lost wake does not lose work.

A duplicate wake does not authorize duplicate side effects.

A process restart does not erase the intended schedule.

A retry request is associated with the fire owner that created it.

A stale worker cannot commit the retry state of a newer owner.

---

# 26. Why the Model Was Not the Main Bottleneck

The user-visible system originally felt extremely slow.

A naive diagnosis might have been:

> The 27B local model is too slow.

Production traces showed otherwise.

Actual model review:

```text
~6 seconds
```

Original orchestration delays included:

```text
~155 s first scheduler opportunity
~155 s post-resource-defer gap
foreground post-ingest tool exploration
scheduler alignment
resource recovery
notification bookkeeping
```

The majority of latency existed outside token generation.

The key performance lesson is therefore:

> Measure orchestration before replacing the model.

---

# 27. Remaining Bottleneck

After D5, background Corpus execution is no longer the dominant user-visible delay.

The remaining large interval is increasingly located before campaign creation:

```text
Telegram message
    ↓
foreground Agent
    ↓
context / model processing
    ↓
tool routing
    ↓
formal ingest
    ↓
campaign created
```

A future phase should instrument:

```text
message received
request hook
model start
first token
tool decision
ingest call
campaign creation
terminal foreground reply
```

This work is intentionally separated from the Corpus scheduler changes.

The D5 scheduler path should remain frozen while the next layer is measured.

---

# 28. Threats to Validity

This study has several limitations.

## 28.1 Single Machine

The measurements were collected from one high-end Windows workstation.

Absolute latency values should not be generalized directly to other hardware.

## 28.2 Limited Production Runs

The detailed before/after results come from a small number of production E2E experiments rather than a large statistical benchmark.

The results are therefore best interpreted as a systems case study.

## 28.3 Windows Timing Behavior

Several upstream tests use 10–30 ms timing assumptions.

These exhibited significant jitter on Windows.

Baseline replay was necessary before determining whether a failure represented a regression.

## 28.4 Specific Resource Conditions

The final successful D6 path encountered:

```text
HEAVY_WARMUP
```

The separate `RESIDENT_RESOURCE_CONTENTION` recovery branch was tested structurally and in regression, but was not the primary reason observed in the final production E2E run.

## 28.5 Foreground Latency Remains

The project optimized background Corpus execution.

Foreground Telegram/Agent latency remains a separate optimization target.

---

# 29. Lessons Learned

Several engineering principles emerged from this project.

### 1. Final-answer correctness is not enough.

A system can end correctly while corrupting data or wasting minutes between valid states.

### 2. Never let an LLM recreate canonical user bytes.

Analyze them, but do not silently rewrite them.

### 3. Durable state should own correctness.

Events and notifications should accelerate progress, not define it.

### 4. Retry policy belongs near ownership boundaries.

A child workload should not mutate terminal scheduler state that the scheduler will later overwrite.

### 5. Recovery must be bounded.

A fast retry is useful only if it cannot become an infinite retry loop.

### 6. Failure evidence should be preserved.

Failed candidates, rollbacks, flaky tests, and broken harnesses are part of the engineering record.

### 7. Regression tests need baseline replay.

A failing test is not automatically a regression.

### 8. Test infrastructure is production infrastructure.

A broken test harness can generate both false failures and false confidence.

### 9. Optimize the measured bottleneck.

The model took six seconds.

The pipeline took minutes.

---

# 30. Conclusion

This project began with a single corrupted character:

```text
他
```

becoming:

```text
" he"
```

The resulting investigation expanded into a full reliability redesign of a local LLM Agent–Corpus pipeline.

The final production architecture established:

- exact user-input identity,
- deterministic terminal errors,
- explicit foreground/background ownership,
- durable Cron re-arm,
- low-latency scheduler wake,
- calibrated same-resident resource admission,
- bounded recovery,
- fire-owner fencing,
- one-shot post-run retry,
- recurring fallback,
- and evidence-based regression methodology.

The most visible production improvement was:

```text
155.273 s
→
0.328 s
```

for resource-defer continuation.

That is approximately:

```text
473× faster
```

with approximately:

```text
99.79% latency reduction
```

The complete background campaign improved from:

```text
199.199 s
```

to:

```text
42.213 s
```

while preserving:

```text
DELIVERABLE: 1
UNCERTAIN:   0
REJECTED:    0
FAILED:      0
```

The model itself still required approximately six seconds.

The rest was software engineering.

The central conclusion is therefore:

> **Production Agent bottlenecks are often orchestration problems, not intelligence problems.**

And the central reliability rule is:

> **Durable state is the authority. Everything else is an optimization.**

---

# Appendix A — Major Production Measurements

## Baseline scheduler behavior

```text
job created
06:51:49

review start
06:54:24

≈155 s
```

---

## D4 — Before post-run retry

```text
campaign created
13:02:08.269

first native start
13:02:10.299

first native finish
13:02:35.095

second native start
13:05:10.368

review start
13:05:15.219

review finish
13:05:21.114

campaign stop
13:05:27.468
```

Derived:

```text
campaign → first native        2.030 s
first native runtime          24.797 s
finish → second start        155.273 s
campaign total               199.199 s
review runtime                 5.895 s
```

---

## D6 — After post-run retry

```text
campaign created
14:02:25.646

first native start
14:02:28.229

first native finish
14:02:52.355

second native start
14:02:52.683

review start
14:02:56.544

review finish
14:03:02.375

campaign stop
14:03:07.859
```

Derived:

```text
campaign → first native        2.583 s
first native runtime          24.125 s
finish → second start          0.328 s
review runtime                 5.831 s
campaign total                42.213 s
```

---

# Appendix B — Selected Production Hashes

These hashes are preserved as engineering evidence for the tested configuration.

## Scheduler provider

```text
614C8E19D15B490CB8F80C6236E5E7CE0542C185E3FC47B450363DB66B6E73CB
```

## D3 QoS

```text
97EAF1C26A74360E7972DDF7E154A0B733D3DA2AF8F7C93CE9A63745AE3BFEDF
```

## D5 Cron jobs

```text
B785FA55DA36169EBF4C4BF15C8901A0A81C804337521021EBB7E66B3A5AF6C4
```

## D5 lifecycle

```text
E5A0A1028B52CA2290A9DC0873746D2FE890AF98AC96E04CBA8A167516E5D665
```

## D5 qualification

```text
5D5903E2EEDDB6AD664C5CD0292447C26D106E3179F4AAC241FEBB768D315DDB
```

---

# Appendix C — Known Baseline Timing Flakes

The following tests were independently reproduced against pre-D5 code and therefore were not classified as D5 regressions.

```text
test_claim_succeeds_once_then_blocks

test_run_one_job_refreshes_fire_claim_in_profile_store

test_repeated_heartbeat_errors_cancel_after_bounded_grace
```

They remain documented known limitations rather than silently ignored failures.

---

# Appendix D — Public Evidence Policy

Public evidence should preserve:

- timing data,
- state transitions,
- failure classifications,
- test outcomes,
- code hashes,
- architectural diagrams,
- sanitized logs.

Public evidence should remove:

- Telegram chat IDs,
- personal usernames,
- API keys,
- tokens,
- private conversation content,
- company-confidential material,
- personally identifying local paths where unnecessary.

---

# Status

**Production validated**

```text
First native wake:
~155 s → ~2–3 s

Post-resource-defer continuation:
155.273 s → 0.328 s

Background campaign:
199.199 s → 42.213 s

LLM review:
~5–7 s

Final result:
DELIVERABLE 1
UNCERTAIN   0
REJECTED    0
FAILED      0
```

---

*This document is an engineering case study based on observed production behavior and controlled regression experiments. It is not intended to claim universal performance across hardware, operating systems, models, or Agent frameworks.*
