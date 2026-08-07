# KEP-6251: Publish kubectl drain state as Node conditions

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
    - [Story 1: Observe drain progress](#story-1-observe-drain-progress)
    - [Story 2: Coordinate lifecycle controllers](#story-2-coordinate-lifecycle-controllers)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Definitions](#definitions)
  - [Condition transitions](#condition-transitions)
  - [Multiple Nodes](#multiple-nodes)
  - [Failure and interruption](#failure-and-interruption)
  - [Repeated and concurrent drains](#repeated-and-concurrent-drains)
  - [Permissions](#permissions)
  - [Library consumers](#library-consumers)
  - [Test Plan](#test-plan)
      - [Prerequisite testing updates](#prerequisite-testing-updates)
      - [Unit tests](#unit-tests)
      - [Integration tests](#integration-tests)
      - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha](#alpha)
    - [Beta](#beta)
    - [GA](#ga)
    - [Deprecation](#deprecation)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
  - [Infer drain state from existing signals](#infer-drain-state-from-existing-signals)
  - [Publish annotations or labels](#publish-annotations-or-labels)
  - [Use a taint as the drain signal](#use-a-taint-as-the-drain-signal)
- [Infrastructure Needed (Optional)](#infrastructure-needed-optional)
<!-- /toc -->

## Release Signoff Checklist

Items marked with (R) are required prior to targeting a milestone or release.

- [ ] (R) Enhancement issue in release milestone, which links to KEP dir in
  [kubernetes/enhancements] (not the initial KEP PR)
- [ ] (R) KEP approvers have approved the KEP status as `implementable`
- [ ] (R) Design details are appropriately documented
- [ ] (R) Test plan is in place, giving consideration to SIG Architecture and
  SIG Testing input
  - [ ] e2e Tests for all Beta API Operations (endpoints)
  - [ ] (R) Ensure GA e2e tests meet requirements for [Conformance Tests]
  - [ ] (R) Minimum Two Week Window for GA e2e tests to prove flake free
- [ ] (R) Graduation criteria is in place
  - [ ] (R) [all GA Endpoints] must be hit by [Conformance Tests] within one
    minor version of promotion to GA
- [ ] (R) Production readiness review completed
- [ ] (R) Production readiness review approved
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for
  publication to [kubernetes.io]
- [ ] Supporting documentation—e.g., additional design documents, links to
  mailing list discussions/SIG meetings, relevant PRs/issues, release notes

[Conformance Tests]: https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md
[all GA Endpoints]: https://github.com/kubernetes/community/pull/1806
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes.io]: https://kubernetes.io/
[kubernetes/website]: https://git.k8s.io/website

## Summary

Kubernetes uses cordons and taints to express scheduling and eviction policy,
but these mechanisms do not report whether a Node drain is in progress or
whether the Node is drained. Users and controllers must infer drain state from
Pods, Taints, and provider-specific APIs. This is a brittle inference because
there's no shared signal about the Node's state.

[KEP-5683](https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/5683-lifecycle-conditions) introduced the well-known `DrainInProgress` and `Drained` Node
conditions as a common observability interface. This KEP proposes making
`kubectl drain` a  writer of those conditions. When `kubectl drain`
starts processing a Node, it will publish `DrainInProgress=True`. When the
command has met its selected drain criteria, it will publish `Drained=True`.

This proposal does not change how `kubectl drain` selects or removes Pods. It
does not replace cordons and taints. Its purpose is to enhance the existing
drain operation by sharing the state of that operation on the Node.

## Motivation

`kubectl drain` is a client-side operation. It cordons a Node, selects Pods,
requests deletion or eviction, and waits for the selected Pods to terminate.
The command reports progress to the terminal, but that state is unavailable to
other actors.

Inferring drain state from the Node and its Pods is common practice, but it is
inaccurate. A cordon expresses scheduling policy, not whether a drain is active.
Pods on a Node do not indicate if a Node is drained or if a drain is in progress.
Ignored DaemonSets, unmanaged Pods, Pod filters, newly created Pods, and an
interrupted command all muddy the signal. An observer therefore cannot reliably
distinguish a completed drain from one that is active, failed, or completed.

This ambiguity affects the broader Kubernetes ecosystem. Controllers,
operators, autoscalers, infrastructure providers, and lifecycle
tooling all need drain context to coordinate disruption and explain workload
availability. Without a common signal, each project must infer state or publish
custom conditions, annotations, and labels. Those implementations cannot
reliably interoperate or provide portable behavior across clusters.

### Goals

- Publish `DrainInProgress` and `Drained` conditions from `kubectl drain`
- Keep condition reporting best-effort

### Non-Goals

- Define a universal definition of drain
- Change Pod selection, deletion, eviction, waiting, or force behavior in
  `kubectl drain`
- Add a server-side drain API or controller
- Add exclusive condition ownership, locking, or handoff between drain actors

## Proposal

Add an alpha `--report-node-conditions` flag to `kubectl drain`. When enabled,
the command will publish condition updates for each Node through the Node
`status` subresource.

Immediately before processing a Node's selected Pods, `kubectl drain` will
publish:

```yaml
status:
  conditions:
  - type: DrainInProgress
    status: "True"
    reason: KubectlDrainStarted
    message: kubectl drain started processing this Node
  - type: Drained
    status: "False"
    reason: KubectlDrainStarted
    message: kubectl drain has not completed its selected criteria
```

After all Pods selected by that invocation satisfy the existing completion
criteria, `kubectl drain` will publish:

```yaml
status:
  conditions:
  - type: DrainInProgress
    status: "False"
    reason: KubectlDrainCompleted
    message: kubectl drain completed its selected criteria
  - type: Drained
    status: "True"
    reason: KubectlDrainCompleted
    message: kubectl drain completed its selected criteria
```

### User Stories

#### Story 1: Observe drain progress

As a cluster administrator, I want a clear and shared drain signal on the Node,
so that I do not have to infer drain state.

#### Story 2: Coordinate lifecycle controllers

As a lifecycle controller author, I want to observe a well-known drain signal
on the Node, so that my controller can react to disruption or explain
workload unavailability.

### Risks and Mitigations

- A client can disappear while `DrainInProgress=True`. The condition is
  explicitly advisory and its timestamps expose when it was last updated.
  Administrators remain responsible for clearing abandoned state
- Multiple actors can write the same condition. This KEP does not introduce
  ownership. `kubectl` uses a strategic merge patch and documents concurrent
  drain behavior as last-writer-wins
- `Drained=True` can be misread as "zero Pods." The condition message and
  documentation state that completion depends on the admin's selected
  criteria
- Writing Node status requires an additional permission. Reporting is opt-in
  and best-effort, and this KEP does not change default RBAC roles. The required
  permission is documented in [Permissions](#permissions).
- Additional status writes can conflict with other Node status writers.
  `kubectl` patches only the two lifecycle conditions and retries conflicts
  without replacing the complete Node status.

## Design Details

### Definitions

**Conditions**

| Condition | Definition |
|---|---|
| `DrainInProgress` | The Node is actively being drained according to the administrator's definition of drain. Commonly, this involves removing some amount of Pods, volumes, and networks. |
| `Drained` | The Node has reached the drain criteria selected by the administrator. |

The administrator will be responsible for managing these conditions.

**Reasons**

| Reason | Definition |
|---|---|
| `KubectlDrainStarted` | Kubectl started processing the Pods selected for drain on the Node. |
| `KubectlDrainCompleted` | Kubectl observed that its selected drain criteria were met. |
| `KubectlDrainFailed` | Kubectl could not meet its selected drain criteria because the operation failed or timed out. |
| `KubectlDrainInterrupted` | Kubectl handled an interrupt before its selected drain criteria were met. |

The same reason is written to both conditions during a transition. The
condition type and status report the observed lifecycle state.

### Condition transitions

The two conditions are updated together for each transition:

| Event | `DrainInProgress` | `Drained` | Reason |
|---|---|---|---|
| `kubectl drain` called | `True` | `False` | `KubectlDrainStarted` |
| Selected criteria met | `False` | `True` | `KubectlDrainCompleted` |
| Drain fails or times out | `False` | `False` | `KubectlDrainFailed` |
| Process handles an interrupt | `False` | `False` | `KubectlDrainInterrupted` |

`lastTransitionTime` changes when the condition status changes.
`lastHeartbeatTime` records the time of each successful kubectl update.
`kubectl` does not periodically heartbeat these conditions.

A `DrainInProgress=True` condition therefore means that a drain was initiated
and no terminal outcome was reported. It cannot prove that the originating
process is still running. Consumers that need liveness, ownership, or a
deadline require a coordination API such as the future work discussed in
[Specialized Lifecycle Management](https://github.com/kubernetes/enhancements/pull/5769)

### Multiple Nodes

`kubectl drain` can select more than one Node. The command cordons selected
Nodes before draining them sequentially. Condition reporting will instead
begin immediately before the eviction loop for each individual Node.

This ensures that `DrainInProgress=True` means that kubectl reached that Node.
A failure on one Node will publish a failed outcome for that Node while the
command continues its existing behavior for remaining Nodes.

### Failure and interruption

If Pod selection, eviction, deletion, or waiting returns an error,
`kubectl drain` will attempt to set both conditions to `False` with reason
`KubectlDrainFailed`. The original drain error and exit behavior remain
unchanged.

Today, `kubectl drain` does not install a signal handler for its drain
operation. When the process receives `SIGINT`, `SIGTERM`, or `SIGHUP`, the
operating system terminates it immediately. Kubectl does not cancel the drain
through a context, run a cleanup callback, or make a final API request. If the
process exits after publishing `DrainInProgress=True`, that condition remains
unchanged.

When condition reporting is enabled, this behavior will change. On the first
`SIGINT`, `SIGTERM`, or `SIGHUP`, kubectl will:

1. Cancel the active drain context and stop processing additional Nodes.
2. Immediately print `interrupt received; updating drain conditions for node
   "<name>" before exit` to stderr. This message is printed before waiting for
   the API request so the user can distinguish cleanup from a hung client.
3. Make one best-effort strategic merge patch for the Node currently being
   drained, setting `DrainInProgress=False` and `Drained=False` with reason
   `KubectlDrainInterrupted`.
4. Bound that patch with a new five-second context that is independent of the
   canceled drain context.
5. Report a failed or timed-out patch to stderr. After the patch completes or
   the five-second deadline expires, restore the default signal behavior and
   deliver the original signal to the process.

Condition update errors are written to stderr. They do not stop Pod eviction,
change a successful drain to a failure, or replace the command's original
error.

### Repeated and concurrent drains

Starting a new drain resets `Drained=False` and publishes
`DrainInProgress=True`. This makes a sequential re-run idempotent and prevents
the previous completion observation from being mistaken for completion of the
new invocation.

If `DrainInProgress` is already `True`, the new invocation updates
`lastHeartbeatTime`, `reason`, and `message`. It preserves
`lastTransitionTime` because the status did not transition.

### Permissions

Condition reporting adds one permission to `kubectl drain`: `patch` on the
core `nodes/status` subresource. This is in addition to the existing
permissions to patch Nodes, list Pods, and evict or delete Pods.

This KEP does not change any default RBAC roles. Administrators who want drain
conditions to be published must grant `nodes/status` patch permission to their
drain users. If the user can drain a Node but cannot patch its status, the
drain continues and kubectl prints a warning that condition reporting failed.

### Library consumers

The `k8s.io/kubectl/pkg/drain` package is consumed by projects other than the
kubectl command. Adding unconditional status writes to `drain.Helper` would
change their behavior and permission requirements.

The library integration will therefore be disabled by default. The kubectl
command will explicitly install a condition reporter only when
`--report-node-conditions` is set. Existing library callers will retain their
current behavior when they update the dependency.

### Test Plan

- [x] I/we understand the owners of the involved components may require updates
  to existing tests to make this code solid enough prior to committing the
  changes necessary to implement this enhancement.

##### Prerequisite testing updates

No prerequisite test refactoring is required.

##### Unit tests

Unit tests in `k8s.io/kubectl/pkg/cmd/drain` and
`k8s.io/kubectl/pkg/drain` will cover:

- report-node-conditions flag disabled
- successful single-Node and multi-Node drain
- start and completion patch payloads
- Pod selection, eviction, deletion, and timeout failures
- client interuption and the five-second patch deadline
- interruption cleanup progress and failure messages written to stderr
- client and server dry-run
- forbidden and transient status updates
- repeated drains
- a nil reporter for existing library consumers

##### Integration tests

An API server integration test will verify that:

- the two lifecycle conditions can be applied through `nodes/status`
- unrelated Node conditions and status fields are preserved
- a simulated kubelet status patch does not remove the lifecycle conditions
- updates from distinct lifecycle actors follow last-writer-wins semantics
- the dry-run option does not apply conditions
- authorization failures for apply conditions remain non-fatal to the drain helper

##### e2e tests

An e2e test will drain a Node with reporting enabled and verify the start and
completed conditions. A second test will interrupt an active drain and verify
that both conditions become `False` with reason `KubectlDrainInterrupted`.

### Graduation Criteria

This feature is part of the [Specialized Lifecycle Management](https://github.com/kubernetes/enhancements/issues/5683) graduation.
Adding this feature will be the v1alpha2 milestone.

#### Alpha

- Add the opt-in `--report-node-conditions` flag.
- Publish the transition table defined by this KEP.
- Keep condition reporting disabled by default in the drain library.
- Document the additional `nodes/status` permission.
- Add unit, integration, and initial e2e coverage.

#### Beta

- Enable condition reporting by default in the kubectl command.
- Allow users to disable reporting with
  `--report-node-conditions=false`.
- Keep condition reporting opt-in for consumers of the drain library.
- Gather feedback from administrators and lifecycle controller authors.
- Resolve all known issues with stale state, concurrent writers, and selected
  drain criteria.
- Complete all functional, security, monitoring, and testing requirements.

#### GA

- Allow at least two releases for feedback after beta.
- Resolve all issues identified during beta.
- Document production use by lifecycle condition consumers.

#### Deprecation

This KEP does not deprecate existing behavior. Any future removal of the
`--report-node-conditions` opt-out will follow the kubectl flag deprecation
policy.

### Upgrade / Downgrade Strategy

During alpha, this feature is additive and opt-in. Upgrading kubectl adds the
flag but does not change existing invocations. Starting in beta, upgrading
kubectl enables condition reporting by default; users can retain the previous
behavior with `--report-node-conditions=false`. Downgrading to a kubectl
version without the feature stops condition reporting but does not otherwise
affect drain.

Conditions already stored on Nodes remain after downgrade. An administrator
may clear them by updating or removing the two condition entries. Downgrade
does not automatically infer whether the recorded state is still valid.

### Version Skew Strategy

A kubectl version that supports the flag can publish the conditions to any
supported API server when the user has `nodes/status` patch permission.

Older kubectl versions and other drain implementations may not publish these
conditions. Consumers must treat an absent condition as "no drain observation
was reported", not as proof that the Node is not being drained.

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

###### How can this feature be enabled / disabled in a live cluster?

- [x] Other
  - During alpha, the user enables reporting with `kubectl drain
    --report-node-conditions`
  - Reporting is disabled by omitting the flag
  - No control-plane or Node restart is required

###### Does enabling the feature change any default behavior?

No. The alpha behavior is opt-in. When enabled, it adds best-effort Node status
writes without changing Pod selection or eviction behavior.

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

Yes. Stop passing `--report-node-conditions` or use a kubectl version without
the feature during alpha. Once reporting defaults to enabled, pass
`--report-node-conditions=false`. Existing conditions remain until an
authorized administrator clears them.

###### What happens if we reenable the feature if it was previously rolled back?

The next drain invocation updates the two drain condition entries
according to the transition table.

###### Are there any tests for feature enablement/disablement?

Unit and e2e tests will verify both the enabled and disabled command paths.

### Rollout, Upgrade and Rollback Planning

###### How can a rollout or rollback fail? Can it impact already running workloads?

A status write can be forbidden, rejected by admission, conflict, or time out.
The command warns and continues its existing drain behavior. The conditions
can be missing or stale, but the reporting path does not directly modify
running workloads.

Consumers could incorrectly assign behavioral meaning to advisory conditions.
Documentation will require consumers to handle absence, staleness, and
conflicting writers.

###### What specific metrics should inform a rollback?

Operators should inspect API server request failures and latency for `PATCH`
requests to `nodes/status`, audit events for the requesting user and kubectl
user agent, and reports of stale lifecycle conditions.

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

This will be tested before beta.

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

No.

### Monitoring Requirements

###### How can an operator determine if the feature is in use by workloads?

Operators can inspect Nodes for `DrainInProgress` or `Drained` conditions with
a `KubectlDrain*` reason. API audit logs can identify writes to
`nodes/status` by the requesting user and kubectl user agent.

###### How can someone using this feature know that it is working for their instance?

- [x] Node condtions are set and have the correct reasons
  - Condition name: `DrainInProgress`
  - Condition name: `Drained`

###### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

Condition reporting must not reduce the success rate of drain operations.
When the API server accepts status updates, the terminal condition update
should complete within the normal API request timeout after the drain reaches
an outcome.

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

- [x] Other
  - API server request success rate and latency for Node status updates.
  - Age and reason of the lifecycle conditions on affected Nodes.

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

A count of drains that failed to publish conditions would be useful, but a
short-lived CLI does not expose a metrics endpoint. For alpha, stderr output,
conditions, and API audit logs are used.

### Dependencies

###### Does this feature depend on any specific services running in the cluster?

It depends only on the Kubernetes API server and authorization for the Node
status update. If the API server is unavailable, condition reporting fails and
the drain operation follows its existing API failure behavior.

### Scalability

###### Will enabling / using this feature result in any new API calls?

Yes. Each Node normally receives two additional patch requests: one when
processing starts and one on completion. A handled failure or interrupt also
receives a terminal patch request instead of the completion request. Retries
can add a bounded number of requests.

The calls originate from kubectl and target `nodes/status`. Their rate is
bounded by user-initiated drain operations and the number of selected Nodes.

###### Will enabling / using this feature result in introducing new API types?

No.

###### Will enabling / using this feature result in any new calls to the cloud provider?

No.

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

Yes. A Node that did not already contain the conditions gains two
`NodeCondition` entries.

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

No control-plane SLI is expected to change. A drain can make two additional
status requests per Node, but reporting failures do not block the drain beyond
bounded request timeouts and retries.

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

No. The operation is user initiated, does not add a watch or controller, and
stores two entries on each affected Node.

###### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?

No. The number of condition types is fixed and retries are bounded.

### Troubleshooting

###### How does this feature react if the API server and/or etcd is unavailable?

The condition update fails. Kubectl prints the reporting error. If the API
outage also prevents Pod eviction or observation, the drain fails through its
existing path.

###### What are other known failure modes?

- `DrainInProgress=True` remains after the client disappears
  - Detection: inspect `lastHeartbeatTime`, `lastTransitionTime`, active
    administration, and Pods remaining on the Node
  - Mitigation: an authorized administrator decides to clear or replace
    the condition
  - Testing: handled interrupts are tested; uncatchable process and host
    failure are not automatically recoverable
- The user lacks `nodes/status` permission
  - Detection: kubectl prints a forbidden warning and audit logs record the
    denied request
  - Mitigation: grant the permission or run without reporting
  - Testing: unit and integration tests cover forbidden updates
- Concurrent writers publish conflicting observations
  - Detection: inspect condition timestamps, reasons, messages, and audit logs
  - Mitigation: stop concurrent drain actors and have an administrator publish
    the observed state
  - Testing: integration tests cover competing condition updates

###### What steps should be taken if SLOs are not being met to determine the problem?

Inspect the kubectl warning, API server request latency and errors,
authorization and admission decisions, audit records for `nodes/status`, and
the current condition values. Confirm whether the drain itself succeeded
independently of condition reporting.

## Implementation History

- 2026-07-20: Enhancement issue opened.
- 2026-08-07: Initial provisional KEP draft.

## Drawbacks

The proposal adds status-writing behavior to a client-side command that cannot
guarantee cleanup after it disappears. The signal is useful for observability,
but it is weaker than a reconciled server-side operation so it must not be
treated as a lock or durable transaction.

## Alternatives

### Infer drain state from existing signals

Observers can inspect `spec.unschedulable`, taints, and Pods. A cordon does not
mean a drain is active, and independent observers can select different Pods or
reach different completion results. This stil has the interoperability problem.

### Publish annotations or labels

Kubectl could write a Node annotation or label using the same Node permission
it already needs for cordon. This avoids the additional status permission, but
continues the ad hoc signaling pattern that KEP-5683 replaced. It also mixes
observed lifecycle state with user metadata.

### Use a taint as the drain signal

A taint is mean effect scheduling or eviction policy. Drain progress is an
observation. Coupling the two would not implement the ideal behavior.

## Infrastructure Needed (Optional)

No new project infrastructure is required.
