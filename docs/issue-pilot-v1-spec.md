# Issue Pilot v1

- Status: pre-spike architecture candidate. No product code exists. Only slice 0 is authorized;
  provider execution, sandbox containment, repository provisioning, and replay coordination
  remain provisional until their named proofs pass.
- Product: a private, one-owner issue-to-reviewed-draft-PR control room.
- Target host: `issues.nielseriknandal.com`.

## Problem and baseline

The baseline is not "nothing". The owner already runs one Codex or Claude session against a
local checkout and finishes with `gh pr create`. That baseline already gives capture,
execution, and draft-PR creation, and it is free. Issue Pilot must be measured against it.

Over that baseline this product buys exactly four things:

1. **Durable unattended runs.** Work survives a closed laptop, a crashed process, and a
   multi-day pause on an owner decision, instead of dying with a terminal session.
2. **Cross-vendor review that never receives the producer's reasoning transcript.** A second
   vendor re-derives the finding and reviews the exact diff, so agreement is evidence rather
   than an echo.
3. **Artifact-shaped handoff.** The decision inputs are immutable, schema-valid, and
   addressable, instead of being scrollback the owner must re-read.
4. **Capability isolation by construction.** Git authority, GitHub credentials, and lifecycle
   transitions are host-owned, so an injected repository or model cannot reach them. In the
   baseline the only control is the prompt.

Everything else this document describes exists to make those four true and honest. The v1
discovery gate is fourteen of the first twenty started runs reaching Ready without owner action
after Start. Every started run stays in the denominator, including policy-required approvals,
owner stops, defects, and purges, so the measure cannot hide product friction by reclassifying
it. The denominator is successfully reserved `RunRequest` facts, not duplicate HTTP deliveries
or rejected concurrent clicks. Twenty runs are discovery evidence, not a statistical SLO.
Missing the gate means the pipeline is not paying for its stations and the design collapses
toward the baseline.

## Decisions

Issue Pilot is not a ticket tracker and not a Linear clone. It is an execution control room
that answers: what is moving, what needs me, what evidence exists, and what is safe to review.

- Build a distinctive **Runway** view over one fixed pipeline.
- Keep subscription credentials and source execution on one trusted local worker.
- Reuse the exact `provider-runtime[agent-sdks]` revision recorded in `## Primary references`
  rather than reimplement subscription execution. That revision must prove subscription
  enrollment, fresh per-run non-auth state, nested tool-process containment, and the split
  between provider-control and guest-command egress before it is accepted. Existing behavior is
  not assumed to satisfy this contract.
- Keep product facts and durable replay coordination in Postgres. A fixed state machine is the
  durable operation; there is no second workflow store or general workflow graph.
- Let models edit a worktree, but reserve Git, GitHub, credentials, and lifecycle transitions
  for host-owned services.
- Create draft PRs only; the owner retains ambiguity, risk, merge, and deploy authority.

Decision maturity is explicit:

| State | Decisions |
|---|---|
| Accepted | Product boundary, bespoke Runway, fixed cross-vendor lane, Postgres facts, local worker, host-owned effects, exact-SHA readiness, draft-PR-only authority. |
| Provisional until slice 0 | Provider-runtime capability, provider account-state isolation, dedicated-Linux-VM containment, supported-repository profile, Postgres replay kernel. |
| Deferred | Quick lane, arbitrary workflows, multiple workers, hosted execution, multiple users, merge, deploy, and release. |

Eve/Foreman-style role separation and blind review are patterns, not runtime dependencies.
Adding another agent lifecycle or permission system would create a second authority rather
than remove work.

This plan assumes one owner, one trusted worker machine, GitHub-hosted repositories, and
acceptance of Vercel hosting, managed Postgres, a managed OAuth provider, and a private GitHub
App. A second user, hosted model execution, or automatic merge/deploy requires a new threat
model and spec.

## Goals and capability contract

The owner can capture a report in one surface, explicitly start a run, answer a material
question, inspect immutable evidence, and open the final draft PR. An agent can create/read
issues through MCP, but a captured issue never executes without an independently authorized
owner Start action.

The product guarantees:

1. one closed, versioned pipeline; no arbitrary workflow graph;
2. immutable, schema-valid artifacts for every completed station;
3. review without the producing agent's reasoning transcript, with producer-authored fields
   delivered to the reviewer inside a delimited untrusted-content envelope and review criteria
   drawn only from content-hashed `guidance/`;
4. bounded repair and honest `ProofOutcome.Pass`, `Fail`, `Blocked`, and `NotRun` results;
5. readiness bound to one exact local, remote, PR, review, and verification SHA;
6. replay-safe coordination and discoverable external effects;
7. no provider fallback, hidden API billing, force-push, merge, deploy, or branch deletion.

### Final state

- `issues.nielseriknandal.com` is deployed, private to the configured owner subject, and
  connected to its Postgres, OAuth, and private GitHub App resources.
- The local worker runs as an owner service inside its dedicated execution boundary, is
  enrolled with both subscriptions, and is visibly online; each GitHub repository has one
  hosted registration, one worker-local path/command binding, and one owner-created containment
  ruleset pair the App can verify but not alter, plus a passing supported-profile admission
  before its first run.
- Codex and Claude can connect to `/mcp`; an `AGENTS.md` instruction can capture an issue
  without embedding a credential or starting work.
- The owner can move from raw report to an exact-SHA, evidence-complete draft PR and perform
  the only general code review required of them.

## Target behavior

### Pipeline

`Inbox → Investigate → Propose → Build → Verify → Ready`

Reviews are checkpoints inside their station, not duplicate columns.

1. **Capture:** persist the raw report. `Create & run` is explicit; agent-created reports
   always stop in Inbox. Start atomically reserves one `RunRequest` and withdraws any current
   readiness; duplicate delivery converges and a competing Start is rejected. The hosted adapter
   then performs an authoritative GitHub read and a replayable activation mutation creates `Run`
   with a non-null base SHA and derived branch. A crash before activation leaves the request
   claimable by reconciliation, never an ownerless half-run. Before any model session, the host
   requires an exact-base `RepositoryAdmissionReceipt`; a missing/stale receipt queues
   `RepositoryAdmissionJob` and renders in Inbox. `Supported` continues automatically.
   `Unsupported` requires repository setup or an owner stop and never consumes a model call.
2. **Investigate:** one Codex session produces an `IssueBrief` and evidence-backed
   `InvestigationReport` against the fetched, pinned base SHA. Claude inspects the same issue
   and base, receives both artifacts as untrusted content, and verifies the findings without
   receiving Codex's reasoning transcript. Its disagreements are explicit inputs to Propose,
   not an Investigate retry. `IssueBrief` is a separate artifact, not a separate model session.
3. **Propose:** Codex reconciles both investigations into `ChangeProposal`; Claude accepts or
   returns specific blocking findings. A rejection may consume one of the run's two shared repair
   cycles: Codex emits a superseding proposal and Claude reviews it again. No accepted proposal,
   no Build.
4. **Risk gate:** ambiguity or protected capabilities create one `InputRequest`. Auth,
   credentials, destructive data changes, CI/workflows, deploy/IaC, and — when the target is
   Issue Pilot itself — `guidance/`, `contracts/`, `worker/`, and the CI-admission evaluator
   require owner approval before Build or before any unexpected sensitive diff is pushed.
   `.github/workflows` is stricter: the App cannot push it; the owner must publish that
   reviewed commit out of band before the run can continue.
5. **Build:** Codex edits one disposable guest clone with model-command network denied; SCM
   metadata with real authority remains in a separate broker-owned clone.
6. **Verify:** the host commits locally, then runs the registered deterministic checks inside
   the same network-denied, credential-free sandbox tier used for Build, against a throwaway
   copy of the committed tree. The broker holds no SCM credential while any check process is
   alive, and check output is a bounded, schema-parsed receipt. Claude then reviews the exact
   commit diff. Proposal rejection and blocking test/check/code-review failure share one run
   repair budget of two cycles, then require the owner.
   At Propose, a repair cycle gives Codex only the current proposal and blocking
   `ReviewDecision`; it emits a superseding proposal and Claude re-reviews. At Verify, a repair
   cycle gives Codex the accepted `ChangeProposal`, failing `ProofRecord`, blocking `CodeReview`
   findings, and current head diff — never a transcript — then the host recommits, reruns the
   same checks, and requests a new `CodeReview`. Each cycle increments the run-scoped
   `repairCycle` ordinal across both stations. Earlier artifacts are superseded under the
   `Artifact` retention rule. Repair never reopens Investigate, and Verify never silently rewrites
   the accepted proposal; a code finding that invalidates it exhausts the remaining budget
   immediately.
7. **Publish:** the broker pushes fast-forward-only with the expected old ref value supplied in
   the update command, never `--force` and never `--force-with-lease`; the REST refs API
   offers no expected-value precondition and is never used to move a ref. A rejected update is
   owned-branch drift and defects rather than retrying. The GitHub adapter then creates or
   updates one draft PR and observes required checks under a named check-observation deadline.
   On expiry it rereads GitHub once and, if the required-check policy is still unsatisfied,
   appends `CheckObservationDeadlineExpired` and pauses with `ChecksNotObserved`; it does not
   synthesize a `ProofOutcome` or terminal `Run outcome`. A later signed webhook or explicit
   `Recheck GitHub` performs the same authoritative read and may continue the same run.
   Once checks satisfy, the host composes `PrHandoff`, updates the draft PR through the sole
   renderer, and stabilizes the exact rendered-body digest through an authoritative read.
   Automated push is admitted only while the repository's untrusted-PR CI contract remains
   proven; otherwise the run pauses as `HumanPublishRequired` before any remote effect.
8. **Ready:** publish readiness only when all evidence points to the same SHA. A run remains in
   Verify throughout Publish and required-check observation. One serializable mutation appends
   both the `Readiness` publication and the `Completed` run outcome; Ready exists only while that
   publication remains current and satisfies the invariant below.

Changing the PR head, required checks, accepted proposal, or verification after publication
creates a readiness withdrawal. Completion means that the pipeline produced one reviewed
candidate, not that it remains green forever. A post-completion observation may republish Ready
only when the same immutable candidate and evidence satisfy the invariant again; any change that
requires code, proposal, or proof work needs a new owner Start and a new run. A terminal run is
never reopened. It can never leave a stale green row.

### Ready invariant

```text
candidate SHA
= local commit
= remote owned-branch head
= draft PR head
= deterministic ProofRecord SHA
= accepted CodeReview SHA
= required-check observation SHA
```

There must also be an accepted proposal review, no unresolved blocking finding, no unanswered
input request, no base-drift attention, no open `RunDefect` for the run, and no newer active
RunRequest or Run for the issue. Every credential issuance for the run must have a known terminal
revocation/expiry receipt; ambiguity cannot be Ready. The current PR title/body digest must equal
the digest of `GitHubHandoffRenderer(PrHandoff)`. The observed base must equal the latest
`Base pin`; the base is read in the same batched observation as the head, because a head that is
current over a base that moved is not the change that was reviewed.

A dead letter is a defect, not an expected outcome. Failure before activation commits a durable
`RunRequestDefect`; failure after activation commits `RunDefect`, in both cases before the
coordination item is abandoned. Readiness reads the run fact rather than coordination state.
The issue row owns the resulting `DeadLettered` attention, primary `Replay`, and secondary
`Stop run` actions. Replay appends the matching immutable defect resolution only after it has
safely reconstructed or retired the abandoned coordination; it does not wait for the entire run
to finish and never mutates the original defect. An owner stop first fences and retires that
coordination, then appends the resolution and either `RunRequestCancellation` or
`StoppedByOwner`. This is the escape from a deterministic recurring defect and releases the
issue's execution reservation. `/settings` renders only a cross-issue list of these same facts.

## Product design

[`issue-pilot-v1-product-design.md`](issue-pilot-v1-product-design.md) owns Runway information
architecture, interaction, literal copy, visual direction, accessibility, and artifact-content
quality. Its binding product contract is:

- one issue per row across `Inbox → Investigate → Propose → Build → Verify → Ready`;
- lifecycle state is derived and never draggable; Publish remains Verify and Ready requires a
  current exact-SHA readiness publication;
- the board answers repository, current station, actor/activity, elapsed time, required owner
  action, evidence, and PR availability without opening the dossier;
- the dossier is an evidence document—Source, Findings, Proposal, Change set, Proof, Handoff—not
  chat or a transcript;
- owner actions are bounded, explicit, keyboard/screen-reader operable, and never enabled from
  model prose alone;
- graphite, dark-only, dense instrument-panel composition is bespoke rather than a Linear clone;
  semantic text/shape/color and WCAG 2.2 AA are release requirements; and
- every generated artifact and human-input surface has one schema, quality rubric, strong fixture,
  unacceptable fixture, rendering hierarchy, and exhaustive copy owner before engineering.

## System structure

```text
Browser ─┐
MCP ─────┼─> Next.js control plane ─> Postgres facts + replay coordination
GitHub ──┘          │                 fixed run state machine
                    │ closed stage jobs
                    v
              outbound local worker
              ├─ provider-runtime -> Codex / Claude subscriptions
              └─ guest clone + SCM broker clone -> GitHub App -> draft PR
```

### Hosted control plane

- Next.js App Router on the Node runtime; React Server Components for reads, Server Actions
  for browser commands, and Route Handlers only for MCP, worker, OAuth, and GitHub webhooks.
- Node/TypeScript and `pnpm` are the declared primary runtime and package manager. Tailwind
  and source-owned shadcn/Radix primitives provide accessible mechanics; Runway composition
  and visual language are bespoke, not a dashboard template.
- Neon Postgres with Drizzle-owned schema/migrations is the only authoritative persistence and
  coordination system. A run is a closed, domain-owned durable state machine. Its facts are the
  replay state; there is no workflow memo store, mutable `run.status`, arbitrary workflow graph,
  or second queue.
- `activateRunRequest(runRequestId)` is the one hosted pre-run adapter. It rereads GitHub outside
  a transaction, then invokes a replayable activation mutation with that observation and a
  replay-stable candidate Run id. The mutation selects the issue's execution reservation and
  either creates the one Run or returns the already-created identical Run; a conflicting
  observation retries from a fresh read. Exhaustion commits `RunRequestDefect`.
- `ensureRunProgress(runId)` is a convergent replayable database mutation. Under serializable
  isolation it reads the run's durable facts, derives the one legal next transition, and appends
  at most one next-transition fact set: a `RepositoryAdmissionRequest`, `StageRequest`,
  `InputRequest`, terminal outcome, or no-op result. It performs no external call. Stable request
  identity and `StageKey` make duplicate and concurrent invocations converge; an impossible
  competing request commits a `RunDefect` rather than being selected by insertion order.
- After Start, the request path awaits `activateRunRequest`; after worker completion, human input,
  stop, stabilized GitHub observation, or another durable run stimulus commits, it awaits
  `ensureRunProgress` as a separate operation. A one-minute platform-scheduled reconciler scans
  both unactivated RunRequests and active Runs whose derived state permits progress and invokes
  the same respective operation. `justify-polling`: a process can crash after a stimulus commit
  and before the follow-up begins; all reservations and progress prerequisites are canonical
  Postgres facts, so the scan closes that crash gap without a transactional outbox or cross-store
  resume token. One minute is below the shortest external stage deadline. The scheduler route
  requires its dedicated platform secret, accepts no caller-selected issue/run id, scans one
  server-bounded page, and is only a wake signal; it cannot author a transition.
- Delayed retry is a `StageRequest.availableAt` fact interpreted against the database clock.
  Human input is durable because an unanswered `InputRequest` requires no live waiter. Stage
  claims use attempt epochs and retained request identity; server restart cannot lose or invent
  work. Retry exhaustion commits `RunDefect` before the request is abandoned.
- While visible, Runway performs bounded adaptive snapshot revalidation and always rereads
  canonical facts. `justify-polling`: v1 has no push/notification transport for worker and
  GitHub changes, so the named Runway schedule refreshes every two seconds for one minute after
  activity, then every fifteen seconds with jitter; it stops while the tab is hidden or the view
  is unmounted and refreshes immediately on visibility or command completion. Delay can stale the
  view but cannot lose or invent product state.

- The current GA Vercel Workflow runtime is a deliberate non-selection, not an assumed immature
  option. It would supply managed durable steps, retries, hooks, versioning, and run observability.
  Here, however, human answers, GitHub observations, worker completions, defects, and readiness
  are already authoritative queryable Postgres facts. Making the workflow event history
  authoritative would split lifecycle truth; keeping Postgres authoritative would still require
  a database-to-hook outbox and reconciliation. The slice-0 ADR compares both designs against the
  same named crash and concurrency cases. A failure of the standalone Postgres proof reopens this
  decision before slice 1.
- A managed OAuth 2.1 provider is the authorization server for browser and MCP; Descope is the
  v1 default. Only one configured issuer/subject may enter. The control plane serves
  `/.well-known/oauth-protected-resource` (RFC 9728) naming that issuer, answers an
  unauthenticated `/mcp` request with `401` and a `WWW-Authenticate: Bearer resource_metadata=…,
  scope=…` challenge, and a scope failure with `403` and `error="insufficient_scope"`.
  Protected-resource metadata is the only client discovery path in the pinned MCP revision.

### Execution and publication security

The local worker and GitHub App form one security boundary whose mechanisms, threat model,
admission rules, blast radius, and recovery are owned by
[`issue-pilot-v1-security.md`](issue-pilot-v1-security.md). This spec owns only the capability
contract:

- The worker runs under the supported dedicated VM/account profile with the exact pinned
  `provider-runtime` revision. Provider control may reach one vendor and its enrollment; model
  commands, checks, resolver execution, the tree importer, and the SCM broker have separate,
  least-privilege OS capabilities.
- SDK-native configuration isolation disables automatic repository/user rules, hooks, skills,
  memory, trust, MCP discovery, and hosted network tools. Every run receives fresh non-auth state.
  Repository text remains untrusted and may steer output, but cannot widen tools, mounts, egress,
  credentials, Git, GitHub, or lifecycle authority.
- A local mode-`0600` registry owns paths, exact command ids/argv, limits, registries, and
  protected-path policy. Server data cannot provide them. A current live
  `RepositoryAdmissionReceipt.Supported` for the `prebuilt-no-scripts` profile is required before model
  execution.
- Models edit only a disposable guest clone. A credential-free importer applies a bounded regular
  file delta to a separate broker clone after all guest processes exit. Only the broker can mint a
  just-in-time GitHub installation token and publish the exact checked and reviewed SHA.
- The private GitHub App is constrained by minimum permissions, owner-created non-bypassable
  rulesets, metadata-only untrusted-PR automation admission, expected-old-ref fast-forward push,
  and fresh authoritative reads. Candidate code never executes on a hosted runner before owner
  review. The App cannot merge, deploy, release, delete branches, or publish workflow files.
- Installation tokens have GitHub's fixed one-hour expiry. Mint just in time, scope to one
  repository and the minimum permission subset, application-encrypt into crash-recoverable escrow
  before delivery, revoke after use/fencing, and treat failed or ambiguous issuance as exposure
  until recorded or conservative expiry.
- Only bounded artifacts leave the worker for the hosted control plane. Source and tool results
  required for model work necessarily transit to Codex and Claude under their subscription terms;
  raw local transcripts are not hosted. Emergency purge is the sole deletion exception and never
  claims to remove provider, GitHub, or unexpired managed-backup copies.

The capability ledger is provisional until slice 0. Any unavailable or unproven isolation fact
blocks that provider or repository rather than weakening this contract.

## Domain and schemas

All private primary keys are UUIDv7, generated by the application before insert. Ids minted
inside replayable operations come from the operation's replay-stable id primitive, so retry
reaches the same row. Outward identifiers are opaque sealed handles.
`IP-<number>` is a globally monotonic display alias allocated through the shared short-handle
infrastructure in the same transaction as the issue insert; gaps are permitted, it resolves
server-side to a typed issue, and it never authorizes anything. Timestamps use the database
clock. Relations are normalized; no cascade deletes, generic metadata table, mutable status
mirror, or speculative index. Artifact JSON is allowed because it is one closed, versioned
tagged union owned by the artifact subsystem.

`Station` is the closed derived vocabulary `Inbox`, `Investigate`, `Propose`, `Build`,
`Verify`, `Ready`. `Attention` is the closed derived vocabulary `None`, `InputPending`,
`RepositoryAdmissionPending`, `RepositoryAdmissionRejected`, `SensitiveChangePending`,
`ProviderUnavailable`, `CredentialExposurePending`, `RefreshRequired`, `RefreshExhausted`,
`HumanPublishRequired`, `ChecksNotObserved`, `WorkerUnavailable`, `WorkerOutOfDate`,
`RepairExhausted`, `DeadLettered`, `ReadinessWithdrawn`, `ReadyForReview`. Both are derived, never
stored, and owned by one projection that Runway, the left rail, the trays, and `issue_search`
consume exhaustively.

`Stage.kind` is the closed set `Investigate`, `InvestigationReview`, `Propose`,
`ProposalReview`, `Build`, `Check`, `CodeReview`, `Publish`, mapping onto stations as:
`Investigate` and `InvestigationReview` in Investigate; `Propose` and `ProposalReview` in
Propose; `Build` in Build; `Check`, `CodeReview`, and `Publish` in Verify. Ready is derived from
a current `Readiness` publication rather than a stage. `Check` and `Publish` are host-run; the
other six are model sessions. Proof outcomes
are namespaced `ProofOutcome.Pass`, `ProofOutcome.Fail`, `ProofOutcome.Blocked`,
`ProofOutcome.NotRun`, so they never collide with the cell-state words.

Minimum durable facts:

| Owner | Facts and invariant |
|---|---|
| Repository | GitHub `owner/name`, default branch, App installation, required-check policy, the versioned CI admission proof with its fingerprint, and the owner's current external-automation attestation; local paths never hosted. |
| Repository worker binding | Immutable hosted observation that one Worker registry revision binds the RepositoryKey, naming only worker, command ids, profile, and policy fingerprint. A later binding supersedes rather than mutates it. Paths and argv remain local. |
| Repository admission request | Pre-model support state naming repository, exact observed default-branch SHA, worker, contract revision, and provisioning profile. It is claimable only by that repository's bound worker and has one terminal receipt; runs consume its receipt by repository/base identity rather than owning the request. |
| Repository admission receipt | Immutable worker result over request, repository/base, manifest and lockfile hashes, provisioner revision, OS-image identity, registered command ids, profile, and outcomes. `Supported` means dependencies provisioned and every command reached a bounded classified Pass/Fail result under policy; a deterministic failing baseline is evidence, not rejection. `Unsupported` means the command could not execute within the profile. The latest exact-base `Supported` receipt permits the first model stage; changed inputs derive stale admission rather than mutating the receipt. |
| Issue | Repository, raw report, provisional title, source principal, and creation fact. Archive is a separate one-to-one fact. |
| Issue purge tombstone | Content-free record that the owner requested purge and the hosted issue aggregate was removed: sealed issue handle fingerprint, operator, reason, and database time. Local completion is derived from the separate purge dispatch receipt. The tombstone authorizes nothing and retains no report, artifact, repository path, SHA, branch, or PR content. |
| Purge dispatch | Content-free worker coordination created while the issue graph still exists, containing only a purge handle, worker identity, and opaque run-cleanup handles. Its separate completion receipt proves removal of matching delivery-outbox rows, raw transcripts, disposable clones, and scratch. It never contains source, paths, SHAs, artifact bodies, or credentials. |
| Issue context | Immutable owner/MCP submission with body, source principal, host time, and evidence links; not a generated artifact. |
| Run request | The pre-activation reservation: Issue, requester, replay key, exact report hash and ordered `IssueContext` ids visible at Start, plus the `PipelineRevision`, `ContractRevision`, and `GuidanceManifestHash` resolved at request time. It is active while neither a Run nor `RunRequestCancellation` references it. A serializable select-before-insert permits at most one active request or active Run for an issue and withdraws current readiness in the same Start mutation; duplicate Start with the same replay key returns the reservation and a competing Start is rejected. Later context or guidance is for a future run. |
| Run request cancellation | Terminal pre-activation fact naming the request and a closed reason union of direct owner stop or defect-retirement resolution. It is appended only after activation coordination is fenced and any open request defect is resolved. |
| Run request defect | Pre-activation dead letter naming the request, base-observation boundary, and retained retry state. It is open while no immutable resolution references it; replay resolves it only after activation is reconstructed, and stop resolves it only after activation is retired. |
| Run request defect resolution | Immutable fact naming one request defect and the replay/stop operation that reconstructed or retired activation coordination. Recurrence appends a new defect. |
| Run | Activated request, the initial GitHub-observed base SHA, and host-derived `OwnedBranchName` matching `issue-pilot/ip-<number>-<opaque-run-slug>` under `refs/heads/issue-pilot/**`; neither component exposes a private database id. A run is active while no `Run outcome` references it. An open `Run defect` is not an outcome, so a defective run stays active and blocks a new run until replay recovers or owner stop safely retires it. Base/branch are never nullable lifecycle placeholders and are never mutated; a later base comes from `Base pin`. |
| Run stop request | At most one immutable owner command for an active run. After it commits, no new stage or external effect may start. Unclaimed work stops immediately; a claimed local attempt receives a cooperative stop and must settle as `Interrupted`; an already-started uncertain Git/GitHub effect stabilizes before termination. The run outcome is appended only after retained coordination is settled. |
| Base pin | Append-only per run: the initial activation pin plus one row per accepted refresh, each naming the observed SHA and the drift that caused it. The latest row is the run's current base and the SHA the Ready invariant compares against. |
| Run outcome | Exactly one terminal tagged union per run: `Completed` references the `Readiness` publication appended in the same mutation; `StoppedByOwner` names `BeforeFirstStage \| AfterStage(StageKey)`, the run-stop request, and any defect resolution; exhaustion variants name their causing cycle and finding/drift. `ChecksNotObserved` is recoverable attention, not an outcome. A defect is not an outcome; stop resolves its coordination before appending `StoppedByOwner`. |
| Run defect | Committed before any defect path abandons its item, by every producer listed in Failure and safety rules; names the run, the boundary, and the retained coordination state. It is open while no `Run defect resolution` references it. |
| Run defect resolution | Immutable fact naming one defect and the replay/stop operation that safely reconstructed or retired its abandoned coordination. Resolution occurs at recovered ownership, not only when the whole run terminates; recurrence appends a new defect. |
| Stage | Closed `{kind, repairCycle, refreshCycle}` key, both ordinals run-scoped and shared across kinds so the two budgets never collide, exact typed input links to artifacts and host facts including the admission receipt, executor role, and the stage `GuidanceHash` resolved only from the run request's pinned manifest. |
| Stage request | Immutable claimable coordination for one Stage key, with replay-stable identity, `availableAt`, retry policy revision, and one completion or open defect. A retry retains the Stage key and increments only the attempt epoch; a repair or refresh has a new Stage key. |
| Worker attempt | One claim of a closed target union `StageRequest \| RepositoryAdmissionRequest \| PurgeDispatch`, with worker/executor/start fact, current fencing epoch, credential issuances, and at most one terminal completion union: `Succeeded`, `ExpectedFailure`, or `Interrupted`. Only a Stage-target attempt may own an Agent turn. |
| Agent turn | One or more per Stage-target worker attempt. Preallocated call handle, backend/model/runtime revision and usage when supplied, and a typed failure when it failed. No raw reasoning transcript or invented subscription cost. |
| Artifact | Kind, schema revision, immutable payload, SHA-256 content hash, creation time, and a `producer` union of `AgentTurn` or `Host`. An `AgentTurn` producer derives its attempt and stage through that turn; a `Host` producer names the host-run stage directly. `ProofRecord` and `PrHandoff` are always `Host`; either one carrying an `AgentTurn` producer is a defect that fails the run. During normal lifecycle artifacts are never deleted; a later cycle supersedes, never replaces. Emergency whole-issue purge is the sole exception. |
| Human input | Separate immutable request and response rows; one response wins by select-then-insert under serializable isolation. |
| Pull request | Repo/number/URL/branch plus observed head and rendered-handoff digest facts; webhook observations never overwrite history. |
| Check observation | Authoritative required-check policy and results for one candidate SHA. Deadline expiry is a separate observation fact that derives `ChecksNotObserved`; a later observation may satisfy the same active run. |
| Readiness | Append-only per run. A publication references exact proposal review, code review, proof, checks, `PrHandoff`, PR, and candidate SHA. A withdrawal references both the publication it invalidates and the invalidating fact; a publication is current when no withdrawal references it. Restore is a new publication, never a third row kind. Order is the database `created_at` with the row id as deterministic tiebreak. |
| GitHub ingress | Delivery-ID receipt plus authoritative reread receipt; payload order never owns state. |
| Worker | Registration, one-time expiring enrollment allocation, hashed scoped credential verifiers, revocation and expiry facts, reported contract revision, and last claim-poll and last attempt-heartbeat instants on the database clock. The raw bootstrap is shown once. Enrollment creates one application-encrypted credential-delivery row; identical exchange replay returns the same credential until its first authenticated claim consumes the bootstrap and explicitly deletes delivery material. A worker is online while `now - last claim poll` is under the named worker-online TTL. Liveness is never a terminal fact and never cancels or reassigns a stage. |
| Installation credential issuance | Pre-call coordination fact naming attempt, epoch, repository, permission subset, start time, and bounded call deadline. It receives exactly one known-failure receipt, credential lease, or ambiguous-issuance receipt. Ambiguity blocks another claim/effect for that attempt until the conservative deadline plus GitHub's fixed lifetime. |
| Installation credential lease | Private security resource naming attempt, epoch, repository, exact permission subset, issued-at, and GitHub's expiry. A separate one-to-one credential-material row holds only application-AEAD ciphertext and key revision until revocation or expiry; a terminal revocation/expiry receipt is append-only and the material row is then explicitly deleted. The encryption key is deployment secret material. The lease is outside the purgeable issue aggregate so fencing can finish after issue content is gone. |

Every mutation is serializable-equivalent and uses explicit select-then-create/update.
Lifecycle invariants such as at-most-one-active-run live in application code plus defects,
never in database check constraints or unique indexes. No external call occurs inside a
database transaction. Every external mutation names one wrapper: the Git push is an uncertain
transition plus stabilization against an authoritative GitHub read; `ensureDraftPullRequest` is
an idempotent external wrapper, convergent by contract; installation-token issuance is an
uncertain transition with a pre-call fact and modeled ambiguity hold, while known-token
revocation retries until success or expiry. Stabilization's recheck precedes execution; it is not
a trailing read appended to every side effect. Retry and polling schedules are named constants in
one policy catalog under `src/platform/`, as `retries.md` requires. Every artifact is addressable
by its sealed handle from the issue dossier.

## Interface contracts

There is no duplicate public REST CRUD API.

### Browser commands

Server Actions expose one strict owner-only command per product intent:

- `createIssue(repositoryKey, report, SaveToInbox | CreateAndRun)`;
- `startRun(issueHandle)` and `addIssueContext(issueHandle, body, evidenceLinks)`;
- `answerInput(inputHandle, closedChoice)`, `refreshBase(runHandle)`,
  `stopExecution(issueHandle)`, `replayDefect(defectHandle)`, and
  `recheckGitHub(runHandle)` or `restorePrHandoff(runHandle)`;
- `archiveIssue(issueHandle)` and `unarchiveIssue(issueHandle)`;
- `purgeIssue(issueHandle, typedAlias, acknowledgeGitHub, acknowledgeBackups)`;
- `registerRepository(githubOwnerName, appInstallation, providerDisclosureAcknowledged,
  externalAutomationMode)`; and
- `createWorkerEnrollment()`.

Each command accepts framework-owned replay identity, parses handles and drafts at ingress, and
returns one closed result union; delivery replay returns the original result and conflicting
intent is rejected. `CreateAndRun` inserts Issue and RunRequest in one mutation. Sensitive-change
approval/rejection is an `answerInput` choice, not another authority path. Repository paths,
commands, provider enrollment, readiness, and arbitrary lifecycle state are unrepresentable.

### MCP

Expose one stateless Streamable HTTP `/mcp` endpoint using the official TypeScript SDK and
current pinned protocol revision. Validate Origin, OAuth audience, expiry, owner subject,
per-tool scopes, body bounds, and owned schemas.

Initial tools:

- `issue_create(repositoryKey, report)` → inert Inbox issue, handle, URL;
- `issue_get(issueHandle)` → report plus addressable artifact summaries; a purged handle returns
  typed `Purged` with no former content;
- `issue_search(repositoryKey?, station?, attention?, query?, includeArchived?, limit<=20)`,
  where `query` is an unindexed case-insensitive substring match over the raw report and
  current title, and archived issues are excluded unless `includeArchived` is set;
- `issue_add_context(issueHandle, context)` → immutable submitted issue context; a context added
  after Start is excluded from that run's frozen request inputs;
- `run_get(runHandle)`.

MCP can never set station, verdict, attention, SHA, branch, PR, readiness, model, permissions,
or command. Its only grant is `issues:capture issues:read`; start, stop, answer, approve, replay,
purge, and GitHub recheck are browser-owned mutations and are not MCP tools. An `AGENTS.md` may
instruct an agent to call `issue_create`; credentials live in the client's OAuth store, never in
that file. Pipeline Codex/Claude sessions do not receive the Issue Pilot MCP server or any
repository-discovered MCP server. This prevents self-propagating capture or lifecycle control
from inside a run.

### Worker API

The authenticated, versioned surface is narrow:

- `POST /api/worker/v1/enroll` (one-time bootstrap token → scoped worker credential);
- `POST /api/worker/v1/repositories/{repository}:bind` (worker identity, command ids, and local
  policy fingerprint; never path or argv);
- `POST /api/worker/v1/jobs:claim` (one `WorkerJob.v1` or immediate `204`);
- `POST /api/worker/v1/attempts/{attempt}:heartbeat` (continue, stop, or purge-fence directive);
- `POST /api/worker/v1/attempts/{attempt}/events` (closed phase enum, ordinal, and counters only;
  no free text, path, source, prompt, or output);
- `POST /api/worker/v1/attempts/{attempt}:complete` (closed completion union matching its job);
- `POST /api/worker/v1/attempts/{attempt}:fail`;
- `POST /api/worker/v1/scm-credentials:issue` (`Fetch` or exact-SHA `Publish`).

Enrollment resolves the one-time bootstrap by verifier lookup and returns one scoped worker
credential. Identical exchange replay returns the same application-encrypted delivery until the
first authenticated claim consumes the bootstrap and deletes delivery material; conflicting
exchange or later reuse is rejected. `:bind` resolves path/argv only from the authenticated
worker's local registry, reports the non-secret policy fingerprint and command ids, appends a new
binding when they change, and queues exact-base admission. It cannot bind a GitHub identity that
does not match the hosted Repository and its App installation.

`WorkerJob.v1` is the closed union
`StageJob.v1 | RepositoryAdmissionJob.v1 | PurgeJob.v1`; purge work, then admission, has
priority over a new stage. Every variant contains a fenced `WorkerAttemptHandle`. `StageJob.v1`
also contains handles, `RepositoryKey`, `StageKey`, base/head SHAs where applicable, the exact
admission-receipt handle and input artifact handles, the `PipelineRevision`, `ContractRevision`,
and requested `GuidanceHash`. A fenced attempt handle carries a per-attempt epoch that strictly
increases on every claim,
including resume of the same sticky attempt; every attempt-scoped route rejects a stale epoch.
Every epoch increment — reconnect as well as owner reassignment — revokes every installation
token issued to the attempt before the new claim is admitted, because GitHub honors the bearer
token and not our epoch, so a stale worker fenced out of our API could otherwise still push.
GitHub offers no revoke-by-id call — `DELETE /installation/token` revokes only the token
presented — so before returning a token the control plane application-encrypts it into the
dedicated credential-material row for its `InstallationCredentialLease`. The AEAD key remains
in the deployment secret store; plaintext never enters Postgres, logs, or another row. The
material is explicitly deleted after a revocation/expiry receipt. GitHub fixes
installation-token lifetime at one hour; Issue Pilot cannot shorten it. Tokens are minted just
in time for one repository with the narrowest permission subset, revoked immediately after use
or fencing, and treated as exposed until recorded expiry when revocation cannot reach GitHub.
A crash after GitHub issues a token but before escrow commits can leave one untracked token
alive until that fixed expiry; this accepted issuance gap is contained by repository rulesets
and represented by the pre-call issuance fact plus an ambiguous receipt on recovery. Claim
atomically creates or resumes that worker's sticky attempt only after every recoverable prior
lease is revoked or expired and every ambiguous issuance has crossed its conservative expiry.
The job contains no path, command, environment, credential, or raw policy.
Before launch, the worker proves the receipt's OS image, provisioner, command ids, and local
registry policy still match; mismatch returns `RepositoryAdmissionStale`, requests re-admission,
and makes no provider call.

`RepositoryAdmissionJob.v1` contains its attempt handle, the request handle, `RepositoryKey`,
exact base SHA, profile revision, and contract revision. The worker resolves all paths and command policy from
its local registry, fetches that exact base through the SCM broker, runs the supported preflight,
and returns the bounded receipt. Identical completion returns the stored receipt; conflicting
completion is `409`. Repository registration runs the first admission. After Start activates a
Run from a fresh GitHub base read, `ensureRunProgress` queues a new admission when no `Supported`
receipt names that exact base; the run remains in Inbox and no model stage is requested until it
passes. A rejected receipt exposes `View setup` and `Stop run` rather than weakening the profile.

`PurgeJob.v1` contains only its attempt handle, the purge handle, opaque local cleanup handles,
and contract revision. The worker resolves those handles through its own registry/outbox, kills a matching
guest process group, deletes matching delivery payloads, transcripts, disposable clones, and
scratch, then returns a content-free receipt. Cleanup is convergent: a known target already
absent returns `AlreadyAbsent`, while a structurally invalid or never-issued handle is rejected;
the local purge receipt makes repeated completion identical and conflicting completion returns
`409`. Server-supplied paths are unrepresentable. A late stage completion after purge fencing is
rejected and cannot recreate the issue.

An attempt has at most one terminal completion: a repeat `:complete` for an already-terminal
attempt returns the stored result, and a completion whose artifact content hash differs from
the stored one is `409`. Request byte identity is never the dedup key. All commands accept
`Idempotency-Key`; identical replay returns the accepted result, conflicting replay is `409`,
and transport loss never owns cancellation. The worker reports its contract revision on claim;
an unsupported revision returns `WorkerOutOfDate`, which queues the run rather than failing it.

The SCM credential route issues a narrowed, short-lived GitHub installation token only to the
job's bound worker at the current epoch. `Fetch` is valid for Stage or RepositoryAdmission
attempts; `Publish` is valid only for the host-run Publish stage and additionally requires
accepted exact-SHA review.
The SCM broker requests it only after every model and check process has exited, keeps it only
in memory, and prevents credential helpers or child processes from inheriting it.

A normal owner stop commits `RunStopRequest` before signaling the worker. No new provider, Git,
or GitHub effect may begin after that commit. A running local attempt receives `stop` on heartbeat,
is cooperatively terminated, and settles as `Interrupted`; partial output is never promoted to an
artifact. If a Git or GitHub call already crossed its effect boundary, its adapter stabilizes the
unknown result before `StoppedByOwner` is appended. A pre-activation stop instead fences activation
and appends `RunRequestCancellation`. Transport loss cannot turn a stop request into permission to
continue.

Boundary schemas are authored once as strict TypeScript schemas under `contracts/source/`,
emitted to `contracts/generated/` as committed JSON Schema, and consumed by the Python worker
and provider native structured-output support. `pnpm generate:contracts` is the owning
generation command and `pnpm check:static` fails when generated files are missing,
non-canonical, or stale. Generated files are never hand-edited. Untrusted and generated values
are parsed once at ingress, then converted to narrow owned types.

## Failure and safety rules

- Expected outcomes: invalid request, missing local binding, `WorkerOutOfDate`, precise
  subscription login/quota unavailability, pending/rejected repository admission, deterministic
  test failure, review finding, base drift, owner stop, input request, sensitive-change approval,
  `HumanPublishRequired`, `ChecksNotObserved`, bounded ambiguous-credential expiry wait, repair
  exhaustion, and refresh exhaustion.
- Worker offline means queued, not failed. Stop prevents future effects; it does not pretend
  completed or already-started uncertain effects were rolled back. It settles retained
  coordination before appending the terminal outcome.
- Exhausted pre-activation GitHub read/reconciliation produces `RunRequestDefect`. Unknown
  provider failures, schema mismatch, missing expected post-activation GitHub objects, exhausted
  infrastructure retry, abandoned stage claim, exhausted run-progress reconciliation, and
  invariant/projection drift produce `RunDefect`. These lists are exhaustive. Defects appear in
  `Stopped` with cause, primary `Replay`, and secondary `Stop run`; they never auto-clear or
  synthesize success. Successful replay appends a resolution; owner stop appends one only after
  retained coordination is fenced and retired; recurrence appends a new defect.
- Base drift is `RefreshRequired`: a conflict-free host refresh appends a new `Base pin`, still
  requires full re-verification, and re-enters at Verify under a new `refreshCycle`, superseding
  the earlier `ProofRecord` and `CodeReview` without deleting them; a conflict requests human
  input. Refresh has its own budget of two cycles per run, separate from the repair budget; a
  third drift stops the run with `RefreshExhausted`.
- Model/repository content is untrusted. It cannot widen permissions or reach the SCM, OAuth,
  provider-account, worker, or run-coordination capabilities. The mechanisms are SDK-native
  configuration isolation, fresh per-run state, the network-denied credential-free sandbox tier
  covering Build and check execution, the separate credential-free resolver tier whose only
  egress is the registered package registries, the named denial of vendor-hosted network tools,
  and the host-authored `InputRequest` frame — not the assertion itself.
- Emergency purge is a named durable operation, not generic CRUD. Its first replayable mutation
  records owner confirmation, fences the active attempt, prevents new Issue Pilot-authorized
  effects, queues revocation of every outstanding credential lease, and withdraws readiness. Its
  next database mutation explicitly selects and deletes the content-bearing issue
  graph in dependency order with exact-row assertions, inserts the content-free tombstone and
  local purge dispatch, and uses no cascade. The worker cleanup receipt completes Issue
  Pilot-owned deletion; an offline worker leaves a loud `/settings` item. GitHub data is never
  represented as purged: the UI links the existing PR and branch to a separate owner-run incident
  procedure. Outstanding installation-token leases remain only in the separate security ledger
  until revocation or expiry and are shown as exposure, not cleanup success. Backup expiry is
  disclosed rather than called immediate deletion.

## Operations and data handling

- Managed Postgres must provide encrypted point-in-time recovery with a declared retention of at
  most seven days. Before first release and after any backup/storage change, restore the latest
  backup into a disposable environment and prove issue, artifact, readiness, and purge-tombstone
  reads. A configured backup with no recorded restore proof is `NotRun` and blocks release.
- Application logs contain correlation handles, boundary names, result variants, durations, and
  provider/GitHub receipt ids only. They never contain report/context bodies, prompts, artifact
  payloads, diffs, command stdout/stderr, OAuth headers, installation tokens, subscription state,
  or local paths. Bounded check evidence belongs only in `ProofRecord`. Redaction tests inject
  canaries at every ingress and fail on any emitted match.
- Successful worker stages delete provider session/transcript material after artifact
  acknowledgement. Defect diagnostics are application-encrypted under a worker-keychain key,
  retained for at most seven days, never uploaded, and deleted earlier by emergency purge. This
  is the only raw-transcript retention path.
- Long-lived hosted secrets live only in the deployment platform's encrypted environment store;
  worker secrets live only in the dedicated account's OS keychain. Ephemeral credential delivery
  and short-lived GitHub tokens are the only exceptions: their application-AEAD ciphertext is
  durable in dedicated Postgres material rows only until first authenticated use or
  revocation/expiry. Rotation procedures name the GitHub App key and webhook secret,
  credential-escrow key, OAuth client secret, worker verifier, handle-sealing key, database
  credential, worker diagnostic-encryption key, and both provider enrollments. Sealed handles
  carry a key revision and never authorize; rotation signs new handles with the new key while old
  keys remain verify-only so durable URLs do not break. Escrow rotation rewraps active material
  or retains the old key as decrypt-only until no row names it. Rotation is rehearsed in a
  non-production environment before first release.
- Browser mutations enforce same-origin and CSRF protections. MCP, worker, OAuth, and webhook
  routes have owned body bounds and per-principal or per-installation rate limits; rate limiting
  never converts an unknown or rejected mutation into success.
- Browser responses enforce a nonce-based Content Security Policy with `default-src 'self'`,
  `object-src 'none'`, `base-uri 'self'`, `frame-ancestors 'none'`, and the narrow owned
  `connect-src`; no `unsafe-eval`, untrusted HTML, Markdown rendering, or automatic model links.
- `/settings` is the one operational surface for worker liveness, repository admission,
  contract revision, open request/run defects, pending local purges, active or ambiguous
  installation-token exposure, GitHub installation health, and latest backup-restore evidence.
  There is no external alert transport in v1, so durable unattended means work survives absence,
  not that the owner is proactively notified.

## Verification and acceptance

[`issue-pilot-v1-verification.md`](issue-pilot-v1-verification.md) owns the command surface,
red/green/refactor protocol, one proof per ownership boundary, live release evidence, delivery
compatibility, and every product acceptance criterion.

The binding 80/20 shape is:

1. pure units for schemas and derived decisions;
2. real-Postgres integration for every lifecycle, concurrency, replay, and purge boundary;
3. real-runtime component tests for Runway interaction and accessibility;
4. a small set of whole-product E2Es with only external provider/GitHub boundaries controlled;
5. HTTP-level provider and GitHub substitutes in CI, never internal module/database mocks; and
6. live supported-profile provider, sandbox, admission, GitHub, backup-restore, and deployment
   proofs in `pnpm accept:release`.

Pass, Fail, Blocked, and `NotRun` remain distinct. A focused test, fake, direct-host sandbox run,
missing credential, zero-step hosted job, or stale-head result never proves a broader or live
boundary. Only slice 0 may proceed while the architecture status is provisional.

## Files and implementation slices

```text
src/app/                    Next.js entrypoints only
src/product/issues/         issue service, storage, derived views
src/product/runs/           state machine, stages, artifacts, readiness, reconciliation
src/platform/environment.ts declared environment contract; the only reader of process env
src/platform/database/      Postgres client and migrations
src/integrations/github/    App auth, CI admission, effects, webhook reconciliation
src/interfaces/mcp/         MCP transport and tools
src/interfaces/worker/      hosted worker protocol
src/ui/                     Runway, dossier, capture, attention, settings
contracts/source/           hand-authored TypeScript boundary schemas
contracts/generated/        committed JSON Schema, owned by pnpm generate:contracts
guidance/                   revision manifest
guidance/stages/            prompts/skills, rubrics, positive/negative fixtures
worker/                     uv project: Python worker, guest clone, registry/outbox, SCM broker
docs/issue-pilot-v1-spec.md product, lifecycle, schemas, interfaces, slices
docs/issue-pilot-v1-product-design.md Runway, accessibility, copy, content schemas/rubrics
docs/issue-pilot-v1-security.md execution, publication, threat model, incident cleanup
docs/issue-pilot-v1-verification.md proof matrix, commands, release and acceptance
docs/decisions/             slice-0 ADRs and capability-ledger decisions
docs/operations/            restore, rotation, worker enrollment, GitHub incident runbooks
scripts/                    generation, seed, launcher, and acceptance scripts
tests/support/              shared fixtures, factories, launchers
tests/fixtures/             workflow-YAML corpus, hostile trees, recorded signed payloads
tests/contract/             generated-schema and TS/Python round-trip proofs
tests/integration/          database-backed service tests
tests/e2e/                  cross-boundary product journeys
```

`src/product/` owns domain/application services and the ports they require; it never imports a
concrete integration. `src/integrations/` implements those ports and may import only `product`
and `platform`. `interfaces` and `ui` may each import `product` and `platform`, but never one
another or an integration. `app` is the composition root and may import those four peers; no
other layer imports `app`. `platform` imports no product layer, and platform modules do not
import one another. `pnpm check:static` enforces this DAG. `src/product/runs/` owns both run facts
and the fixed transition function; scheduled and request entrypoints invoke that public product
operation rather than writing coordination rows. The retry and schedule catalog is
TypeScript-only under `src/platform/`; `worker/` owns its own catalog, and the two are not shared.
Environment variables are read only in `src/platform/environment.ts`, which states every
variable, whether it is required or optional, and its default.

Implementation order and write ownership are fixed:

| Slice | Exit | Exclusive write ownership |
|---|---|---|
| 0. Capability gate | Both live subscriptions, SDK config isolation, two-run state isolation, command/in-process-tool containment, structured output, Issue Pilot admission plus all four rejection profiles, and the standalone Postgres progress crash/concurrency spike are Pass. Retain the ADR comparing the current GA Vercel Workflow runtime, exact provider pin, capability ledger, environment identity, hostile fixtures, and live acceptance command; discard only unused scaffolding. A failed Postgres progress proof reopens orchestration before slice 1. | `spikes/**`, `docs/decisions/**`, `scripts/accept/provider-*`, plus an explicitly authorized provider-runtime branch outside this repo if the pin must advance. No product source or contract freezes. |
| 1. Contract freeze | One representative job/artifact round-trips TypeScript → generated JSON Schema → Python → completion; guidance manifest/fixtures, environment contract, and a real-Postgres migration smoke proof freeze at revision 1. No product walking skeleton is built in this slice. | `contracts/**`, `guidance/**`, `src/platform/environment.ts`, `tests/contract/**`, and generation commands only. This slice is serial and ends before parallel work. |
| 2. Domain/runtime | Facts, projections, fixed run state machine, admission gating, retry/refresh/repair, readiness, defect resolution, purge ordering, and real-DB proofs pass. | `src/product/**`, `src/platform/database/**`, `src/platform/policy/**`, `tests/integration/product/**`. |
| 3. Hosted interfaces | Composition root, owner auth, MCP discovery/capture/read, worker protocol, browser mutation entrypoints, rate/body/CSRF boundaries, and interface integration proofs pass. | `src/app/**`, `src/interfaces/**`, `tests/integration/interfaces/**`. |
| 4. Worker/provider | Registry, job polling, outbox, admission, sandbox tiers, provider adapters, importer, purge cleanup, and worker-owned tests pass. | `worker/**`, `tests/fixtures/worker/**`, `scripts/accept/worker-*`. |
| 5. SCM/GitHub | Broker Git, token lifecycle, push reconciliation, App admission, webhooks, and fake/live conformance pass. | `src/integrations/github/**`, `tests/integration/github/**`, `tests/fixtures/github/**`, `scripts/accept/github-*`. |
| 6. Product UI | Runway, dossier, capture, attention, settings, content states, accessibility, and component proofs pass. | `src/ui/**`, `tests/component/**`. |
| 7. Composition/release | Whole-product E2Es, CI, backup restore, redaction, deployment, and exact-head release acceptance pass without changing implementation to satisfy the harness. | `tests/support/**`, remaining `tests/fixtures/**`, `tests/e2e/**`, CI definitions, `scripts/accept/release-*`, `docs/operations/**`; nothing under `src/**`, `worker/**`, `contracts/**`, or `guidance/**`. |

Slices 0 and 1 are serial. Slices 2–6 may run in parallel only after revision 1 freezes.
Each writes its own boundary test red before implementation and may not edit another slice's owned
path. A required boundary change stops affected work, returns to the slice-1 contract owner for
one revisioned change, regenerates contracts once, and restarts only the affected proofs. Slice 7
integrates; it never papers over a product defect in test-only code.

## Explicit tradeoffs and non-goals

| Decision | Cost accepted | Why |
|---|---|---|
| Custom fixed Runway | Sacrifices general project planning and drag/drop. | It makes lifecycle truth scannable and invalid transitions impossible. |
| TypeScript control plane + Python worker | Two locked toolchains and a generated-contract tax on every boundary change. | Reusing the tested subscription security kernel is safer than a Node rewrite. Revisit only when it exposes a stable language-neutral daemon. |
| Postgres fixed state machine | Issue Pilot owns a small transition function and one-minute reconciler, giving up workflow-managed durable steps, retries, hooks, code versioning, run traces, and operational UI. | The lifecycle is closed, human waits are facts, and every transition prerequisite is already in Postgres. A workflow-authoritative design would split queryable domain truth; a database-authoritative design would still need a hook outbox and reconciliation. Slice 0 must prove the smaller kernel against the same crash/concurrency cases or reopen this decision before contracts freeze. |
| Managed OAuth 2.1 | One identity vendor and setup flow. | Standards-compliant remote MCP auth is not an appropriate subsystem to invent. |
| Private GitHub App | More setup than local `gh`. | Narrow short-lived credentials and webhooks make exact-head readiness observable and revocable. |
| Per-repository CI admission | Repositories with ordinary PR lint/test/build CI require human publish; automated publication admits only metadata automation with no candidate checkout/execution, credentials, or self-hosted runner. | A read-only token is not containment: candidate code on a networked runner can exfiltrate the private checkout or use ambient capability before review. Local credential-free checks are the v1 proof. |
| Same-repository publication race | GitHub cannot condition push or PR creation on the exact base/config observation, so a trusted owner changing CI in either final read-to-effect gap can trigger newly unsafe code before later detection. | Re-read immediately before both effects and accept the residual only in the one-owner threat model. A second writer requires fork publication or a new spec. |
| Third-party repository automation | V1 machine-admits GitHub Actions, not the behavior or complete inventory of installed Apps, webhooks, external CI, or preview deployers. | Automated publication requires a versioned trusted-owner attestation that none executes code or privileged effects for the relevant events; otherwise publish is human-owned. Another administrator requires fork publication or a new admission design. |
| Fresh GitHub read for Ready | Adds page latency and API usage. | Webhook delivery is not ordered or synchronous enough to be readiness truth. |
| Exact PR handoff | Editing the generated PR title/body withdraws Ready until the canonical handoff is restored or a new run replaces it. | The promised final review surface must not silently diverge from the immutable evidence shown in Issue Pilot. |
| No Eve/Foreman runtime | Gives up packaged general-agent features. | Their lifecycle and permissions would duplicate the product state machine and provider-runtime; only the blind-review pattern is needed. |
| Adaptive worker polling | Up to 20 seconds idle latency, 60 seconds after a transport error, plus the one-minute run-reconciliation scan; and recurring requests. | It avoids an inbound daemon/WebSocket broker while durable state remains server-owned. |
| Structured hosted artifacts | Selected repo facts leave the machine, and a model can place repository source into a free-text field. | They make remote review possible; raw transcripts, credentials, and whole checkout snapshots do not upload to the control plane, and the destination is the owner's own service. |
| Vendor-visible model context | Prompts, selected source, diffs, and tool results needed for Codex and Claude leave the worker and are governed by each subscription's settings and terms. | Cross-vendor implementation/review is the product; registration requires explicit acknowledgement and repositories that cannot disclose source are unsupported. |
| No successful-run transcripts | Deep after-the-fact debugging cannot recover a successful model conversation; defect diagnostics disappear after seven days. | Immutable typed artifacts are the product evidence. Retaining raw reasoning indefinitely creates more privacy and injection risk than operational value. |
| Dedicated local Linux VM | Enrollment, VM lifecycle, and repository setup are less frictionless; direct macOS execution cannot release. | One proven kernel/mount/network/seccomp boundary is safer and cheaper to verify than parallel Linux and Seatbelt profiles, and provider tools cannot inherit the owner's laptop or ambient sockets. |
| Guest clone plus broker clone | More disk and one bounded file-delta import. | Guest Git metadata is disposable and cannot mutate host-owned refs/index. |
| SDK-native configuration isolation | Repository agent configuration is not automatically honored, provider upgrades can change the boundary, and each revision needs a live conformance gate. | Disabling settings, hooks, MCP discovery, rules, and memory at the harness boundary is enforceable; filename hiding cannot prevent arbitrary repository text from influencing a model. |
| Prebuilt-no-scripts repository profile | Repositories that require install scripts, source builds, custom registries, or test-time network are unsupported in v1. | Registration rejects them before model spend rather than executing dependency-authored bytes in the only egress-capable tier. |
| Sensitive-change human gate | Final-PR-only review is not absolute. | Pushing CI/auth/credential/deploy changes can cause effects before final review. |
| One Standard pipeline | Every issue, including a one-line fix, spends six model sessions minimum, ten at the full repair budget, and twelve if both base refreshes are also consumed. Subscription quota is consumed per station, not per difficulty; a handful of runs can exhaust a weekly limit and pause the queue with no fallback. | `IssueBrief` and `InvestigationReport` share the first Codex session; a second lifecycle still doubles proof paths before usage evidence justifies it. Revisit when measured per-run quota makes a Quick lane cheaper than not running at all. |
| One active stage, one worker | Many runs may be active, but only one stage executes at a time; every other run waits with no queue-position surface, and nothing executes while the worker machine is off. | Concurrency multiplies the sandbox and credential surface before one run is proven safe. |
| Emergency aggregate purge | A destructive owner-only operational path weakens absolute append-only retention and needs cross-boundary cleanup proof. | Normal evidence stays immutable, but incident response must remove accidentally captured source or credentials from Issue Pilot-controlled storage. A content-free tombstone preserves that the purge occurred. |
| Accepted duplicate subscription spend | A crash between provider completion and local persistence spends the quota twice for one stage, and the duplicate is discarded, not recovered. | Exactly-once provider invocation is impossible across that crash; a byte-level dedup key would be a fiction over nondeterministic output. |
| Installation-token issuance gap | A crash after GitHub returns a token but before encrypted escrow commits can leave an untracked token live for at most the fixed provider lifetime and hold the attempt for that conservative window. | GitHub has no idempotency key or revoke-by-id. A pre-call fact makes ambiguity visible, rulesets contain refs, and recovery blocks rather than minting overlapping authority. |
| Invariants in application code | At-most-one-active-run and one-response-wins are enforced by serializable select-then-insert plus defects, not by database constraints, so a bug in that code can violate them where a unique index could not. | `database.md` forbids encoding lifecycle invariants in schema; the database owns storage shape, application code owns meaning. |
| Fully derived lifecycle state | `Station` and `Attention` are recomputed from the fact log on every read; there is no status field an owner can correct, so a projection bug is wrong on every row at once and the only remedy is a code change and a redeploy. | A mutable status mirror can disagree with the evidence and render a stale green row, which is the exact failure this product exists to prevent. |
| No notifications | An unattended dead letter or a stalled worker is invisible until the owner opens the app. | An out-of-band channel is a second delivery system and a second credential store. |
| Desktop-first, dark-only | Mobile is review/capture, not full operations; no light theme. | It avoids two complex control-room layouts and two palettes in v1. |
| Subscription execution on both vendors | Both routes are single-user by their terms and are replaced by commercial/API authentication at a second user. | It is the only route that uses the plans the owner already pays for. |

Non-goals: mutable kanban status, custom workflows, Quick mode, labels, assignees, sprints,
estimates, comments/chat, attachments, notifications, analytics, prompt UI, cloud model
execution, API-key fallback, multiple workers, merge/deploy/release, branch cleanup, arbitrary
shell jobs, and multi-user/RBAC support.

## Primary references

- Repository standards: `docs/rules/`, especially `testing.md`, `operation-types.md`,
  `database.md`, `boundaries.md`, `frontend.md`, and `generated-text.md`.
- Owned appendices: [`issue-pilot-v1-product-design.md`](issue-pilot-v1-product-design.md),
  [`issue-pilot-v1-security.md`](issue-pilot-v1-security.md), and
  [`issue-pilot-v1-verification.md`](issue-pilot-v1-verification.md).
- `provider-runtime[agent-sdks]`: `git@github.com:NielsdaWheelz/llm-calling.git` at immutable
  revision `a5d9c8e0c1c851daee0731554e0a4a326d3c2819`. Slice 0 may advance the pin only with a
  recorded capability-ledger diff and the full live gate.
- [OpenAI Codex SDK](https://learn.chatgpt.com/docs/codex-sdk),
  [non-interactive safety](https://learn.chatgpt.com/docs/non-interactive-mode), and
  [CLI authentication](https://learn.chatgpt.com/docs/codex/cli), plus
  [web search](https://learn.chatgpt.com/docs/web-search).
- [Vercel Workflow](https://vercel.com/workflows) and its
  [GA execution model](https://vercel.com/blog/a-new-programming-model-for-durable-execution).
- [Claude headless mode](https://code.claude.com/docs/en/headless),
  [Agent SDK settings](https://code.claude.com/docs/en/agent-sdk/claude-code-features),
  [authentication](https://code.claude.com/docs/en/authentication), and
  [subscription-use boundary](https://code.claude.com/docs/en/legal-and-compliance).
- [Linux seccomp filter limits](https://docs.kernel.org/userspace-api/seccomp_filter.html).
- [MCP Streamable HTTP](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http),
  [authorization](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization),
  and [RFC 9728 protected resource metadata](https://datatracker.ietf.org/doc/html/rfc9728).
- [Descope MCP authorization](https://docs.descope.com/mcp).
- [GitHub App permissions](https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/choosing-permissions-for-a-github-app),
  [installation tokens](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/generating-an-installation-access-token-for-a-github-app),
  and [Actions security hardening](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions).
