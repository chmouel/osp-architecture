---
id: ADR-0003
title: AI failure investigation as a follow-up PipelineRun
date: 2026-10-13
status: proposed
authors:
  - "@chmouel"
reviewers:
  - "@theakshaypant"
  - "@zakisk"
tags:
  - pipelines-as-code
  - ai
---
<!-- Status lifecycle:
- Proposed: Under discussion, not yet accepted
- Accepted: Approved, ready for implementation
- Implemented: Done and reflected in the product
- Deprecated: Not required anymore
- Superseded: Replaced by a newer ADR (link to it)
- Rejected: Not accepted (stays in repo for reference)
-->
---

# ADR-0003: AI failure investigation as a follow-up PipelineRun

## Context

`Required`

Pipelines-as-Code (PAC) has a tech preview that sends a failed PipelineRun
to an LLM and posts the answer on the pull request
([docs](https://pipelinesascode.com/docs/guides/llm-analysis/)). The PAC
controller calls the OpenAI or Gemini HTTP API itself, in the reconcile
loop, before it starts the next queued PipelineRun. Known problems:

- A slow model call holds up the repository concurrency queue.
- Every rerun posts a new comment instead of updating the last one
  (SRVKP-11549).
- Logs go to the model without secret redaction (SRVKP-9107).
- The context lacks the PR description and Kubernetes events
  ([#2303](https://github.com/tektoncd/pipelines-as-code/issues/2303),
  [#2304](https://github.com/tektoncd/pipelines-as-code/issues/2304)).

SRVKP-14253 asks for an agent based version, with OpenCode on GitHub as
the first delivery. A prototype moved the analysis into child PipelineRuns
that PAC builds and that run an agent CLI (codex, claude, gemini or
opencode). We rebased and hardened that prototype, and the work exposed
costs that hardening alone does not remove:

- The Repository CRD gains a field for each backend concern: image,
  Vertex project and region, environment, token limits. PAC has to track
  the CLI flags of four agents.
- The agents run with permission prompts disabled, next to a checkout of
  the pull request. Most of the hardening went into keeping
  PR-controlled files and git credentials out of their reach.
- The documented agent image was never published, and OpenShift
  Pipelines has no process to build, scan and ship it.
- Results come back by parsing pod logs, and the agent output can imitate
  the format.
- The proposed "apply suggestions" flow pushes commits to the PR branch,
  which needs push credentials, fork handling and an ACL check.
- End-to-end tests need a real model or a fake agent image.

The PAC team also built paco
([pipelines-as-code/paco-cli](https://github.com/pipelines-as-code/paco-cli)),
which reviews pull requests with Claude. It runs from a shared remote
Pipeline that a repository references in a short PipelineRun. It runs no
subprocesses, gives the model read-only tools over a source snapshot,
reads its rules from the base branch, redacts credentials on the way in
and scans the model output on the way out. It ships its own image.

## Decision

`Required`

Chosen option: run AI failure investigation as a user-owned follow-up
PipelineRun that uses paco. PAC starts it with a redacted failure summary
and publishes what it reports. We prefer this to shipping the agent runner
from the prototype or keeping the in-controller HTTP calls.

The `spec.settings.ai` block of the Repository CRD gets deprecated and
then removed.

### Architectural Overview

`Required`

1. A repository adds `.tekton/investigate.yaml`. It is a short
   PipelineRun that references paco's shared investigate Pipeline and
   asks to run when another PAC PipelineRun on the same event fails.
2. When the failed PipelineRun finishes, PAC matches the follow-up the
   same way it matches any event. PAC always reads the follow-up
   definition from the default branch, so a pull request cannot add one
   or change it.
3. PAC writes a failure summary into a ConfigMap owned by the follow-up
   run: failed tasks and steps, error snippets, log tails, Kubernetes
   events, PR title and description. It replaces the values of the
   secrets attached to the failed run, and the git provider token, with
   `*****`.
4. paco reads the summary, a snapshot of the PR head and the roles in
   `.tekton/ai/` from the base branch. It calls the model with read-only
   tools and writes a report.
5. PAC checks the report, redacts it again and posts it through its own
   provider layer. That gives one PR comment per PipelineRun and role,
   updated on `/retest`, or a check run. Suggested fixes become
   provider-native suggestions that a person applies.

The follow-up trigger is generic. Teams can use it for chat
notifications or flaky test reports without any AI involved.

### Implementation Details

`Required`

PAC, upstream in `tektoncd/pipelines-as-code`. The mechanics go to an
upstream PAC design issue, and this ADR only records the shape:

- A follow-up event (working name `pipelinerun-completed`), with a CEL
  `pipelinerun` object (name, status, failed tasks). Follow-up runs never
  trigger further follow-ups and do not take a slot in the concurrency
  queue.
- The failure summary ConfigMap, exposed as a `{{ failure_context }}`
  variable.
- Report ingestion. The report step prints the report between markers
  that carry a per-run nonce and records its SHA-256 as a Tekton result,
  since results default to 4 KB. PAC rejects a report whose nonce or
  digest does not match.
- Publishing through the existing provider interface for GitHub, GitLab,
  Gitea and Bitbucket, reusing the comment marker and redaction code
  written while hardening the prototype.

paco, in `pipelines-as-code/paco-cli`:

- An `investigate` mode next to `review`, with the same split into
  context, model and report steps, and a shared
  `paco-investigate.pipeline.yaml`.
- A recorded-response model fake for tests.

OpenShift Pipelines:

- Build and ship the paco image downstream, or document the upstream
  image as unsupported (open question below).

Affected components: PAC controller and watcher, Repository CRD, paco,
PAC documentation.

## Security & Compliance

`Optional`

- The follow-up PipelineRun receives model credentials, so PAC only
  loads it from the default branch. This matches the existing
  `pipelinerun_provenance: default_branch` setting.
- The model never runs a shell. paco exposes read-only snapshot tools,
  and web search can be turned off.
- Redaction happens twice on the way in (PAC for exact secret values,
  paco for credential patterns) and twice on the way out (paco scans
  the output, PAC redacts it before posting).
- The git provider token stays in the PAC controller. Nothing pushes
  commits for the user.
- Data leaves the cluster: log excerpts, PR text and source go to the
  configured model provider (Anthropic API or Vertex AI). The feature is
  opt-in per repository, and the docs must list what is sent.
- Exact-value redaction misses a secret that a step prints encoded or
  split across lines. The docs keep the existing warning about
  pipelines that print secrets.

## Compatibility & Migration

`Optional`

- `spec.settings.ai` stays for one release with a deprecation warning,
  then goes away. It is a tech preview, so no support commitment covers
  it.
- Repositories that still use the old `provider` and `api_url` fields
  get an `LLMAnalysisInvalidConfig` event that links to the migration
  docs.
- `tkn pac generate` gains an option that turns an existing `settings.ai`
  block into `.tekton/investigate.yaml` plus `.tekton/ai/*.md` roles.
- Rollback: a repository deletes `.tekton/investigate.yaml`. The
  follow-up trigger has no effect on repositories that do not use it.

## Testability

`Required`

- PAC e2e on Gitea uses a stub follow-up step that reads the failure
  summary, checks that a planted secret was redacted and prints a canned
  report. The tests assert one comment per PipelineRun and role, an
  update in place on `/retest`, and that a follow-up does not start
  another follow-up.
- PAC unit tests cover matching, default-branch loading, the nonce and
  digest checks and redaction.
- paco tests run `investigate` against recorded model responses.
- CI needs no real model and no agent image.

## Operational Readiness

`Optional`

- Follow-up runs are ordinary PipelineRuns, visible in the console and
  in `tkn pac` like any other run.
- PAC emits Repository events when a report fails validation or cannot
  be published.
- A failing or slow investigation does not block the repository queue
  or change the commit status of the original run.

## Scope

`Required`

- **In-Scope:**
  - Explaining failed PipelineRuns on pull requests and pushes.
  - The PAC follow-up trigger, failure summary and publishing.
  - paco `investigate`.
  - Deprecating `spec.settings.ai`.
- **Out-of-Scope:**
  - PR code review, which paco already covers.
  - Pushing fixes to pull request branches.
  - Model providers beyond what paco supports in its first release.
  - Running agent CLIs (codex, claude, gemini, opencode) inside PAC.

## Acceptance Criteria

`Required`

The decision is considered accepted when:

- [ ] Problem, scope, and non-goals are clear
- [ ] Alternatives were considered and compared
- [ ] Chosen option's trade-offs (not just benefits) are documented
- [ ] Backward compatibility / migration path addressed
- [ ] Security/RBAC impact assessed
- [ ] Observability and rollback plan defined
- [ ] PAC upstream maintainers agree on the follow-up trigger
- [ ] Product agrees to change the SRVKP-14253 delivery scope
- [ ] Decision on shipping the paco image downstream
- [ ] Required reviewers approved
- [ ] Open concerns resolved or tracked

## Considered Options

`Required`

| Option | Pros | Cons |
| ------ | ---- | ---- |
| Follow-up PipelineRun with paco (chosen) | No shell for the model; one image that paco already ships; PAC keeps provider coverage; the trigger helps non-AI use cases | Needs an upstream PAC feature and paco work; Claude only at first; two repositories to coordinate |
| Ship the prototype (PAC builds child runs that run agent CLIs) | Mostly written; several backends; matches the OpenCode request | Agents run with full shell access; PAC tracks CLI flags of four tools; no published image; log parsing can be forged |
| Keep the in-controller HTTP calls | Already in the product; no image | Blocks the queue; no tools for the model; hard to extend; hardening still needed |
| Do nothing | No effort | Tech preview keeps its known bugs and no redaction |

## Decision Drivers

`Required`

- Security: a pull request author must not be able to steer an agent
  that holds credentials.
- Maintenance: PAC should not own agent CLI flags or an agent image.
- Coverage: results must reach every git provider PAC supports.
- Reuse: paco already solves model access, redaction and output
  scanning.

## Consequences

`Required`

- Positive outcomes:
  - The model runs without a shell or credentials beyond its own API
    key.
  - Repository owners configure the feature in `.tekton/`, where they
    already configure everything else.
  - The CRD loses the backend specific fields.
- Risks:
  - Upstream PAC maintainers may reject or reshape the follow-up
    trigger.
  - paco supports Claude only, while SRVKP-14253 asked for OpenCode.
  - Two repositories have to release in step.
- Technical debt or follow-up ADRs required:
  - Push events have no pull request to comment on. Check run only, or
    skip them?
  - Should follow-up runs get their own small concurrency limit?
  - An OpenAI-compatible adapter in paco, if product needs one.

## Dependencies

`Required`

- Upstream PAC acceptance of the follow-up trigger.
- paco `investigate` mode and its shared Pipeline.
- Access to Anthropic API or Vertex AI from the cluster.
- A Konflux build of the paco image if OpenShift Pipelines ships it.

## References

`Required`

- [PAC LLM analysis documentation](https://pipelinesascode.com/docs/guides/llm-analysis/)
- [paco-cli](https://github.com/pipelines-as-code/paco-cli) and its [security design](https://github.com/pipelines-as-code/paco-cli/blob/main/docs/security.md)
- [tektoncd/pipelines-as-code#2303: PR description in LLM context](https://github.com/tektoncd/pipelines-as-code/issues/2303)
- [tektoncd/pipelines-as-code#2304: Kubernetes events in LLM context](https://github.com/tektoncd/pipelines-as-code/issues/2304)
- [JIRA: SRVKP-14253](https://redhat.atlassian.net/browse/SRVKP-14253)
- [JIRA: SRVKP-11549, duplicate comments](https://redhat.atlassian.net/browse/SRVKP-11549)
- [JIRA: SRVKP-9107, redaction](https://redhat.atlassian.net/browse/SRVKP-9107)

## Change Log

`Required`

| Date | Author | Change Summary |
| ---------- | -------- | ------------------- |
| 2026-10-13 | @chmouel | Created initial ADR |
