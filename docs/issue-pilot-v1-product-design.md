# Issue Pilot v1 product design

This document owns the Runway information architecture, interaction behavior, visual direction,
accessibility, literal product copy, artifact content schemas, and content-quality rubrics.
Product lifecycle and architecture are owned by
[`issue-pilot-v1-spec.md`](issue-pilot-v1-spec.md); acceptance is owned by
[`issue-pilot-v1-verification.md`](issue-pilot-v1-verification.md).

## Runway UX

Runway is a dense semantic grid: one issue per row and one station per cell. Completed cells
form an evidence rail; the active cell shows actor, state, and elapsed time. State is derived
and never draggable.

Stations are exactly `Inbox`, `Investigate`, `Propose`, `Build`, `Verify`, `Ready`. `Risk gate`
renders in the station it blocks, `Publish` renders in `Verify`, and `Ready` is rendered only
from a current `Readiness` publication. Every cell
renders exactly one of `NotRun`, `Requested`, `Working`, `Reviewing`, `Repairing`, `Proven`,
`Blocked`, `Failed`, `Withdrawn`, each with its glyph, its word, and its link to the artifact
or to the invalidating fact. A `Proven` cell that consumed repair cycles renders the count and
is never an unqualified pass. `Waiting for worker` and `Checking GitHub` are freshness overlays
that qualify a cell's state rather than replace it; both are derived at read time from the
worker-liveness TTL and the GitHub read, and neither is a tenth state.

- Above the runway: bounded five-row `Needs you`, `Stopped`, and `Ready` summary trays with
  `View all`; left-rail views are the exhaustive lists.
- Left rail: All, Inbox, Needs you, Stopped, Ready, then repository filters. Every tray's
  `View all` has a destination.
- Global capture (`C`): repository plus one required Report field; primary `Create & run`,
  secondary `Save to Inbox`. `Create & run` requires a registered repository and worker binding;
  exact-base admission may then keep the authorized run in Inbox before its first model call.
- Dossier route: Source, Findings, Proposal, Change set, Proof, Handoff. It is an evidence
  document, not chat. A right rail shows Now, Next, Why stopped, worker availability, exact
  SHA, and PR.
- Attention surface: an open `InputRequest` pins above the dossier tabs and is the row's only
  enabled action on Runway. It renders the one question, why blocked, what was already
  attempted, each bounded choice with its consequence, and the recommendation. Answering is
  one choice plus `Submit answer`; the control disables and relabels `Already answered` once a
  response row exists. `A` opens the oldest open request from anywhere.
- URLs own repository, filters, selected issue, and selected artifact. `/` always renders the
  unfiltered Runway ordered attention first, then activity, then age. View and filter state
  are always explicit in the URL and never vary with data.
- Before `IssueBrief`, the first report line is a provisional label; `IP-<number>` remains
  stable identity.
- Routes: `/` Runway, `/repositories/:key`, `/issues/:handle`, and `/settings`. `/settings`
  owns worker enrollment, repository registration, the one-time worker token, contract
  revisions, the fixed first-twenty discovery counter, Dead letters, installation-token exposure,
  and incomplete local purge cleanup.
- A claimed stage whose last heartbeat is older than the named worker-liveness TTL reads
  `Waiting for worker` with `Execution unavailable`; expiry is derived at read time and never
  continues to look actively `Working`.
- An archived issue runs nothing and is excluded from Runway, every left-rail view, the trays,
  and `issue_search` unless explicitly requested. Archive is enabled only when no active
  `RunRequest` or `Run` exists; active work must reach a terminal outcome first.
- `Purge Issue Pilot data` exists only in the dossier danger zone for the owner. Confirmation
  requires typing the issue alias and acknowledging that GitHub PR/commit data and encrypted
  backups are separate systems. Before deletion it renders the current PR/branch links and the
  GitHub incident runbook, because the tombstone retains none of them. Purge fences the active
  attempt, withdraws readiness, removes
  the issue aggregate from hosted product reads, queues content-free local cleanup, and leaves a
  tombstone. It is absent from Runway row actions, MCP, and bulk operations.

Runway is an ARIA grid: one tab stop, roving `tabindex`, arrow keys across cells and rows,
`Enter` opens the focused cell's artifact, `Escape` returns focus to the row; semantic table
markup inside it. Single-key bindings are inert while a text input, textarea, or dialog has
focus.

Visual direction: graphite instrument panel, strong grid, Geist Sans/Mono, electric cyan for
interaction, and green/amber/red only for proof/attention/stop. The product renders one dark
theme; there is no light mode in v1. No gradients, glass, card piles, or decorative charts.
Desktop is primary; narrow screens render each row as an ordered station summary. Density
geometry is fixed: 4px base unit, 40px rows, hairline borders only, and `tabular-nums` for
elapsed time, cycles, and SHAs.

WCAG 2.2 AA is the accessibility target: 4.5:1 text, 3:1 non-text for state glyphs, borders,
and focus ring, 24×24 with 24px spacing for in-row cell targets, and 44×44 for `Create & run`,
`Save to Inbox`, `Submit answer`, `Stop at safe boundary`, and `Open draft PR`; the whole row
is the target for opening the dossier. Every state carries text plus shape plus color, focus
is always visible, motion respects the reduced-motion preference, and announcements are polite
and limited to attention and terminal transitions.

Required copy is literal, honest, and declared. No undeclared sentence ships; the station-cell
words and the two freshness overlays above are the only other user-visible labels.

Run-attention copy is owned by one `runAttentionMessage` helper with exactly one entry per
`Attention` variant, so adding a variant fails the type check until its copy exists:

- `None`: no message.
- `InputPending`: “Answer needed before {station} can continue.” / `Answer`
- `RepositoryAdmissionPending`: “Checking whether {repository} can run safely at {sha}…”
- `RepositoryAdmissionRejected`: “{repository} cannot run in the v1 profile: {reason}.” / `View setup` / `Stop run`
- `SensitiveChangePending`: “This change touches {area}. Approve before it is built or pushed.” /
  `Approve` / `Stop run`
- `ProviderUnavailable`: “{provider} is unavailable until {time}. This run is paused; no fallback was used.”
- `CredentialExposurePending`: “A GitHub credential may still be valid until {time}. This run
  will resume only after revocation or expiry.”
- `RefreshRequired`, no conflict: “The base branch moved. Re-verification is required.” / `Refresh base`
- `RefreshRequired`, conflict: “The base branch moved and the change no longer applies.” / `Answer`
- `RefreshExhausted`: “The base moved three times during this run. No PR is ready.” / `Start new run`
- `HumanPublishRequired`, admission failed: “{repository} cannot be pushed automatically. Publish
  the reviewed commit yourself, then this run continues.” / `Copy commit`
- `HumanPublishRequired`, workflow file: “This change edits `.github/workflows`, which this App
  cannot push. Publish it yourself, then this run continues.” / `Copy commit`
- `ChecksNotObserved`: “Required checks did not report before the deadline. No PR is ready.” /
  `Recheck GitHub` / `Stop run`
- `WorkerUnavailable`: “Execution unavailable. The worker stopped reporting mid-stage.”
- `WorkerOutOfDate`: “Worker is on {running}; this run needs {required}. Update the worker.”
- `RepairExhausted`: “Two repair cycles did not clear {station}. No PR is ready.” / `Start new run`
- `DeadLettered`: “Stopped by a defect in {station}. It will not retry on its own.” / `Replay` / `Stop run`
- `ReadinessWithdrawn`: “{what changed} after publication. This is no longer ready.” /
  `Recheck GitHub` for observation-only invalidation, `Restore handoff` for PR title/body drift,
  or `Start new run` when code/proof must change; the invalidation kind is closed and exhaustive
- `ReadyForReview`: “Ready for your review.” / `Open draft PR`

Surface copy is owned by the component that renders it:

- empty: “Nothing in motion. Capture an issue or start one from Inbox.”
- filtered empty: “No issues match these filters.” / `Clear filters`
- loading: “Loading Runway…”; load failure: “Runway could not load.” / `Retry`
- worker offline: “Worker offline. Runs are queued; nothing will execute until it reconnects.”
- missing binding: “{repository} has no worker binding. Nothing will execute.”
- provider disclosure: “Codex and Claude receive the source, diffs, and tool results needed for
  this workflow under your subscription settings. Issue Pilot does not upload raw transcripts to
  its control plane.” / `I understand`
- external automation attestation: “Automated publish supports GitHub Actions only. Confirm that
  no installed App, webhook, external CI, or preview deployer executes code or privileged effects
  when this repository receives an Issue Pilot branch or draft PR.” / `No external executor` /
  `Require human publish`
- admission stale at Start: “{repository} moved before Start. The new base will be checked before any model call.”
- resolving base: “Reading {repository} to pin a base commit…”
- input answered: “Answered {time}. Resuming {station}.”
- context added during a run: “Context saved for the next run. This run's inputs are frozen.”
- run stopped by owner: “Run stopped in {station}. No PR is ready.” / `Start new run`
- stop confirmation: `Stop at safe boundary` / “Start no new step or external effect. A running
  local process will stop; an in-flight GitHub effect will be checked before the run stops. Work
  already pushed — commits, branch, and draft PR — stays as it is.”
- one-time worker token: “Copy this token now. It is not shown again.”
- archived: “Archived. It runs nothing and is hidden from every view.” / `Unarchive`
- archive blocked: “Stop this run before archiving the issue.” / `Stop run`
- purge title: “Permanently purge Issue Pilot data”
- purge warning: “This removes the report, generated artifacts, and local run data. It does not
  remove GitHub commits or pull requests, and encrypted backups expire on their retention
  schedule.” / `Open GitHub cleanup` / `Purge Issue Pilot data`
- purge pending: “Hosted data was removed. Local cleanup will finish when {worker} connects.”
- token exposure: “A GitHub token may remain valid until {time}. Rulesets still limit it to Issue Pilot branches.”
- discovery: “Unattended Ready: {ready} of {started}/20. Every Start counts.”


## Content design contracts

Every feature slice has a named content owner — the same person, wearing that hat before
engineering starts. That owner writes the schema, rubric, strong fixture, unacceptable
fixture, rendering hierarchy, empty/loading/error/action copy, and review criteria. This is
design-time ownership, not another runtime model pass.

Host-observed facts—SHAs, commands, paths, timestamps, links, and test results—are injected by
the host, never invented by a model. Valid accepted prose is preserved. Model- and
repository-authored text renders as plain text everywhere in the UI — no Markdown, HTML,
image, or auto-linking — and evidence links are host-validated to `https:` on the registered
repository's host or `github.com`, rendered with an explicit external-link treatment and
`rel="noopener noreferrer"`. They permit owner navigation, never application fetch or model
egress.

| Artifact | Required shape | Good means |
|---|---|---|
| `IssueContext` (submitted fact, not an `Artifact`; it has no producer) | body, source principal, host time, evidence links | Additive and attributable; never overwrites the report or asserts lifecycle state. Render under Source. |
| `IssueBrief` | title, problem, observed/expected behavior, impact, evidence, constraints, unknowns | Faithful and factual; no invented reproduction or premature solution. |
| `InvestigationReport` | base SHA, reproduction, findings, evidence refs, causal explanation, affected paths, rejected hypotheses, unknowns | Observation and inference are distinct; every material claim is falsifiable and linked. |
| `ReviewDecision` | verdict, criteria coverage, findings, residual risk | No generic approval; each blocker states criterion, evidence, consequence, and correction. |
| `ChangeProposal` | objective, behavioral acceptance, non-goals, boundaries, API/data/UI effects, file/proof plan, rollback, risks | Observable, testable, explicit about exclusions, and no unnecessary internal prescription. |
| `ChangeSummary` (Codex, at `Build`) | outcome, behavior/files changed, deviations | Concise and truthful; host owns file/SHA facts; no hidden deviation. Render under Change set. |
| `CodeReview` | base/head SHA, criterion checks, located findings, severity, evidence, repair, verdict | Exact-diff correctness/security/contract review; no stylistic noise. |
| `ProofRecord` (host, at `Check`) | head SHA, criterion and boundary outcomes, commands, environment, evidence, residual risk | `ProofOutcome.Pass`/`Fail`/`Blocked`/`NotRun` stay distinct and every claim is exact-head. |
| `InputRequest` (host-authored human-input fact, not an `Artifact`; it has no producer) | one question, why blocked, attempts, bounded choices/consequences, recommendation | One material decision answerable without reading a transcript. Every rendered field is either host-authored from the closed enumeration or displayed as attributed, visually demoted, quoted untrusted evidence — including *why blocked* and *attempts*, which are the fields an injected repository would otherwise use to steer the choice. The choice set is closed: `Approve`, `Reject`, `StopRun`, and `SelectOne` over host-extracted options, where an option is a host-validated value — a path, SHA, identifier, or registered command checked against host facts — never model prose. An ambiguity that cannot be expressed in that set stops the run rather than degrading into free text. |
| `PrHandoff` (host, at `Publish`) | why, change, proof, resolved findings, residual risk, checklist, PR URL, head SHA | Composed by the host from accepted artifacts — `IssueBrief`, `ChangeSummary`, `ProofRecord`, `CodeReview` — so it costs no model session and cannot disagree with what it summarizes. Orients the owner's own code review in about a minute; it never replaces that review and never asserts certainty the artifacts do not carry. |

Prompts, stage skills, schemas, and fixtures are versioned under `guidance/`; a run records
their content hash and the worker rejects an unknown hash. A prompt editor is not a v1 feature.
GitHub rendering uses the security appendix's single generated-text renderer; UI-safe plain text
is not assumed to be safe GitHub Markdown.
