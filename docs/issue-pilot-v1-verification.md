# Issue Pilot v1 verification and acceptance

This document owns the v1 test command surface, one proof per ownership boundary, live release
evidence, delivery compatibility, and product acceptance. Product behavior is owned by
[`issue-pilot-v1-spec.md`](issue-pilot-v1-spec.md); security mechanisms are owned by
[`issue-pilot-v1-security.md`](issue-pilot-v1-security.md), and interaction/content contracts by
[`issue-pilot-v1-product-design.md`](issue-pilot-v1-product-design.md).

Testing follows [`rules/testing.md`](rules/testing.md). Every implementation slice starts red,
implements the smallest target behavior that turns green, and refactors without weakening the
observable assertion. A focused proof never stands in for a broader gate.

## Command surface

- `pnpm check:static`: format, lint, type check, generated-file freshness, and Python
  format/lint/strict-type gates.
- `pnpm check:deps`: dependency and known-vulnerability audit.
- `pnpm check:ci`: Issue Pilot's own workflow-configuration admission gate.
- `pnpm test:unit`, `pnpm test:component`, `pnpm test:integration`, and `pnpm test:e2e` are
  independently runnable; unit and integration include the matching worker pytest markers.
- `pnpm test`: unit + component + integration, without E2E.
- `pnpm build`: the sole product build and includes `uv build`.
- `pnpm verify`: static + build + test.
- `pnpm verify:full`: verify + dependency audit + CI admission + E2E, with no subscription or
  GitHub credential.
- `pnpm test:live`: opt-in, subscription-backed, evidence-recorded, and never CI.
- `pnpm accept:release`: full verify plus supported-profile provider, sandbox, repository
  admission, GitHub scratch-repository, backup-restore, and deployment gates.

The live provider gate is outside `verify:full` because hosted CI holds no subscription
credential. A missing credential, tenant, scratch repository, supported worker profile, or
backup resource reports `NotRun`, never Pass. Every required `NotRun` blocks release.

CI orders static → unit → component/integration → E2E and uploads failure artifacts, browser
traces, coordination diagnostics, and exact-SHA proof. Tests do not mock internal modules or the
database and do not intercept browser API calls.

External boundaries and CI substitutes are exhaustive:

- GitHub is an HTTP-level fake.
- `provider-runtime` is substituted only at the worker's single provider adapter with scripted
  outcomes.
- Postgres and the fixed run coordinator are real.
- Descope is a real test tenant.

Nothing else is substituted. A fake or adapter substitute is never accepted as the corresponding
live release proof.

## One proof per ownership boundary

| Boundary | Primary proof |
|---|---|
| Schemas/domain | Pure units for ingress and derived projections; real-Postgres integration for idempotency, concurrency, readiness, exact-SHA invalidation, purge, and forward migrations. |
| Run coordination | Real Postgres with the server killed after Start reservation, before/after activation, and after each durable run stimulus but before/after `ensureRunProgress`; restart the reconciler and assert one legal activation/request, stable semantic identity, duplicate-delivery convergence, competing-Start rejection, delayed retry, input pause/answer, bounded repair/refresh, loud request/run defects, durable stop with no later effect, stop retirement, and explicit resolution. |
| Worker protocol | Real control-plane/worker contract for enrollment replay/consumption, path-free repository binding, epoch fencing, just-in-time token revocation, reconnect, outbox resend, identical/conflicting completion, purge, and `WorkerOutOfDate`; a stale worker's push is rejected by GitHub, not merely by our API. |
| Repository provisioning | Supported-profile live preflight for Issue Pilot, an admitted deterministic baseline-test failure, plus committed rejection fixtures requiring install scripts, source builds, redirected registries, and test-time network; no provider call occurs before a current `Supported` receipt. |
| CI admission | Table-driven units over committed workflow/ruleset fixtures covering owned-namespace push, `create`, every privileged trigger, empty token permissions, candidate checkout/`run`/local-action/container execution, dynamic action/ref/input selection, secrets/OIDC, action allowlist/pins, reusable workflows, runner labels and registration, required workflows/check satisfiability, both containment rulesets, and owner-attested external automation. Unknown syntax or candidate/external execution denies automated publish. |
| Delta importer | Hostile real-tree fixtures for traversal, symlinks, hardlinks, special files/modes, size/depth bounds, deletes, submodules, `.git`, dirty source, protected agent configuration, and sensitive build-tooling changes. |
| Sandbox | The production spawn API runs hostile commands and check execution against credential/file canaries and callback sinks on every allowed/denied route; provider in-process file tools target the same canaries. One callback or readable canary fails release. |
| Provider | Live Codex and Claude under the supported profile: subscription auth and structured output succeed; root/nested repository config, hooks, skills, MCP, trust, and memory do not load; run two sees no run-one state; provider egress works while local and hosted network tools remain denied. Model behavior is never the assertion. |
| Git | Real temporary repository and bare remote for untouched source, broker isolation, expected-old-ref fast-forward push, rejected drift, crash reconciliation, exact SHA, and hook/attribute neutralization. |
| GitHub App | Live malicious scratch installation for repository/permission scoping, fixed expiry and revocation, encrypted lease recovery, ambiguous-issuance hold, ruleset rejection of default-branch update and owned-ref rewrite/delete, draft-only PR creation, pre-push/pre-PR admission rereads, observed-gap refusal, inert generated-title/body text, workflow-file refusal, webhook duplication/reordering, and Ready withdrawal. An owner credential—not the App—mutates fixtures and is revoked at teardown. |
| GitHub adapter | One conformance body runs against the HTTP fake in full verify and live scratch repository in release acceptance; divergence blocks release. |
| OAuth/MCP | RFC 9728 discovery, `401`/`403` challenges, audience/origin/expiry/scope negatives, body/rate limits, and proof that MCP exposes no lifecycle mutation. |
| UI | Real-runtime component accessibility for Runway grid navigation, capture, attention, dossier, Ready semantics, purge confirmation, and honest empty/loading/failure states. |
| Purge | Real Postgres plus real worker: active-attempt fencing, hosted deletion without cascade, `410 Purged`, late-completion rejection, outbox/transcript/clone/scratch cleanup, offline pending state, content-free tombstone, and honest GitHub/backup exclusions. |
| Operations | Disposable backup restore, structured-log canaries, successful-session deletion and seven-day defect-quarantine expiry, secret-rotation runbook checks, route/CSRF/CSP security negatives, and `/settings` health projection. |
| Composition | Real-stack E2Es with provider/GitHub controlled: create-to-Ready, input pause/answer-once, late-check recovery, MCP capture-is-inert, and emergency purge. |

## Slice 0 capability gate

Only slice 0 is authorized by the provisional architecture. It passes only when all of these have
exact evidence:

1. the pinned provider repository/revision resolves reproducibly;
2. both real subscriptions return schema-valid structured output inside the supported profile;
3. repository/user configuration, hooks, MCP, trust, memory, and cross-run state are absent;
4. provider control traffic works while model commands, checks, and hosted network tools cannot
   reach callback sinks or canary credentials;
5. Issue Pilot and a deterministic baseline-test failure pass `prebuilt-no-scripts` admission,
   while each unsupported fixture fails before a model call;
6. the standalone Postgres progress spike converges across concurrent calls and every named
   crash point;
7. an ADR compares that result with the current GA Vercel Workflow runtime and reopens
   orchestration if the Postgres proof fails; and
8. the capability ledger, environment identity, commands, timestamps, and retained artifacts
   record the result.

Disposable scaffolding may be removed. Hostile fixtures and the live acceptance command become
permanent release assets. Any Fail or required `NotRun` leaves the architecture provisional.

## Delivery contract

The control plane and worker version independently. `WorkerJob.v1` carries the contract revision;
the worker accepts a closed set and reports `WorkerOutOfDate` otherwise. Contract revisions are
added, never changed in place.

Deploy in this order: additive migration, control plane, worker. Migrations remain compatible
with the previous release; rollback redeploys the previous control plane. The deployment gate
proves worker claim reachability, `/mcp` initialization and OAuth discovery, running/required
worker revisions, current backup-restore evidence, and one scratch-repository run to Ready.

## Product acceptance

Every criterion names an owning proof above. A criterion without an owning proof is unproven and
blocks release.

### Capture and control

- `Create & run` closes capture and presents the new Runway row without manual reload.
- Agent capture lands inert in Inbox: no job, branch, model call, or PR.
- Duplicate Start delivery returns one active reservation and a competing Start is rejected; a
  crash before base activation is reconciled, while exhausted activation becomes a visible
  request defect that Replay or Stop can retire without creating a second run.
- Start freezes the report hash and ordered context ids; context added later is visible on the
  issue but absent from every current-run stage input and labeled for the next run.
- Start freezes pipeline, contract, and guidance-manifest revisions; every stage hash derives from
  that manifest even if repository guidance changes mid-run.
- Create & run requires a registered repository and worker binding. It activates against a fresh
  base read, remains in Inbox while exact-base admission runs, and performs no model call before
  a `Supported` receipt.
- Repository registration records explicit provider-data acknowledgement and a versioned external
  automation attestation. Unknown or privileged third-party execution forces human publish.
- MCP supports capture/read only; it cannot start, stop, answer, approve, replay, purge, or
  recheck.
- The first twenty started runs expose the discovery numerator and denominator; fourteen must
  reach Ready without owner action after Start before the design expands. Every Start counts,
  including a later approval, stop, defect, or purge; idempotent delivery replay does not create a
  second reservation or count. `/settings` derives and renders the same count from immutable
  facts; it is the sole v1 product metric, not a general analytics system.

### Runway and content

- Runway exposes repository, station, activity, attention, elapsed time, and PR availability
  without opening the dossier.
- Every completed station links its immutable artifact.
- An open input appears in Needs you as the row's sole action; exactly one answer advances once
  and a second submission is rejected.
- Input copy shows one question, reason, bounded choices/consequences, recommendation, and
  recovery without requiring a transcript.
- Stopped work shows cause and last valid artifact. Expected terminal outcomes expose exactly one
  permitted next action; a defect exposes primary Replay and secondary Stop run because permanent
  replay failure must have an escape.
- Archive is rejected while a run request or run is active; a terminal or never-started issue
  archives inertly and disappears from default views/search.
- Capture, answer, evidence inspection, purge confirmation, and handoff are keyboard and
  screen-reader usable.

### Pipeline

- The Standard happy path consumes exactly six model sessions; the first Codex session emits
  separate `IssueBrief` and `InvestigationReport` artifacts.
- Investigation and code reviewers receive the issue/base, accepted typed artifacts, exact diff,
  and content-hashed review guidance, but no producing AgentTurn transcript or reasoning state.
- Proposal rejection re-enters Propose only through the run's shared repair counter; Propose and
  Verify together consume at most two repair cycles and ten total model sessions. Repair
  exhaustion makes no further model call.
- Conflict-free base drift re-verifies; conflict requests one input; a third drift stops.
- A successful replay resolves its defect when abandoned coordination is reconstructed; a
  recurrence appends a new defect.
- Worker disconnect, quota, failure, skipped proof, stale SHA, unanswered input, or open defect
  cannot render Ready or green; neither can a known or ambiguous unretired credential issuance.
- A stale-epoch completion is rejected, and a stale worker cannot push with an unrevoked token.
- Stop delivery replay converges on one `RunStopRequest`; no new stage or external effect starts
  after it, a running local attempt settles without promoting partial output, and an uncertain
  GitHub effect stabilizes before the terminal outcome.

### Repository and sandbox

- Issue Pilot and a normal deterministic baseline-test failure admit; install-script,
  source-build, redirected-registry, and test-network fixtures reject before provider execution.
- Repository settings, root/nested instructions, hooks, skills, MCP, trust, and memory do not
  auto-load through either provider.
- A model cannot change protected agent configuration through the imported delta.
- The same hostile fixture passes only inside the dedicated Linux VM release profile; direct
  macOS/host evidence is not equivalent.
- Commands and provider in-process tools cannot read enrollment/canary files, use network, push,
  or mutate an unregistered repository while the provider call succeeds.

### Publication and Ready

- Sensitive changes and `.github/workflows` block automated push with no remote effect.
- Admission rejects owned-namespace push, `create`, every privileged trigger, reachable
  secrets/OIDC, any candidate checkout/command/local action/container execution, reusable
  workflows, self-hosted runners, unsatisfiable required checks, unknown syntax, and missing
  containment rulesets. Read-only permissions alone never admit candidate execution.
- Publish and required-check observation remain in Verify. Deadline expiry exposes Recheck; a
  later authoritative passing observation advances the same run.
- Ready links to a draft PR whose current head equals local commit, owned remote ref, deterministic
  proof, accepted code review, and required-check observation over the latest pinned base.
- Ready requires an addressable host-authored `PrHandoff` and an authoritative PR title/body
  digest equal to its inert `GitHubHandoffRenderer` output; owner/body drift withdraws readiness
  and only explicit Restore handoff overwrites it.
- Duplicate, delayed, and reordered webhooks cannot write truth. Later head/base/check/rule drift
  withdraws Ready without reopening the terminal run. An authoritative later read may republish
  only the same exact candidate/evidence; anything requiring code or proof starts a new run.

### Purge and operations

- Emergency purge fences the active attempt, immediately removes the hosted aggregate, returns
  typed `Purged` without former content, and rejects late completion.
- Pending local cleanup remains visible until the worker proves outbox, transcript, clone, and
  scratch deletion. The tombstone retains no product content.
- Known and ambiguous installation-token exposure survives aggregate purge in the content-free
  security ledger until revocation or conservative expiry and is visible in Settings.
- Confirmation and completion never claim GitHub data or unexpired encrypted backups were erased.
- Logs contain no report, prompt, artifact, diff, path, command output, or credential canary.
- A release cannot pass without current restore evidence and every required live boundary proof.
