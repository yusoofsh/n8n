
# Create an Instance AI workflow eval

Each eval is **one JSON case** — authored locally as a file in
`packages/@n8n/instance-ai/evaluations/data/workflows/` (the disk loader
auto-discovers `*.json`, no registration step), with a LangTracer suite as its
durable home. Cases validate against
[`harness/schema.ts`](../../../../../packages/@n8n/instance-ai/evaluations/harness/schema.ts)
(`.strict()` — unknown keys fail at load). The eval
[README](../../../../../packages/@n8n/instance-ai/evaluations/README.md) is the
exhaustive field reference; this skill is the opinionated *how*.

> **Committing new case JSONs into the repo is no longer the recommended
> approach.** Author the file locally (uncommitted), calibrate it against a real
> build, then **push it to a lang-tracer suite** with `eval:langtracer-push`
> (see [Push to a lang-tracer suite](#push-to-a-lang-tracer-suite)) —
> `--suite baseline` for the consolidated corpus n8n CI runs, or a dedicated
> capability suite like `agents`.
> The suite is the home for the case; the eval CLI reads it back via
> `--source langtracer`. You still write the JSON file — it's just the input to
> the push, not a committed artifact.
>
> **Exception — seeded cases.** The case-write API has no `seed` field, so seeded
> cases are never pushed. A `seed.mode: "replay"` case is a local throwaway (don't
> commit it either — it dies when its trace is pruned); a `seed.mode: "inline"`
> case isn't transient and has no suite home, so it's the one sanctioned
> exception — it lives as committed JSON. See [`case-shapes.md`](../../case-shapes.md).

## Set the autonomy level first

**Before you source, draft, or run anything, decide how hands-on the driver
wants to be — and say it back.** This skill runs at one of two autonomy levels.
If the request makes the level clear ("just author and calibrate it yourself" vs.
"stop me at each step", or an explicit mode), adopt it, state it in one line, and
note how to override (e.g. "say 'stop me at calibration' to add a checkpoint").
**If it's not clear, ask the driver one question** offering the two levels
*before* doing any work.

The skill has four natural decision **gates** — **selection** (which real
failure to encode), **shape + expectations** (archetype, must-haves, scope
trim), **calibration** (classify each red and resolve keep/loosen/drop), and
**push** (kind + tier). The level decides what happens at each gate:

| Level | Who decides when to stop | Behaviour |
|---|---|---|
| **autonomous** | agent | Runs all four gates start-to-finish; reports a **decision log** at the end for the driver to review. |
| **checkpoint** | driver, per gate | Stops at each gate with a compact **proposal + recommendation**; driver says "go" or redirects. At the **calibration** gate, hands the driver a link to the just-built thread on the live instance plus login credentials so they can review the real conversation and workflow themselves before confirming (below). |

**Calibration is special-cased at both levels.** A calibration verdict that
flips a case's *meaning* — a real capability-gap red vs. a harness-caused red, or
any loosening that would let a known-bad build pass — is **surfaced explicitly**
(interactively in checkpoint; in the decision log in autonomous), never silently
committed. It's the one call where a quiet mistake corrupts the suite, so it
never fully auto-commits.

**Checkpoint calibration — review the real thread on the instance.** Because the
calibration verdict is trust-critical, in checkpoint mode you don't ask the
driver to trust your reading of the run. You built the case against a live
instance with `--keep-workflows` (step 4), so the thread and the workflow are
still there — hand the driver a direct link and let them look:

- **Thread:** `<base-url>/assistant/<threadId>` — the exact conversation the case
  ran (the run prints the `threadId`; the built workflow prints as `BUILT (<id>)`
  and opens at `<base-url>/workflow/<id>`).
- **Login:** the email + password the eval signs in with (the owner you seeded on
  the instance — see [`running-evals.md`](../../running-evals.md); the default local
  seed is `nathan@n8n.io` / `PlaywrightTest123`).

Present, per red: the assertion, whether it went green/red, your proposed
classification (real capability gap / harness limitation / noise) and
keep/loosen/drop, and the review link. The driver logs in, reads the thread and
the workflow, and confirms or redirects before you write the verdict back into
the case `description`.

## Where the best cases come from

The strongest cases encode a **real** failure, not an invented premise. Two
connections help you find and verify one: **LangTracer** clusters real
conversations into capability-gap themes (discover what actually fails, at
scale), and **LangSmith** holds the raw traces (verify exactly what happened in a
run). LangTracer is the discovery layer; the durable artifact is almost always a
synthetic case you author from what you learn (use `seed.mode: "replay"` only per
[`case-shapes.md`](../../case-shapes.md)). See
[`sourcing-cases.md`](../../sourcing-cases.md) for connecting the MCPs and the
discover → verify → encode workflow.

## Pick the case shape first

The corpus is four archetypes. Decide which you're writing before you draft — it
determines the fields, the grading, and how you validate. They compose (a seeded
case can still assert outcome), but the primary shape drives the work.

| Archetype | Question it answers | Primary fields |
|---|---|---|
| **Build** (default) | Does the workflow the agent builds actually *work*? | `outcomeExpectations` + `executionScenarios` |
| **Behaviour / process** | Does the agent *converse* correctly (ask the right clarifying question, not re-ask, honour a correction, respect plan approval)? | `processExpectations` + multi-turn director script; often **build-only** |
| **Credential** | Does the build behave correctly given a specific credential view? | `credentials[]` |
| **Seeded** | Reproduce a conversation mid-thread and drive the turn under test | `seed` (`mode: "inline"` or `"replay"`) |

**Build** is documented in full below. The other three, the director-script
vocabulary, and the seeding modes are in [`case-shapes.md`](../../case-shapes.md).

## Core principle (all shapes)

**Write expectations from intent, then calibrate against a real build.** Decide
up front what makes *any* correct solution correct — the must-haves implied by
what the user actually said — then build the workflow once for real to calibrate
granularity: loosen what's over-specified, confirm the must-haves are
achievable, and catch requirements the agent legitimately satisfies a different
way. Don't transcribe one observed build into assertions — that overfits the
eval into "did the agent reproduce that run" instead of "did it solve the
problem."

**Keep the conversation in the user's voice.** State the goal and real
constraints the way a user would — don't name node types, wire up the structure,
or restate your `outcomeExpectations` in the prompt. If the conversation spells
out the build, the case only tests whether the agent can follow instructions and
the expectations become tautological; the gap between what the user asks for and
how a correct workflow realizes it is the capability under test. Even when the
anchor *is* honoring a user's stated technical preference, phrase it as their
need + constraint ("I need field X and the built-in node doesn't expose it, so
pull it straight from the API") — not as an implementation spec ("use an HTTP
Request node").

**Write the conversation in English** unless the user asked otherwise (or the
case exists specifically to test non-English handling). Sourced real threads are
frequently non-English — translate the intent into English when you rewrite the
prompt in the user's voice; the failure mode is the anchor, not the original
language.

**Trim to the smallest multi-turn conversation that reproduces the issue.**
Real sourced threads are long (dozens of turns of setup, debugging, and
tangents) — do **not** transcribe them. Distill to the fewest turns that still
drive the build or behaviour under test. Every retained turn must earn its place:
a turn stays only if it is *load-bearing* — a value the agent must ask for
(withheld until asked, via a director note), a correction/push-back the case
exists to test, or a plan approval that gates the build. If removing a turn
doesn't change what's tested, remove it. **Collapse to a single turn** whenever
the whole request can be stated at once without a load-bearing exchange; keep it
multi-turn *only* for those exchanges, and keep each director script in one turn
(don't fabricate assistant "done" turns to sequence steps — see
[`case-shapes.md`](../../case-shapes.md)). A minimal conversation isolates the
capability; a transcribed one buries it in noise and tests instruction-following.

**Size the build, not just the assertions.** Real sourced prompts are often
kitchen-sink ("production-ready, runs forever, 3 feed posts *and* 8 stories a
day", "generate 50 articles daily") and reliably blow the ~900s build budget (see
"Known harness limitations"). A **faithful trim is a legitimate authoring move**:
reduce batch sizes, drop one of several parallel pipelines, or merge adjacent AI
steps so the case builds within budget — then note the reduction in the case
`description` ("the original request also asked for an 8-stories/day pipeline;
scoped to feed posts so it builds in budget"). Keep the capability under test; cut
the combinatorial bulk. A case that never builds tests nothing.

## A red is signal — surface it, don't work around it

Calibration exists to right-size assertions, **not** to make a case pass. When a
run turns a scenario or expectation red, classify the red first — then keep it.

**First rule out the environment.** Before reading any red as a signal about a
case, check the shape of the failures across the run. If **every case fails the
same way** — every scenario with the *same* execution error while builds succeed,
*or* every **build** erroring identically before it starts (an `Agent error:
Something went wrong…` with **zero tool calls**) — that is almost never the cases;
it's a broken environment, most often a **stale dist** after a branch or worktree
switch. Two shapes to know:

- **Stale `packages/core` / `packages/cli` dist** — a refactor moved a runtime
  export and the built dist still calls the old one, so *builds succeed but every
  execution fails the same way* (e.g. `(0 , n8n_workflow_1.createDeferredPromise)
  is not a function` after `createDeferredPromise` moved to `@n8n/utils`).
- **Stale/half-built `@n8n/instance-ai` dist** — every run errors *before building*
  (`Agent error…`, zero tool calls) and the instance log shows `Cannot find module
  '@/utils/...'` from `dist/skills/*.js`: the build's `tsc-alias` step (which
  rewrites `@/` path aliases to relative requires) didn't complete, so the dist is
  internally inconsistent.

- **Out-of-sync `node_modules`** — `pnpm build` itself dies early with `Cannot find
  module '@n8n/<pkg>'` even though that package is a declared `workspace:*`
  dependency *and* has a `dist/`. The workspace symlink is missing from the
  consumer's `node_modules` (typical after a branch or worktree switch). Confirm
  with `ls -d packages/<consumer>/node_modules/@n8n/<pkg>`; fix with a plain
  `pnpm install` — no need for the heavier `pnpm reset --full`.

Fix it, don't calibrate around it: run a full ordered `pnpm build` (a targeted
`--filter` build can fail on unrelated stale-dep type errors; for the instance-ai
shape, `cd packages/@n8n/instance-ai && pnpm build` runs `tsc && tsc-alias`), then
**restart the instance** — the running node process holds the old dist in memory,
so rebuilding on disk changes nothing until restart (and `kill` by env-var pattern
misses it — kill the actual `lsof -t -iTCP:<port>` PID). Re-probe one case, confirm
it builds and executes, then re-run the batch. Only once uniform environment
failures are excluded do the three categories below apply:

- **Real build / capability gap** — the agent's workflow is wrong or missing
  something the user asked for (a miswired branch, a missing retry, wrong field
  keys). This is exactly what the eval is for. **Keep it red.** Don't loosen the
  assertion or drop the scenario; a currently-red gap is the capability signal
  today, and a re-introduction guard once the builder improves.
- **Harness limitation** — the build is correct but the mock/execution layer
  can't exercise the path (see "Known harness limitations", below). **Keep the
  scenario and say so in its `description`** — that this red is harness-caused,
  not a build defect — so nobody misreads it as a product bug. Keep it out of
  gated tiers if it hard-fails every run; when the harness gains the capability it
  starts earning its keep with no re-authoring.
- **Genuine non-determinism** — the *same* build flips green/red across runs.
  This is the only real "noise". Confirm it with `--iterations N` before calling
  it flaky, then de-tier and note it; deletion is the last resort.

**Annotate every kept red in the case `description` with a scannable prefix** so a
future reader tells the two apart at a glance. Use `Harness note: …` for a
harness-caused red (name the limitation and why the build is still correct), and
`Capability-gap finding: current build reds because <X> — a real builder bug
(flips to a regression guard once fixed)` for a real gap. Consistent prefixes keep
the corpus greppable and stop harness reds from being misread as product bugs.

The one move to never make is **working around a red by weakening what the case
checks** — deleting a failing scenario, loosening an assertion until a wrong
build would pass, or quietly converting to build-only. That makes the suite look
greener than the product is, which is the opposite of the eval's job: bugs and
harness gaps are the deliverable, so **highlight them, don't engineer around
them**. If you catch yourself editing a case so that a known-bad build would now
pass, stop.

**Who confirms the classification depends on the autonomy level.** In
*checkpoint* mode the keep/loosen/drop decision is the driver's to confirm: you
hand them the thread link + login (see [Set the autonomy level
first](#set-the-autonomy-level-first)) so they can review the real conversation
and workflow, then you write the agreed `Harness note:` / `Capability-gap
finding:` prefix back into the case `description`. In *autonomous* mode the agent
proposes it explicitly in the end-of-run decision log. Either way the
classification is stated in the open, never silently committed — misreading a
harness red as a real gap (or the reverse) is the one calibration mistake that
quietly corrupts the suite.

## Example

Minimal build case:

```json
{
  "description": "What this case tests.",
  "conversation": [{ "role": "user", "text": "<the build prompt>" }],
  "complexity": "medium",
  "tags": ["build", "<nodes>", "<concepts>"],
  "triggerType": "schedule",
  "outcomeExpectations": ["<a must-have any correct workflow satisfies>"],
  "executionScenarios": [
    {
      "name": "happy-path",
      "description": "<what this run exercises>",
      "dataSetup": "<what the external services return>",
      "successCriteria": "<observable proof the run succeeded>"
    }
  ]
}
```

A fuller case with a multi-turn director script (withhold a value until asked,
push back on a wrong plan):

```json
{
  "description": "Scheduled GitHub-bugs digest to Slack. Repo and channel are withheld until the agent asks; the plan must filter to the 'bug' label before it's approved.",
  "conversation": [
    { "role": "user", "text": "Every weekday at 9am, fetch this week's open bugs from our GitHub repo and post a short summary to Slack." },
    { "role": "assistant", "text": "Which repo and which Slack channel should I use?" },
    { "role": "user", "text": [
        "[Withhold the repo and channel until the agent asks; then say the repo is 'acme/widgets' and the channel is '#eng-bugs'.",
        "When the agent shows a plan or setup card, reject it unless it filters issues to the 'bug' label — a digest of ALL issues is wrong. Once it filters to bugs, approve.]"
    ] }
  ],
  "messageBudget": 8,
  "complexity": "medium",
  "tags": ["behaviour", "build", "schedule", "http-request", "slack"],
  "triggerType": "schedule",
  "processExpectations": [
    "The agent asked for the repo and Slack channel before building, since the prompt named neither.",
    "The agent's final plan filtered issues to the 'bug' label — if its first attempt didn't, it corrected after the user pushed back rather than summarizing all issues."
  ],
  "outcomeExpectations": [
    "A Schedule Trigger runs the workflow on a recurring weekday-morning cadence.",
    "Open issues are fetched from GitHub (HTTP Request or GitHub node) and filtered to the 'bug' label before the summary is built.",
    "One Slack message summarizing the fetched bugs is posted to the #eng-bugs channel the user gave."
  ],
  "executionScenarios": [
    {
      "name": "posts-bug-digest",
      "description": "Three open bugs are returned; a summary is posted to Slack",
      "dataSetup": "The GitHub issues request returns three open issues labelled 'bug' ('Login 500', 'Timezone off by one', 'CSV export truncates'). The Slack postMessage call returns { \"ok\": true, \"ts\": \"1700000000.0003\" }.",
      "successCriteria": "The run completes without errors and posts one Slack message to #eng-bugs that references the three fetched bug titles."
    }
  ]
}
```

What each piece is doing:

- **`conversation[0]` is sent to the builder raw.** The opening turn is the real
  prompt — never put a `[director note]` in it (it would leak verbatim).
- **The `[bracketed]` turn is a director script** for the user-proxy — behaviour,
  never spoken. Here it withholds values until asked and rejects a plan that
  misses the label filter. Keep the whole script in one turn and encode ordering
  inside it (don't fabricate assistant "done" turns to sequence steps — see
  [`case-shapes.md`](../../case-shapes.md)). `applies-each-change-when-asked` (in the
  `baseline` LangTracer suite) is a good real example.
- **`dataSetup` describes only what external services return.** That's the layer
  the harness controls (below).

### Known harness limitations that turn scenarios red regardless of the build

These produce a **reliable** red on a *correct* build. Don't engineer around them
— write the scenario for the behaviour you want and note in `description` that the
red is harness-caused (per "A red is signal", above):

- **Resource-locator fields left empty for setup** (Google Sheets / Drive /
  Calendar and similar node pickers). The agent legitimately leaves the
  document/folder/calendar ID blank for the user to pick at setup; the mock
  substitutes `__evalMockResource`, and the node then crashes looking it up
  ("Sheet with ID __evalMockResource not found", or "Cannot read properties of
  undefined"). Any scenario whose success path runs *through* such a node
  hard-fails before anything downstream executes.
- **Trigger and Data-Table-read pin data is *LLM-generated*, so not byte-exact.**
  Both are steered by your `dataSetup` (see the mock-layer section above — you
  *can* influence what a trigger emits or what a stored-row read returns), but
  because the values are generated, a scenario that asserts exact values or counts
  off them is flaky. Assert shape/branch/relative facts, not exact figures. (The
  residual hard red here: polling / form triggers still occasionally fail to load
  entirely — "workflow not found".)
- **Mock response shape** — the LLM-generated mock response can omit the real
  envelope, crashing a downstream parse/format node. Recurring, reproducible
  shapes to expect (all produce a red on a *correct* build):
  - **OpenAI structured output** — historically the mock returned a plain
    `{content: "..."}` instead of the Responses envelope
    (`output[0].content[0].text`), so a **Structured Output Parser** /
    **Information Extractor** / **Text Classifier** got nothing and errored with
    **`Model output doesn't fit required format`**. The Responses-envelope
    normalizer (PR #33578, merged) fixes the flat-envelope case, so many of these
    now execute cleanly. A **narrower residual red remains** for structured-output
    schemas declared with strict **`additionalProperties: false`**: the normalized
    `output` wrapper (and any extra fields the mock invents, e.g. `subject`/`date`)
    violate the strict schema, so the node still rejects the mock output. Both the
    old and residual forms are the *mock*, not the build — carry correctness in
    `outcomeExpectations` and note the red as harness-caused.
  - **Gmail** mock returns headers as top-level capitalized fields (`From`,
    `Subject`) instead of under `payload.headers`, so a Code/Filter node reading
    the sender/subject gets empty strings (e.g. a "drop no-reply senders" safety
    gate lets everything through). Assert the *wiring/ordering* of such a gate in
    `outcomeExpectations`, not its runtime effect in a scenario.
  - A less-common API (e.g. Gemini's top-level `candidates`) can omit its envelope
    the same way.
  - **Google Drive resumable upload** — the initiate-upload mock omits the
    `Location` header carrying the session URL, so a Drive file-upload node fails
    with a 400. Any build that uploads a generated image/file to Drive can red on
    this.
- **Agent-tool nodes can't be executed standalone.** An AI-Agent *tool* node
  (`toolHttpRequest` and other `supplyData`-only LangChain nodes with no `execute`
  method) only runs when the agent invokes it; the harness executing it directly
  fails with `has a "supplyData" method but no "execute" method`. A near-universal
  red for chat-trigger / AI-agent build cases whose tool is an HTTP-request tool —
  the build is correct, so carry correctness in `outcomeExpectations` (agent wired
  to trigger + model + tool) and note the execution red as harness-caused.
- **Poll/wait loops can't be fast-forwarded.** A workflow that submits an async
  job then polls for completion (generate → poll status until ready → download)
  can't advance the mocked status deterministically, and a `Wait` node runs in
  real time, so the scenario reds with an execution timeout (`framework_issue`).
  The build is correct — carry it in `outcomeExpectations` and note the red as
  harness-caused.
- **A build can time out and produce no scored result at all** — the run reports
  `BUILD FAILED: Run timed out` and zero graded expectations. Don't assume "spec
  too big": the more common cause is a **single-prompt case where the agent asks
  a clarifying `ask-user` question** and the build hangs on the unanswered
  question until the per-iteration timeout (see [`case-shapes.md`](../../case-shapes.md)
  — only confirmations auto-approve). Before treating a timeout as spec size,
  **classify it**: read the agent's final response in the report / trace (did it
  ask a question? flag an infeasibility? or genuinely churn through a huge
  build?), and **re-run the case solo (`--concurrency 1`)** — concurrency both
  masks a stalled build (it hits the cap) *and* can time out a perfectly healthy
  build purely by queueing it behind the per-instance build cap (default 4), so a
  solo run either surfaces the real reason in seconds or simply passes outright.
  Fix per cause: a solo run that now passes → it was **concurrency contention**,
  not the case (split big batches across lanes — see
  [`running-evals.md`](../../running-evals.md)); a clarifying-question stall → author
  multi-turn with a director note that pre-answers it; a genuine infeasibility → it's an
  infeasibility/honesty behaviour case (`processExpectations`), not a build case;
  a true oversized spec → the timeout is itself a finding, but note it so the
  zero isn't mistaken for a scored failure.

## outcomeExpectations vs processExpectations

Both are natural-language assertions graded by the same Sonnet judge, and each
**counts as a pass-rate unit**. They judge different surfaces:

- **`outcomeExpectations`** — the **resulting workflow**, judged from the
  workflow JSON. Assert node choices and configuration, connection topology and
  branch wiring, data/expression references, trigger cadence, gating conditions.
  They run everywhere, including prebuilt/MCP runs (no transcript needed).
- **`processExpectations`** — **how the agent behaved during the build**, judged
  from the transcript. Assert clarifying questions asked (or not re-asked),
  tool-call behaviour, plan/approval handling, batching, honouring a correction,
  ordering. They need a transcript, so they're **skipped in prebuilt/MCP runs**.

Rule of thumb: an assertion about *the artifact* is an outcome expectation; an
assertion about *the conversation or the agent's choices along the way* is a
process expectation. A case with **no** `executionScenarios` is a valid
**build-only** case, graded by these expectations plus the workflow checks.

## Sizing each assertion

Right-size against **what the agent was actually told**. An assertion is
well-sized when every correct build passes it and a wrong or lazy build fails
it — and it holds the agent only to what the conversation specified, not to one
run's arbitrary choices. Two failure modes:

- **Too tight** — pins a choice the conversation *left open*. If the prompt never
  named a vendor, "calls flightaware.com" fails a valid build that used a
  different source. **But if the conversation specified it, pin it** — when the
  user said "email me via Gmail," "sends via a Gmail node" is correct and
  *required*, not too tight.
- **Too loose** — a non-solution would also pass. "Fetches data from somewhere"
  passes a workflow that fetches but never compares — it doesn't prove the
  change-detection the prompt asked for.

Quick check — the *substitution test*: would a reasonable alternative
implementation *of what the user asked for* still pass? Examples (flight-status
case, where the source and channel were **left unspecified**):

| Verdict | Assertion | Why |
|---|---|---|
| ❌ too tight | "Has an HTTP Request node calling `flightaware.com`" | Vendor was unspecified; a valid AeroDataBox build fails. (If the user *had* said "scrape FlightAware", this would be correct.) |
| ❌ too tight | "Publishes via HTTP Request nodes" | Pins the *transport* when a first-party node is the idiomatic path — e.g. the Facebook Graph API node is the correct way to reach the Instagram Graph API, so a valid build using it fails. Assert the capability ("publishes to Instagram via the Graph API, through the Facebook Graph API node or HTTP Request"), not the mechanism. |
| ❌ too loose | "Fetches flight data from somewhere" | A workflow that fetches but never compares passes — doesn't prove change-detection. |
| ✅ right | "Persists the previously-seen status and compares it to the freshly-fetched one" | The defining behaviour; substitution-proof across vendors and storage choices. |
| ✅ right | "Alert is sent only on the change-detected branch, gated by a conditional" | Proves the gate without pinning node or channel. |

Put intent the conversation only *implied* (a preferred but unstated channel) in
`processExpectations`, not `outcomeExpectations`.

## Robust design vs harness flakiness

Two different things — keep them apart:

- **Robust assertion design (always do this).** The agent's unspecified choices
  vary run to run. Source-agnostic `outcomeExpectations` for an unspecified
  source aren't a concession to flakiness — they're the *correct* assertion.
- **Harness limitations (surface them, don't hide them).** Some paths hard-fail
  on a correct build regardless of `dataSetup` — empty resource-locator fields
  that crash Sheets/Drive/Calendar nodes, polling triggers failing to load (see
  "Known harness limitations" above). (State-bearing Data Table *reads* are no
  longer in this bucket — they're pinned from `dataSetup`; only the write path and
  exact-value assertions stay unreliable.) The fix is to *document*, not to *work
  around*: note the limitation in `description` and keep a hard-failing scenario
  out of gated tiers.
  Only when a scenario flips **non-deterministically** run to run is it genuine
  noise worth removing — a scenario that reliably fails for a documented harness
  reason is a standing record of what the harness can't yet test, and stays.

## Push to a lang-tracer suite

Once a case is calibrated, push it (and any others) up into a curated lang-tracer
suite instead of committing the JSON. `eval:langtracer-push` **upserts** over the
REST API: it creates cases missing from the suite, updates ones whose content
drifted, leaves the rest unchanged, and never prunes. It's the inverse of
`--source langtracer` (which pulls a suite down).

```bash
cd packages/@n8n/instance-ai
# preview first — no writes (use `npx dotenvx`; the bare `dotenvx` binary is usually not on PATH):
npx dotenvx run -f .env.eval -- pnpm eval:langtracer-push --suite baseline --dry-run --changed
# then push (drop --dry-run):
npx dotenvx run -f .env.eval -- pnpm eval:langtracer-push --suite baseline --changed
```

- **Selectors** (at least one required — no accidental push-all): positional
  `<slugs...>` (exact file slugs), `--changed` (new/untracked + staged + modified
  `data/workflows/*.json`, ideal right after authoring an uncommitted case),
  `--filter`/`--tier` (with `--exclude` as a modifier).
- **Multiple positional slugs? Skip pnpm — call the script directly.** `pnpm
  eval:langtracer-push … slugA slugB` forwards the slugs as one joined argument
  (`"slugA slugB"`), so no case file matches and nothing is pushed. Either use a
  no-positional selector through pnpm (`--changed`), or run the script directly so
  each slug is its own argv: `npx dotenvx run -f .env.eval -- npx tsx
  evaluations/cli/langtracer-push.ts --suite <slug> <slug1> <slug2> …`.
- **Env:** `LANGTRACER_URL` + `LANGTRACER_API_KEY` (an `lt_` bearer; one key works
  for MCP + REST) — put them in `.env.eval` and run under `npx dotenvx`.
- **Options:** `--set-kind regression|capability_gap` (default `regression`, must
  match the suite's kind), `--contains-user-data` (default is `synthetic`). A case
  whose **build is correct** (outcome expectations green) but that carries a
  **currently-red execution scenario** from a builder bug is still a `regression`
  case — it guards the fix; reserve `capability_gap` for cases where the *build
  itself* is wrong.
- **Scenarios sync on update too:** `PATCH /cases/:id` reconciles
  `executionScenarios` by name (update in place, insert new, delete missing —
  lang-tracer #48), so scenario edits re-push like any other field. A lang-tracer
  deployment predating that change silently ignores the key; if a pushed scenario
  edit doesn't land, update the scenario in the lang-tracer UI.
- **Seeded cases can't be pushed:** the case-write API has no `seed` field, so the
  push lists any seeded case under `skipped:` and it never reaches the suite. A
  `replay` case shouldn't be committed either — it dies when its trace is pruned
  or deleted — so derive a durable synthetic case as the artifact instead. An
  `inline`-seeded case isn't transient and has no suite home, so it's the one
  exception to "don't commit the JSON" — it lives as a committed artifact.

### Link the pushed case to its source (provenance step — always do this)

A sourced case that isn't linked back to the conversation/finding it encodes is
an orphan: six months later nobody can tell what real failure it guards. The
push CLI doesn't carry provenance, so after pushing, link the case over the
lang-tracer MCP with one **`update_test_case`** call on the new case id (the
push prints it):

1. **`sourceThreadId`** — the source conversation's thread id (plus
   **`sourceRunId`** when the case anchors to one specific run/step within it).
   This is the DB-level link every by-version rollup, conversation float, and
   `?sourceThreadId=` query joins on — the tags/description below are the
   human-readable layer on top, not a substitute. The thread must already be
   imported into lang-tracer (running `get_conversation_analysis` on it, as the
   sourcing flow does, is enough); `source_kind` is derived server-side, and
   the link is only editable on authored cases — promotion-recorded provenance
   is immutable.
2. **`expectedBehavior`** — the rule the case enforces, one paragraph — and
   **`failurePattern`** — what actually happened in the source thread, with
   turn references. Copy/adapt these from the analysis's `extractedCases`
   entry when the case came from `get_conversation_analysis`.

Then **`add_case_tags`** (additive; targets the LT-side `tags` array, not
`evalTags`, so nothing round-trips into eval runs): add a capability tag (e.g.
`instruction-persistence`). Tag normalization is aggressive (lowercase, kebab);
colon-form tags get silently dropped. And keep the thread id + turn refs in
the case `description` too (the drafter's habit of "Sourced from thread <id>"
is the convention) — the description is the only field shown everywhere.

## Other eval harnesses (not this skill)

This skill is for `data/workflows/` cases. Three siblings exist with their own
data dirs and CLIs: **`eval:subagent`** (workflow-build compatibility corpus,
binary-check scored), **`eval:discovery`** (asserts first-hop tool/dispatch
routing, no n8n server), **`eval:pairwise`** (head-to-head build comparison vs
`ai-workflow-builder.ee`). Authoring them is out of scope here — see the README
sections of the same names.
