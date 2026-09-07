## Relationship to the always-on workflow checks

Every successful build is also graded by ~28 always-on binary checks across 7
dimensions (structure, topology, parameter correctness, intent, AI wiring,
craftsmanship, security) —
[`binaryChecks/checks/`](../../../../../packages/@n8n/instance-ai/evaluations/binaryChecks/checks).
Those are broad and low-visibility. **Writing a targeted expectation for your
specific case is still worth it even when a binary check nominally covers it** —
a named case-level assertion gives far better visibility into *this* behaviour
than one row buried in a 28-check rubric. Don't skip an assertion just because a
generic check exists.

When a scenario fails, the verifier tags a **failure category** (`builder_issue`,
`mock_issue`, `framework_issue`, `verification_failure`, `build_failure`). Treat
it as a **hint, not ground truth** — we've seen a genuine node misconfiguration
tagged `mock_issue`, and a real mock problem tagged as a build error. Open the
HTML report and check the actual execution and the generated workflow before
concluding whether the failure is your case, the build, or the harness.

