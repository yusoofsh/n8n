## Workflow-Level Error Workflows

Error workflows are per-target-workflow (`settings.errorWorkflow` must be the
real workflow ID of a separate **published** workflow with an active Error
Trigger — never a name, placeholder, `activeVersionId`, or local SDK id).
n8n has no global error workflow setting; mention that only if the user asks
about global behavior. Before building or attaching an error workflow, load
this skill's `references/error-workflows.md` linked file and follow its
build → publish → assign steps. Do not create one before the user opts in.

## Compositional Workflows

Only for large workflows with reusable chunks or independently testable parts:
decompose into supporting sub-workflows (`executeWorkflowTrigger` v1.1 with an
explicit input schema, built with `isSupportingWorkflow: true`) referenced from
the main workflow's `executeWorkflow` node (`source: 'database'`, real returned
`workflowId`), main workflow saved last. This is part of the approved build
task — not a reason to create a new plan, and simple
workflows stay in one workflow. Before writing multi-workflow code, load this
skill's `references/compositional-workflows.md` linked file for the required
steps and SDK examples.

## Workflow Rules

Follow these rules strictly when generating workflows:

1. Always use `newCredential()` for authentication. Never use placeholder
   strings, fake API keys, hardcoded auth values, invented credential IDs, or
   raw `mock-*` IDs.
2. Zero items end the branch — downstream nodes do not run. Trust this default;
   do not add `alwaysOutputData: true` or empty-check IF gates unless rule 4's
   mandatory-outcome case applies.
3. Use `executeOnce: true` for a node that receives many items but should run
   once, such as a summary notification, report generation, shared-context
   fetch, or API call that does not vary per input item. Duplicate
   notifications or repeated shared-context fetches usually mean this is
   missing.
4. Pick the right control-flow primitive:
   - Per-item loop with side effects: `splitInBatches` with `batchSize: 1`,
     feeding the per-item work and looping back via `nextBatch`.
   - Drop items that do not match a predicate: `filter`.
   - Two mutually exclusive paths that both do real work: IF with `.onTrue()`
     and `.onFalse()` wired on the workflow builder — never as standalone
     statements on the IF node variable.
   - Many mutually exclusive paths keyed off a value: Switch with
     `.onCase(index, target)`.
   - Mandatory outcome when upstream can be empty (digest/alert must still send):
     set `alwaysOutputData: true` on every node that can emit zero items before
     the effect — often both the HTTP fetch (empty `[]`) and the filter (all rows
     dropped). Not on the formatter or notifier; consumers that receive zero
     items never run. `alwaysOutputData` delivers an empty result as one item
     with empty json (`{}`), not zero items — a downstream formatter or Code
     node must treat empty-json items as zero rows (e.g. `const rows =
     $input.all().filter(i => Object.keys(i.json).length > 0)`) before counting
     or listing them.
   - A Filter or IF only selects items; it does not perform the requested side
     effect. If the user asks to archive, update, delete, send, or create only
     matching items, wire the corresponding action node on the matching path.
5. Input and output indices are zero-based. `.input(0)` and `.output(0)` are the
   first input and output. `.input(1)` is the second input, not the first.
6. When Code nodes score, classify, or gate on free-text human fields
   (amounts, timeframes, priorities, intent), normalize before comparing —
   humans write "≈ $12,500", "1.5k", "in three weeks", "ASAP". Strip currency
   symbols/separators before parsing numbers, take the lower bound of ranges,
   match time units broadly (day/days, week/weeks…), and give every classifier
   an explicit fallback bucket — a one-phrasing regex silently misroutes every
   other phrasing.

## Node Configuration Safety Rules

- Fetch `nodes(action="type-definition")` before configuring nodes. Generated
  definitions and `@builderHint` annotations are the source of truth.
- Use live `nodes(action="explore-resources")` for resource locator, list, and
  model fields when credentials are available.
- If a configuration is unclear after reading the definition, ask for
  clarification or use placeholders. Do not guess.
- Pay attention to `@builderHint` annotations in search results and type
  definitions. They contain node-specific configuration rules and examples.
- Gmail archive: the message resource has no `archive` operation. To archive a
  Gmail message, remove the `INBOX` label with `operation: 'removeLabels'` and
  `labelIds: ['INBOX']`; do not add an invented `ARCHIVE` label.
