# Issue Pilot v1 execution and publication security

This document owns the v1 trust model, local execution profiles, provider isolation,
repository admission, source import, GitHub publication, blast radius, and incident cleanup.
The product lifecycle and schemas are owned by [`issue-pilot-v1-spec.md`](issue-pilot-v1-spec.md);
release proofs are owned by
[`issue-pilot-v1-verification.md`](issue-pilot-v1-verification.md).

## Security objective

Repository text and model output are untrusted. Either may steer a model into invoking every
tool it can see. Safety therefore comes from capabilities the host does not grant, never from a
prompt, model judgment, selected filename mask, or claimed branch convention.

The v1 boundary guarantees:

1. provider subscription state is visible only to a provider-control process;
2. model-invoked commands and deterministic checks have no credentials or network;
3. repository configuration cannot automatically add rules, hooks, skills, tools, or MCP;
4. model output cannot access authoritative Git state, GitHub credentials, or lifecycle writes;
5. automated publication cannot execute the candidate tree on a hosted runner before owner review;
6. only a host-owned broker imports a bounded file delta and publishes an exact reviewed SHA;
7. GitHub rules, not worker discipline, contain an installation token to owned branches; and
8. a provider, sandbox, repository, or GitHub fact that cannot be proven fails closed.

## Accepted worker profile

- Python 3.12+ runs under `uv` with `provider-runtime[agent-sdks]` pinned to the immutable
  revision in the main spec.
- Release requires one dedicated local Linux VM on an encrypted persistent volume containing only
  worker-managed repository clones, provider enrollment, and scratch. No personal home, host
  filesystem share, SSH agent, Docker socket, cloud configuration, unrelated checkout, or ambient
  credential socket is mounted.
- Tiers use distinct uids, user/mount/network namespaces, a minimal mount graph, cgroup resource
  limits, and seccomp syscall reduction. Seccomp is never claimed to inspect pathname strings.
- Each run receives a disposable guest clone, a distinct broker-owned clone, and run-scoped
  scratch. Guest Git has no remote credential or authoritative ref/index. The model sandbox
  never mounts broker SCM metadata.
- Direct macOS or everyday-account execution is development-only and cannot satisfy a live
  release proof. Another guest OS or host sandbox is a new profile with its own slice-0 gate.

## Provider-runtime contract

The pinned runtime is an implementation candidate, not evidence that the contract holds. Slice
0 must close this ledger:

| Capability | Current evidence | Required live proof |
|---|---|---|
| Subscription login | Codex and Claude adapters exist; supported-profile execution is unproven. | One real schema-valid call per vendor inside the release profile. |
| Automatic configuration disabled | Claude selects no filesystem setting sources; Codex documents partial CLI controls, but the pinned SDK mapping and `AGENTS.md`/skill/MCP isolation are unproven. | Hostile root and nested fixtures load no rule, hook, skill, trust entry, or MCP server. |
| Fresh non-auth state | Desired but not proven by the pinned revision. | A second run observes no first-run conversation, memory, trust, or writable state. |
| Nested command containment | Runtime sandbox machinery exists; the supported profiles are unproven. | Commands cannot read enrollment/canary files and produce zero network callbacks. |
| Provider-control egress split | Architecture exists; in-process tools remain vendor-specific. | Provider calls work while local commands and vendor-hosted network tools remain denied. |

Claude launches through the Agent SDK with:

- no filesystem setting sources;
- strict, explicit, empty-by-default MCP configuration;
- automatic memory disabled;
- a closed tool allowlist; and
- no WebSearch or WebFetch capability.

Codex launches through the SDK/app-server with the equivalents of the documented
`--ignore-user-config`, `--ignore-rules`, and `--ephemeral` controls, an explicit closed
feature/MCP configuration, and web search disabled. Those flags are necessary, not sufficient:
slice 0 must prove the pinned runtime also declines automatic `AGENTS.md`, skill, trust, and MCP
loading. If either SDK cannot express the complete controls while retaining subscription login,
that provider fails slice 0. Prompt text and filesystem overlays are not substitutes.

The dedicated provider account has a worker-owned audited home containing only enrollment
material. Each invocation receives a fresh writable non-auth state root. If a vendor always
loads a home-level file or managed policy, the release profile names and hashes it; an unexpected
file or changed hash blocks launch. Authentication may persist. Conversation, automatic memory,
project trust, settings, hooks, skills, MCP discovery, and writable session state may not.

Repository `AGENTS.md`, `CLAUDE.md`, `.claude/`, `.codex/`, and `.mcp.json` bytes remain ordinary
untrusted source. They may be read explicitly like any other file, so no selected filename set
is a prompt-injection boundary. The delta importer rejects creation, modification, or deletion
under those agent-configuration paths; base files remain present but are never automatically
honored.

## Process tiers

| Tier | Reads | Writes | Egress | Credentials |
|---|---|---|---|---|
| Provider control | Guest worktree and bounded input artifacts; audited enrollment state as required | Provider-owned fresh run state; guest tree only for Build/repair | Pinned vendor endpoints only | One provider subscription enrollment only |
| Model command | Guest worktree | Guest worktree and run scratch within stage policy | None | None |
| Resolver | Base manifests/lockfiles and registered command ids | Dependency tree/cache for that base | Registered package registries only | None |
| Check | Read-only provisioned committed tree plus isolated scratch | Bounded scratch/result pipe | None | None |
| Tree walker/importer | Frozen guest and immutable base | Broker worktree through validated operations | None | None; cannot read registry file |
| SCM broker | Broker clone and exact host facts | Broker clone and owned remote ref through explicit operation | GitHub only while performing the host operation | One just-in-time installation token only |

Provider and guest-command egress are different OS capabilities. A provider process receives a
rebuilt environment and no GitHub, cloud, worker, or SSH credentials. Model-command subprocesses
run in a nested filesystem/network sandbox with no subscription state mounted. In-process
provider file tools are restricted to the worktree by the adapter; slice 0 drives those exact
tools against canaries because a child-process sandbox cannot prove their behavior.

After any model or check process exits, the worker kills its process group and verifies that no
PID remains before any credential is minted. A non-empty process group is a launch-invariant
defect, not cleanup success.

## Repository registry and admission

A local mode-`0600`, owner-writable registry outside all checkouts maps `RepositoryKey` to:

- canonical checkout and GitHub identity;
- default branch;
- closed setup/check command ids and exact argv;
- time and byte bounds;
- sensitive and protected path policy; and
- the supported provisioning profile.

Server payloads cannot supply paths, commands, environment, prompt text, registries, or
permission modes.

Registration and every observed default-branch move enqueue a worker admission for the exact
base. The worker executes setup and checks inside the release profile and emits a bounded
`RepositoryAdmissionReceipt`: repository/base identity,
manifest/lockfile hash, provisioner revision, OS-image identity, command ids, profile, and
outcomes. It contains no local path. Start rereads GitHub and activates a run against that base;
the run remains in Inbox and no model stage is requested until a `Supported` receipt names the
exact base. `Supported` proves dependencies materialized and every registered command launched,
terminated within bounds, and produced a classified Pass/Fail result under the sandbox. A normal
nonzero test result is admitted baseline evidence; missing tooling/dependencies, forbidden
execution, network attempt, timeout, malformed result, or sandbox violation is `Unsupported`.
A change to the base, commands, provisioner, or OS image invalidates admission.

The only v1 profile is `prebuilt-no-scripts`:

- dependency lifecycle hooks are disabled;
- `.pnpmfile.cjs` and source-controlled registry redirection are not honored;
- Python project/PEP 517 source builds do not run;
- dependencies require prebuilt distributions;
- checks run with no network; and
- an unregistered registry, install script, source build, or test-time network requirement fails
  admission rather than weakening the sandbox.

The resolver is the only guest tier with egress. Initial admission runs it before model output
exists, against the current `Base pin` manifests and lockfiles, allowlisted to registered package
registries, with no access to the registry file, keychain, provider state, outbox, broker clone,
or another repository. Before launch, a host parser proves every lockfile fetch target is a
registered registry artifact; direct URL, Git, local-path, unknown-format, or unresolved targets
fail admission. The network layer independently limits the resolver to those registry origins.
The provisioned tree is mounted read-only for checks.

A model change to a manifest, lockfile, `.npmrc`, build backend, toolchain pin, Makefile, or
`.envrc` is a sensitive build-tooling change. It requires owner approval before the same resolver
acts on model-selected package specifications. Re-resolution retains every original restriction.
Failure is `ProofOutcome.Blocked`, never a reason to open network. Repeated failure invalidates
repository admission. A future network-denied dependency-build profile requires a new threat
model and decision.

## Guest-to-broker import

The importer runs after the guest is frozen read-only, under a uid that cannot read the local
registry, keychain, provider enrollment, delivery outbox, or broker credentials. It computes a
bounded delta against the immutable base and accepts only regular files and directories.

It rejects:

- traversal, absolute paths, overlong/deep paths, and per-file/total-byte overflow;
- escaping or dangling symlinks, hardlinks, device nodes, FIFOs, sockets, and special mode bits;
- every `.git` path component, submodule, or dirty source-checkout mutation;
- creation, modification, or deletion of protected agent configuration;
- an unknown repository, base, guidance hash, or contract revision; and
- unapproved sensitive build-tooling or policy paths.

The broker is a dedicated clone, never a linked worktree. Every Git invocation disables system
and global configuration, sets an empty hooks path, and uses `--no-verify`. Changes to
`.gitattributes` that can alter committed bytes require owner approval. The locally checked,
reviewed, committed, and pushed bytes must be identical.

## Artifact egress and worker identity

Only bounded schema-valid artifacts and normalized receipts leave the worker **for the Issue
Pilot control plane**. Provider-bound data is a separate disclosed egress: prompts, accepted
input artifacts, file contents, diffs, and command/tool results needed by Codex or Claude transit
to that vendor under the owner's subscription settings and terms. V1 cannot promise that source
stays on the machine or that the provider retains nothing. Repository registration requires the
owner to acknowledge both vendors before enabling runs. Raw local transcripts are not uploaded
to the control plane. Successful stage session material is deleted after its schema-valid
artifact is acknowledged; a defect may quarantine its application-encrypted diagnostic session
for at most seven days, and purge deletes it sooner. Schema and byte limits, not content
classification, are the hosted boundary: a model can place repository source or accidentally
captured secrets in free text. Credential-shape matching is a tripwire that defects the run,
never a redactor that converts unsafe output into apparent success.

The owner creates a high-entropy one-time enrollment bootstrap. The exchange creates a separate
scoped worker credential; identical exchange replay returns the same application-encrypted
delivery until its first authenticated claim consumes the bootstrap and deletes that material.
Afterward the raw worker credential exists only in the OS keychain and the server retains a
domain-separated verifier. An SQLite outbox with application-encrypted payloads resends accepted
provider results until the control plane acknowledges them; its key lives in the worker keychain
and it owns delivery recovery only.

`justify-polling`: subscription execution and repository paths live on a worker with no stable
inbound address, so outbound claim polling avoids an inbound daemon or message broker. Requests
return immediately: two seconds for one minute after activity, then exponential backoff with
jitter to twenty seconds; transport failures cap at sixty seconds. Disconnect never cancels,
fails, or reassigns a stage. A sticky claim moves only after explicit owner action, and every new
claim epoch fences the previous worker.

Exactly-once provider invocation is impossible across a crash between provider completion and
receipt persistence. Duplicate subscription spend is accepted and recorded; duplicate output is
discarded rather than presented as recovered work.

## GitHub App boundary

Install one private GitHub App only on registered repositories. Request exactly:

- Metadata read;
- Contents read/write;
- Pull requests read/write;
- Checks and Commit statuses read; and
- Administration read for rulesets, branch protection, workflow permissions, and runner
  admission reads.

Do not request Actions secrets, Deployments, or Workflows.

### Automated-publication admission

Registration and every push recompute a versioned proof over `.github/**`, `CODEOWNERS`, current
repository/organization rulesets, Actions workflow permissions, runners, and every referenced
local workflow. A missing, unreadable, unknown, or unparseable fact is
`HumanPublishRequired`.

Automated same-repository push is allowed only when all of these hold:

1. no `push` workflow can match the owned branch namespace `refs/heads/issue-pilot/**`;
2. no `create` workflow exists, because first push creates the branch and the trigger has no ref
   filter;
3. no workflow exists on `pull_request_target`, `workflow_run`, `check_run`, `check_suite`,
   `status`, `issue_comment`, `pull_request_review`, `pull_request_review_comment`,
   `merge_group`, or `repository_dispatch`;
4. every reachable PR job declares `permissions: {}` and receives no secret, inherited secret,
   OIDC grant, or write authority;
5. no reachable job checks out or executes the candidate tree: `run`, candidate checkout,
   local action, candidate-sourced container/build configuration, and dynamic action/ref/command
   selection all deny automated publication;
6. each remaining metadata-only action is both on the closed audited admission allowlist and a
   commit-SHA pin, and every supplied input is host/base metadata, never PR title/body or
   candidate-controlled text;
7. no remote/local reusable workflow or self-hosted runner is reachable or registered;
8. no ruleset-injected required workflow applies, and the required-check policy is satisfiable by
   the remaining metadata-only automation; and
9. an owner-created ruleset pair proves that outside `refs/heads/issue-pilot/**` the App cannot
   create or update refs, and inside it the App cannot non-fast-forward update or delete refs.
   The trusted owner is the explicit bypass actor where recovery requires one; the App is absent
   from every bypass list and has no permission to change these rules.

GitHub Actions is the only automated code-execution system v1 can machine-admit. Repository
registration therefore records a versioned owner attestation that no webhook, installed App,
external CI, preview deployer, or other integration executes repository code or performs a
privileged mutation on owned-branch push or draft-PR events. A repository whose answer is yes or
unknown is `HumanPublishRequired`. This is an explicit one-owner trust assumption, not part of
the machine proof; adding another administrator or automation installer requires fork-based
publication or a new admission mechanism.

Blocking only force-push and deletion is insufficient: a compromised Contents-write token can
fast-forward the default branch. Installation tokens scope to repositories and permissions,
never refs. GitHub-enforced rulesets earn the branch boundary.

Read-only token permissions alone are also insufficient. Candidate scripts on a normal hosted
runner still have network and the checked-out private source, so they can exfiltrate code or act
on ambient runner capabilities before human review. V1 therefore admits metadata-only PR
automation, not ordinary lint/test/build CI. A future canonical credential-free, network-denied
hosted runner profile needs its own capability proof; it is not inferred from workflow prose.

### Publish and observation

The broker fetches before model execution and pushes only after every model/check process has
exited and exact-SHA review has passed. It derives repository, branch, and latest `Base pin` from
host facts. It persists a local commit receipt, pushes fast-forward-only with an expected old ref
value, and reconciles an ambiguous effect through an authoritative GitHub read. It never uses
force, force-with-lease, or the REST refs update endpoint. Owned-branch drift defects rather than
retrying.

The adapter recomputes base SHA and the complete CI-admission proof immediately before push and
again immediately before draft-PR creation; an observed change stops before that next effect and
returns to base refresh. GitHub offers no atomic “perform this push/create PR only if base and
configuration still equal this observation” precondition. A trusted owner changing CI in either
final read-to-effect gap is therefore an accepted one-owner residual race: the push or PR event
could start newly unsafe automation before Issue Pilot detects the change and withdraws
readiness. A second writer or untrusted base-branch automation requires fork-based publication
or a new threat model; fresh reads cannot claim to erase this gap.

The App cannot publish `.github/workflows`; the owner must publish the reviewed commit out of
band before the run can continue. Merge, deploy, release, branch deletion, and production
mutation are absent capabilities.

Draft-PR title/body and commit messages are generated-text boundaries. One
`GitHubHandoffRenderer` owns them: commit messages use fixed host text plus the issue alias; the
title accepts only bounded single-line display text through an owned GitHub-title type; and the
creation body is fixed host text identifying the pending exact SHA. After checks satisfy, the
final body renders only the host-owned `PrHandoff` through fixed Markdown scaffolding, explicit
host-validated links, and indented literal blocks for every model/repository-authored field. Raw
model Markdown or HTML is never concatenated. The authoritative title/body digest must match that
render before Ready; drift withdraws readiness and only explicit owner `Restore handoff`
overwrites it. Fixtures containing mentions, issue references, links, images, HTML,
backtick fences, controls, and bidirectional text must render as inert attributed text and create
no unintended reference or notification.

Webhook payloads are signed wake signals, not truth. After deduplication, the adapter rereads PR
head, PR base-branch head, checks, commit statuses, and current branch rules. Only that stabilized
observation may publish, withdraw, or restore Ready. Every request that could render Ready
performs the same batched fresh read; GitHub unavailability renders `Checking GitHub`.

### Installation-token lifecycle

GitHub fixes installation-token lifetime at one hour. Before minting, append an
`InstallationCredentialIssuance` fact with the attempt epoch, scope, database start time, and
bounded call deadline. Mint just in time for one repository with the narrowest permission subset.
Before returning the token to the broker, application-encrypt it into the dedicated Postgres
credential-material row; the AEAD key stays in the deployment secret store and plaintext never
enters a row or log. Append the revocation/expiry receipt and explicitly delete the material row
when it is no longer authority-bearing.

A crash after GitHub returns a token but before escrow commits is fundamentally ambiguous because
GitHub provides neither caller idempotency nor revoke-by-id. Recovery records ambiguous issuance
and blocks a new claim/effect for that attempt until the call deadline plus one hour. Every known
lease is revoked before an epoch replacement is admitted. Failed revocation remains visible as
exposure until expiry. Rulesets, not escrow, contain either token's ref authority.

## Blast radius and recovery

A compromised worker can cost both subscription accounts, registered clones, the local registry,
vendor-visible source, and any installation token it legitimately earned. The GitHub rulesets
prevent that token from updating refs outside the owned namespace or rewriting/deleting owned
refs. A compromised database without the deployment escrow key sees no token plaintext; a
compromised control plane has that key and the App private key and therefore full App authority on
every installation.

Recovery is explicit: revoke the worker verifier, rotate the App key, revoke both subscription
enrollments, inspect owned refs and PRs, and restore a damaged ref through owner-controlled
history. No prompt or application log is accepted as containment evidence.

## Emergency purge

Purge is an owner-only durable incident operation, absent from MCP and bulk UI:

1. record typed confirmation, fence the active attempt, block new Issue Pilot-authorized effects,
   queue revocation of known leases, and withdraw Ready;
2. explicitly select and delete the content-bearing issue graph in dependency order, with no
   cascade and exact-row assertions;
3. insert a content-free tombstone and local purge dispatch;
4. make the worker delete matching outbox payloads, raw transcripts, disposable clones, and
   scratch through opaque local handles; and
5. retain a loud settings item until the content-free worker receipt arrives.

Late stage completion is rejected and cannot recreate the issue. Credential leases and ambiguous
issuance records remain in the separate content-free security ledger until revocation or
conservative expiry. GitHub commits/PRs and managed backup copies are different systems: Issue
Pilot links the owner to the remote incident procedure and discloses backup retention; it never
labels any of them as deleted.

## Primary references

- [Claude headless behavior](https://code.claude.com/docs/en/headless)
- [Claude Agent SDK settings](https://code.claude.com/docs/en/agent-sdk/claude-code-features)
- [OpenAI Codex SDK](https://learn.chatgpt.com/docs/codex-sdk)
- [OpenAI Codex non-interactive mode](https://learn.chatgpt.com/docs/non-interactive-mode)
- [Linux seccomp filter limits](https://docs.kernel.org/userspace-api/seccomp_filter.html)
- [GitHub App permission requirements](https://docs.github.com/en/rest/authentication/permissions-required-for-github-apps)
- [GitHub installation access tokens](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/generating-an-installation-access-token-for-a-github-app)
