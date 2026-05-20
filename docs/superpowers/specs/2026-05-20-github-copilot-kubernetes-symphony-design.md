# GitHub Copilot Kubernetes Symphony Design

Date: 2026-05-20

## Purpose

Build a working Go implementation of Symphony for Stellar's GitHub workflow.
The service will use GitHub Projects and labels as the queue, run in Kubernetes
as a long-lived controller, and launch one Kubernetes worker pod per issue
attempt. Each worker uses GitHub Copilot CLI with explicit model routing:
Opus for planning, GPT for plan review, and Sonnet for implementation.

This is a v1 design. It intentionally defers HA, leader election, durable
controller state, and broad hardening until the workflow proves useful.

## Decisions

- Implementation language: Go.
- Tracker: GitHub Enterprise / GitHub Projects via `gh` and GraphQL.
- Project scope: Stellar organization Project 58.
- Dispatch gate: label-driven eligibility.
- Worker runtime: Kubernetes, one pod pipeline per issue attempt.
- Workspace storage: persistent per-issue PVC.
- Agent runner: GitHub Copilot CLI, not Copilot cloud agent.
- Model routing:
  - planning: `claude-opus-4.7`
  - plan review: `gpt-5.5`
  - implementation: configurable Sonnet model, initially
    `claude-sonnet-4.6` unless `claude-sonnet-4.7` is available and
    configured.
- GitHub write ownership: split between controller and worker.

## Architecture

The v1 system has three main runtime surfaces.

1. `symphony-controller` runs as a single-replica Kubernetes Deployment.
   It loads `WORKFLOW.md`, polls GitHub Project 58, claims eligible items,
   creates PVCs and Jobs, watches worker status, records retry state, and
   exposes logs/status for operators. GitHub access uses `gh` subprocesses in
   v1; Kubernetes access uses the Go Kubernetes client so Jobs, Pods, PVCs,
   logs, and watches are typed and testable.

2. Worker Jobs run one pod per claimed issue attempt. A worker mounts the
   per-issue PVC, prepares or reuses the repository checkout, runs the
   Copilot model pipeline, validates the work, and performs PR/workflow writes
   through `gh`.

3. Per-issue PVCs hold durable workspace state: checkout, branch, plan
   artifacts, plan-review artifacts, implementation notes, logs, and
   validation evidence. PVCs survive retry attempts and are cleaned up only by
   terminal cleanup rules or explicit operator action.

The upstream Symphony spec remains the conceptual base: tracker reads,
orchestrator state, workspace safety, retry handling, and observability. This
design changes the concrete integration points from Linear/Codex/Namespace to
GitHub Projects/Copilot CLI/Kubernetes.

## Workflow Configuration

`WORKFLOW.md` remains the repository-owned policy contract. V1 extends the
front matter with these implementation-defined settings:

```yaml
tracker:
  kind: github_project
  owner: stellar
  project_number: 58
  ready_label: symphony-ready
  claim_label: symphony-claimed
  running_label: symphony-running
  blocked_label: symphony-blocked
  terminal_labels:
    - symphony-done

worker:
  kind: kubernetes
  namespace: symphony
  image: ghcr.io/stellar/symphony-worker:latest
  workspace_mount_path: /workspace
  service_account_name: symphony-worker
  pvc_storage_class: ""
  pvc_size: 20Gi
  job_ttl_seconds_after_finished: 86400

runner:
  kind: copilot_cli
  planner_model: claude-opus-4.7
  plan_reviewer_model: gpt-5.5
  implementation_model: claude-sonnet-4.6
  reasoning_effort: xhigh
  post_plan_comment: true
  post_plan_review_comment: true
```

The Markdown body supplies the worker prompt template. It should describe the
team workflow, PR handoff requirements, validation expectations, and how to
use `gh` for issue/PR updates.

## GitHub Tracker And State

The tracker normalizes GitHub Project items into work items containing:

- Project item ID.
- Linked issue or PR node ID.
- Repository owner/name.
- Issue or PR number.
- Title, body, URL, labels, assignees, project fields, and timestamps.

Dispatch eligibility is label based. An item is eligible only when it has the
configured ready label and does not have claim, running, or blocked labels.
Project status fields are read for observability and future policy, but they
are not the v1 dispatch gate.

The controller owns orchestration writes:

- Add the claim label before creating Kubernetes resources.
- Add the running label when a worker Job is launched.
- Remove the running label when the attempt ends.
- Keep the claim label while retry backoff is active.
- Add the blocked label for non-retryable orchestration or startup failures.
- Remove orchestration labels during terminal cleanup.

The worker owns workflow writes:

- Create/update branches and PRs.
- Maintain issue workpad comments.
- Post the planning artifact to the original GitHub issue as Markdown.
- Post the plan-review findings to the original GitHub issue as Markdown.
- Respond to PR review comments.
- Attach validation evidence.
- Move or annotate human handoff states when the workflow prompt requires it.

This split prevents duplicate dispatch while keeping team-specific product
workflow in `WORKFLOW.md`.

## Kubernetes Worker Model

The controller creates or reuses one PVC per issue using a sanitized key. It
then creates one Kubernetes Job per attempt. The worker pod mounts the PVC at a
fixed workspace path, for example `/workspace`.

The worker image includes:

- `git`
- `gh`
- `copilot`
- repository build/runtime dependencies needed by the target codebase
- a small Symphony worker entrypoint

Credentials are supplied through Kubernetes Secrets and scoped environment
variables or mounts:

- GitHub token with repository and project permissions.
- Copilot CLI authentication/config.
- Optional package registry credentials.

The one-pod pipeline runs stages in order:

1. Planning with `claude-opus-4.7`.
2. Plan review with `gpt-5.5`.
3. Implementation with the configured Sonnet model.
4. Validation and PR handoff according to `WORKFLOW.md`.

After the planning stage, the worker posts the plan draft to the original
GitHub issue as Markdown. After the review stage, it posts the GPT-5.5 review
findings to the same issue as Markdown. These comments are operator visibility
artifacts; the PVC remains the durable source of run artifacts.

Issue comments must be idempotent within a run attempt. The worker should use
hidden markers such as `<!-- symphony:plan:<attempt-id> -->` and
`<!-- symphony:plan-review:<attempt-id> -->` so a retry or revised stage can
update the matching comment instead of creating duplicates. A later attempt may
create a new pair of comments when that makes the retry history clearer.

The worker writes artifacts into the PVC under a stable directory such as
`/workspace/.symphony/`:

- `plan.md`
- `plan-review.md`
- `run.jsonl`
- validation logs
- final handoff notes

Configured model unavailability is a configuration error. V1 should not
silently fall back to another model.

## Controller Deployment

The controller runs as a single-replica Deployment. It uses a dedicated
Kubernetes ServiceAccount scoped to the configured namespace.

For v1, the controller needs permissions to:

- Create, list, watch, and delete Jobs and Pods in its namespace.
- Create, list, watch, and delete PVCs in its namespace.
- Read pod logs and events.
- Read named Secrets if the deployment pattern requires it.

The controller may expose an internal HTTP status endpoint such as
`/api/v1/state` through a ClusterIP Service or port-forward. This endpoint is
for operator debugging and is not required for orchestration correctness.

Deferred hardening:

- Leader election.
- Multi-controller HA.
- Durable controller database.
- Network policy templates.
- Autoscaling.
- Fine-grained admission controls.

## Deployment Target

V1 deploys to the `wallet-eng-dev` Kubernetes namespace/environment managed in
the kube repository at:

```text
/Users/adityavyas/Desktop/Work/kube/kube001-dev-eks/namespaces/wallet-eng-dev/
```

The implementation should follow the existing kustomize layout in that
directory:

- Add the Symphony manifest at `symphony/symphony.yaml`.
- Add the new manifest path to
  `kube001-dev-eks/namespaces/wallet-eng-dev/kustomization.yaml`.
- Use ExternalSecrets/Vault for GitHub and Copilot credentials, matching the
  existing `externalsecrets.yaml` pattern.
- Use the existing single-replica controller style as a reference, especially
  `freighter/oncall-triage.yaml`, while keeping Symphony's RBAC scoped to
  Jobs, Pods, PVCs, pod logs, events, and named Secrets in `wallet-eng-dev`.
- Keep local kubeconfig changes out of source control. Cluster access for
  testing is an operator prerequisite, not a committed artifact.

The first deployment target is dev only. Staging or production promotion should
wait until the v1 workflow has completed real disposable issues successfully in
`wallet-eng-dev`.

## Error Handling And Safety

Startup validates:

- `gh` is installed and authenticated with repository and project access.
- The required GitHub labels exist or can be created according to config.
- Copilot CLI is installed and authenticated.
- Configured Copilot models are accepted for the authenticated account.
- Kubernetes context, namespace, worker image, service accounts, and Secrets
  are available.

Dispatch does not start a pod unless the GitHub claim label was applied
successfully.

Retryable failures use exponential backoff and keep the PVC. Non-retryable
configuration, authentication, or startup failures add the blocked label and do
not continue retrying until a human fixes the blocker.

Filesystem safety rules:

- PVC names and workspace paths are sanitized.
- Worker commands run only inside the mounted workspace.
- The worker does not receive broad host filesystem access.

Secrets must be redacted from logs, Copilot output, and status APIs. The
configured secret env var names are passed to Copilot's redaction controls when
possible.

## Observability

V1 observability is intentionally modest:

- Structured controller logs for poll, claim, dispatch, retry, cleanup, and
  errors.
- Per-worker pod logs streamed or linked by issue key.
- Per-issue artifacts in the PVC under `.symphony/`.
- Optional JSON status endpoint with running jobs, retry queue, last errors,
  PVC names, and recent worker events.

This is enough to debug early runs without building a full dashboard.

## Testing And Validation

Unit tests cover:

- Workflow config parsing.
- Label-based eligibility.
- Claim/release transitions.
- Retry backoff.
- PVC and Job name sanitization.
- Copilot command construction.

GitHub tracker tests use mocked `gh` JSON and GraphQL responses. They do not
require live GitHub.

Kubernetes provider tests use fake Kubernetes clients for PVC/Job creation,
status transitions, and cleanup decisions.

A local integration test can run against kind or minikube with a fake worker
image that writes expected artifacts to the mounted PVC.

Live E2E is optional and manually gated. It should target a disposable GitHub
issue/project item and a non-production Kubernetes namespace.

V1 acceptance criteria:

- A labeled Project 58 item is detected.
- The controller applies the claim label.
- A per-issue PVC is created or reused.
- A worker Job is created.
- The worker runs the three configured model stages.
- The worker posts the plan and plan-review findings to the original GitHub
  issue as Markdown comments.
- Logs and artifacts are captured.
- Symphony releases, retries, or blocks the item deterministically.

## Open Implementation Notes

- The first implementation can use `gh` subprocesses for GitHub reads/writes
  to match enterprise authentication behavior quickly.
- A direct GitHub GraphQL client can replace `gh` later if v1 proves valuable.
- The Kubernetes provider should use the Go Kubernetes client from the start,
  because pod/job/PVC watches and fake-client tests are central to the worker
  lifecycle.
- The implementation should keep runner, tracker, and worker-provider
  interfaces small so future integrations do not require rewriting the
  orchestrator.
