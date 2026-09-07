## Workflow

These steps map to the four gates from [Set the autonomy level first](#set-the-autonomy-level-first):
sourcing (before step 1) is the **selection** gate; steps 1–2 are the **shape +
expectations** gate; steps 5–6 are the **calibration** gate; step 7 is the
**push** gate. In *autonomous* mode you flow through all of them and summarize in
a decision log; in *checkpoint* mode you pause at each with a proposal, and at
calibration you hand the driver the thread link + login to review the real build
(see [Set the autonomy level first](#set-the-autonomy-level-first)). Calibration
(step 6) always surfaces meaning-flipping verdicts explicitly regardless of level.

1. **State the must-haves first.** From the conversation alone, list what every
   correct workflow must do (trigger type, essential operations, gating
   condition) — those become draft `outcomeExpectations`. Required fields:
   `conversation` (≥1 turn, first `user`), `complexity`, `tags`, and **at least
   one** of `executionScenarios` / `processExpectations` / `outcomeExpectations`.
2. **Draft the case** from the template below; validate it loads (see
   "Validate").
3. **Smoke-test the *environment* with one case before any batch.** Run a single
   case end-to-end first. This validates auth / model / `--base-url` / the built
   dist for ~1/Nth the cost — distinct from validating a *case*. If that one case
   crashes at execution (especially with an identical error you'd expect to hit
   every case), fix the environment before running the batch (see "A red is
   signal" → environment check). Running 15 cases only to discover a stale-dist
   crash on all of them wastes a full run.
4. **Build it once** against a running instance (see
   [`running-evals.md`](../../running-evals.md)) with `--keep-workflows` so the built
   workflow stays for inspection.
5. **Inspect** — read the built workflow (the run prints `BUILT (<id>)`; fetch
   via `GET /rest/workflows/<id>`) and the HTML report's transcript to see what
   the agent actually did.
6. **Calibrate — sharpen assertions; never dull them to force a green.** Fix
   assertions that are genuinely mis-sized: relax one that pins a choice the
   conversation left open (so a valid *alternative* build wrongly fails), tighten
   one a wrong build would slip past, and phrase `executionScenarios` to match how
   the workflow runs on mocked data. But when a scenario goes red because the
   build has a real gap, or because the harness can't exercise it, **that red is
   the result — keep it and surface why** (see "A red is signal", below). Never
   delete a scenario, weaken an assertion, or drop to build-only just to make the
   run green.
7. **Push to the suite — do NOT commit the JSON.** Once calibrated, push the case
   into its curated lang-tracer suite with `eval:langtracer-push` (see
   [Push to a lang-tracer suite](#push-to-a-lang-tracer-suite)); the suite is the
   case's home, not the repo. Leave the `data/workflows/*.json` file uncommitted
   (or delete it once it's in the suite). Committing new case JSONs into the repo
   is no longer the approach. (Exception: seeded cases can't be pushed — an
   `inline`-seeded case stays committed JSON, a `replay` case is
   a local throwaway; see [`case-shapes.md`](../../case-shapes.md).) For a sourced case,
   finish by **linking it to its source thread/finding** over the MCP — see
   [Link the pushed case to its source](#link-the-pushed-case-to-its-source-provenance-step--always-do-this).

`--iterations N` is available to measure flakiness (pass@k / pass^k) — reach for
it when you suspect a case is non-deterministic or before promoting it to a
gated tier, not as a routine step (each iteration is a full build + execution).

Gut-check: if you can't picture a plausible *wrong* build that this case
reliably turns **red**, the assertions are too loose to guard anything.

**Confirm the precondition fired, not just the green.** For any *conditional*
assertion — "when X happened, the agent did Y" (most `processExpectations`, and
any behaviour case) — a pass has two readings: the agent did Y, or **X never
happened** and the assertion passed vacuously. A behaviour case that hinges on
the mock producing a specific failure (e.g. an AI node simulated to empty so a
downstream parse node fails) is the classic trap: if the mock instead returns
parseable data, the failure never occurs and the case guards nothing while
showing green. Calibration must read the execution trace and the agent's
`finalText` (`buildTrace.finalText` in the verifier snapshot, or the HTML report)
and verify X actually materialised — the direct-loop `eval-results.json` does not
persist per-expectation judge reasoning, so pass/fail alone can't tell you which
reading you got.

**A sourced failure that no longer reproduces is still worth keeping — it's now a
regression guard.** When you encode a real failure and calibration shows the
current build handling it correctly (behaviour drifts across versions), the case
doesn't lose value: it flips from *capability-gap* (currently red) to *regression
guard* (currently green, catches a re-introduction). Keep it — but only after the
non-vacuous check above proves it *would* turn red on the bad behaviour, else the
"guard" guards nothing.

## `dataSetup` and the mock layer

The harness mocks by **intercepting outbound HTTP requests to external services**
and having an LLM answer them from the node's config and API docs. It does **not**
let you set a node's output directly, and it does **not** mock n8n internals
(Code/Set/Merge/IF/Switch run for real on the mocked data; triggers and DB
nodes get LLM-generated pin data). So:

- Write `dataSetup` as **what each external service returns** ("the GitHub
  request returns three issues labelled bug"), not as node outputs or internal
  state.
- The strongest scenarios exercise **external-service responses** — that's what
  the harness reproduces most faithfully.
- **Data Table *reads* are pinned to the scenario.** A read op (`get` /
  `rowExists` / `rowNotExists`) is treated as the scenario's "stored state" and
  pinned with data derived from your `dataSetup`, so change-detection / dedup /
  "last seen" scenarios *can* be exercised — describe the stored rows in
  `dataSetup`. Two caveats: the pinned rows are LLM-generated (steered, not
  byte-exact — don't assert exact values off them), and *writes/inserts* aren't
  pinned (they hit the real per-thread table, recreated schema-only with **no
  rows**), so read-after-write within one run isn't faithful — the read reflects
  `dataSetup`, not what the run just wrote. A third caveat: only Data Table
  *reads* are seedable this way — **dedup / change-detection built on workflow
  static data** (`removeItemsSeenInPreviousExecutions`, `$getWorkflowStaticData`)
  is **not** seedable, because static data starts empty every run, so such a
  scenario reds vacuously (it sees everything as "new"). To get a seedable
  change-detection scenario, steer the build toward a Data Table; otherwise
  accept the static-data red as a harness limit and carry the logic in
  `outcomeExpectations`. Note the agent may *choose* static-data dedup on its own.
- Don't assert exact counts that depend on mock generation ("exactly 7 posts").
  Say "fewer than the original 10".

## Negative execution scenarios

Don't stop at the happy path — but only assert graceful handling the prompt
actually implied. Most agent-built workflows don't add error handling by
default, so "the workflow crashes on bad input" is a legitimate builder finding,
not a test-case bug. Where graceful handling *is* expected, phrase
`successCriteria` as the *absence* of the wrong action ("no alert is sent", "run
completes without error") as much as the presence of the right one: empty /
not-found source, source error / timeout, malformed response.

## Outputs of a run

- **`workflow-eval-report.html`** (in the run's `.data/` dir) — the highest-value
  view: full conversation transcript with tool calls, per-node execution traces,
  the exact intercepted requests and the mock responses, Phase-1 hints, verifier
  reasoning, and the workflow-check rubric. Human-oriented; start here when
  debugging.
- **`eval-results.json`** — structured results (the machine-readable artifact;
  the direct loop produces this even with no LangSmith). Good for an LLM or
  script to parse. **For per-case attribution under concurrency, parse this, not
  the streamed verbose log** — with more than one lane the log lines interleave
  across cases, so a `[scenario] FAIL` line in the stream can't be reliably tied to
  its case. Authoritative fields: `testCases[].buildSuccessCount`,
  `buildExpectationResultsPerRun[][].{pass,reason}`, and
  `scenarios[].runs[].{passed,failureCategory,rootCause,execErrors}`.
- **`eval-pr-comment.md`** — the rendered PR comment (aggregate + regression
  comparison), always written.

## Validate (before running)

```bash
cd packages/@n8n/instance-ai
npx tsx -e "import {loadWorkflowTestCasesWithFiles} from './evaluations/data/workflows/index.ts'; console.log(loadWorkflowTestCasesWithFiles('<slug>')[0].fileSlug)"
```

## Running

You need a running n8n instance with Instance AI enabled and a working sandbox;
point the eval at it. The harness runs in three modes — **direct loop** (no
LangSmith; `eval-results.json` only), **LangSmith** (also records an experiment
+ regression comparison), and **prebuilt** (`--prebuilt-workflows`, score
existing workflows). Narrow a run with `--filter <slug>` / `--tier <name>` /
`--exclude`. See [`running-evals.md`](../../running-evals.md) for the run recipes,
parallel lanes, tiers, and baselines, and the
[README](../../../../../packages/@n8n/instance-ai/evaluations/README.md) for the full
flag list. Run with `--keep-workflows` when you want to review a build by hand —
in *checkpoint* mode calibration this is how the driver opens the built thread
(`<base-url>/assistant/<threadId>`) and workflow on the instance.

