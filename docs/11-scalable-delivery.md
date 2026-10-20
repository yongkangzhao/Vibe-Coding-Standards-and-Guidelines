# Standards 17–19: Scalable Delivery

> A large codebase is not defined only by lines of code. It becomes large when no
> contributor can cheaply predict a change’s ownership, blast radius, verification
> cost, or deployment state.

## Standard 17: Make CI proportional to risk—and fail closed

Running every check for every edit eventually produces long queues, slow feedback,
and pressure to bypass the system. Running only guessed “affected tests” misses
shared contracts and hidden coupling. The answer is a deterministic risk classifier.

### Required properties

- Every change receives a recorded lane and reason.
- Unknown paths use the broad lane.
- Changes to money, identity, authorization, privacy, migrations, shared contracts,
  infrastructure, workflow logic, or the classifier itself use the broad lane.
- A narrow lane is allowed only for explicitly allowlisted paths with exact tests
  proving the generated result.
- One stable aggregate required check reports the lane and outcome so branch
  protection does not depend on a changing list of jobs.
- Fork and untrusted-code isolation remain intact in every lane.

### Efficient execution

Within a lane:

- run cheap static and contract checks before expensive integration shards;
- cancel superseded runs whose revision can no longer merge;
- rebalance shards from measured durations rather than equal file counts;
- preserve complete failure logs while presenting concise summaries;
- rerun only invalidated jobs during diagnosis, then satisfy the required final lane.

### Exact-tree evidence reuse

Do not repeat a full post-merge suite solely because a green PR was squash-merged.
Reuse is safe only when automation proves that the source tree, generated artifacts,
toolchain inputs, and relevant configuration are identical. Deployment-specific
build, migration, rollout, and health checks still run.

If equivalence cannot be proven, rerun. Efficiency must never depend on a human
saying “this is probably the same.”

### Measure before claiming improvement

Retain:

- queue and execution duration;
- selected lane and classifier reason;
- shard and test timings;
- cancellation and retry causes;
- flake classification;
- preview lifetime and teardown result.

Optimize verified time-to-feedback, not the number of tests displayed in a job.

## Standard 18: Ratchet code health instead of demanding a rewrite

Large legacy surfaces cannot be repaired safely by imposing a perfect threshold
overnight. Record a baseline and prevent regression:

- file size and complexity;
- dependency fan-in/fan-out and cycles;
- growth of shared-kernel packages;
- test duration and flake rate;
- suppression count and age;
- unowned modules or undocumented public contracts.

New violations fail. Existing debt begins as visible warnings or an explicit budget.
Each focused refactor lowers the baseline. A suppression requires an owner, reason,
and tracking reference.

The budget is not a license to keep adding debt below the threshold. It is a migration
mechanism that lets ordinary product work continue while architecture improves in
bounded, tested steps.

Prefer decomposition along real capability, transaction, and ownership boundaries.
Do not split files mechanically if the result scatters one invariant across helpers.

## Standard 19: Model delivery and temporary environments as state machines

A preview or release has ordered states:

```text
requested
  → classified
    → configured
      → built
        → deployed
          → verified
            → retired
```

Each transition records:

- source revision and artifact digest;
- generated configuration hashes;
- environment and actor/workflow;
- review and test evidence;
- observed controller and workload state;
- public probe result;
- cleanup result.

Draft, reviewed, selected, live, superseded, and retired configuration are distinct
states. A secret existing does not prove it is selected. A workflow succeeding does
not prove the intended workload serves traffic. A label disappearing does not prove
the namespace, route, data store, identity, and temporary secret were removed.

### Temporary environments

- Create them only for named active verification.
- Bind configuration and evidence to the exact revision.
- Do not let shared staging or production inherit preview-only values.
- Automatically retire superseded, merged, abandoned, and failed previews.
- Verify teardown of compute, namespace, DNS/edge route, data, secrets, IAM, and
  other metered resources.

Cleanup failure is release debt with cost and security impact. It belongs in the
same delivery workflow, not an informal later task.

## The combined effect

Risk-based CI keeps feedback fast without guessing. Ratchets prevent a large codebase
from growing less understandable. Explicit release state prevents green pipelines
from becoming false claims about what is live. Together they let AI-assisted teams
scale output while keeping review and operations grounded in evidence.
