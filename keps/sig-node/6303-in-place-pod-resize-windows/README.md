# KEP-6303: In-place Pod Vertical Scaling (Container Resize) on Windows

> **AI assistance disclosure:** This KEP draft was written with the assistance of an AI coding
> agent. The human author (github.com/MartinForReal) is fully responsible for the content and
> for shepherding this proposal through the Kubernetes enhancement process. Per the contributor
> guidelines, AI use is disclosed here and in the PR description, and the author verifies every
> design decision before it is committed.

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
    - [Story 1: VPA in-place mode on a Windows StatefulSet](#story-1-vpa-in-place-mode-on-a-windows-statefulset)
    - [Story 2: One feature gate across mixed-OS node pools](#story-2-one-feature-gate-across-mixed-os-node-pools)
  - [Notes/Constraints/Caveats](#notesconstraintscaveats)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Kubelet Gating Changes](#kubelet-gating-changes)
  - [CRI Resource Update for Windows Containers](#cri-resource-update-for-windows-containers)
  - [Working-Set vs Commit Memory Semantics](#working-set-vs-commit-memory-semantics)
  - [CPU Resource Update](#cpu-resource-update)
  - [Pod-Level Resources](#pod-level-resources)
  - [Test Plan](#test-plan)
    - [Unit tests](#unit-tests)
    - [Integration tests](#integration-tests)
    - [e2e tests (Windows)](#e2e-tests-windows)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha](#alpha)
    - [Beta](#beta)
    - [GA](#ga)
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
- [Infrastructure Needed (Optional)](#infrastructure-needed-optional)
<!-- /toc -->

## Release Signoff Checklist

- [ ] (R) Enhancement issue in release milestone, which links to KEP dir in kubernetes/enhancements
- [ ] (R) KEP approvers have approved the KEP status as implementable
- [ ] (R) Design details are appropriately documented
- [ ] (R) Test plan is in place
  - [ ] e2e Tests for all Beta API Operations (endpoints)
  - [ ] (R) Ensure GA e2e tests meet requirements for Conformance Tests
  - [ ] (R) Minimum Two Week Window for GA e2e tests to prove flake free
- [ ] (R) Graduation criteria is in place
- [ ] (R) Production readiness review completed
- [ ] (R) Production readiness review approved
- [ ] Implementation History section is up-to-date for milestone
- [ ] User-facing documentation has been created in kubernetes/website

[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/website]: https://git.k8s.io/website

## Summary

In-place pod vertical scaling (the InPlacePodVerticalScaling feature gate, defined in
KEP-1287, see keps/sig-node/1287-in-place-update-pod-resources) is stable on Linux, but the
kubelet hard-rejects resource resize on Windows nodes. Every resize attempt on Windows is
refused with the message "In-place pod resize is not supported on Windows" by the hand-written
per-OS gate in pkg/kubelet/allocation/features_windows.go (functions
IsInPlacePodVerticalScalingAllowed and IsInPlacePodLevelResourcesVerticalScalingAllowed).

This KEP removes that gap. It makes the Windows kubelet honor the same feature-gate path used
on Linux and implements the CRI UpdateContainerResources / UpdatePodSandbox resource-update
path for Windows containers (Host Compute Service / HCS job objects). The PodSpec Resources API
is already GA; no API change is required. The change is confined to the kubelet and the Windows
container runtimes; containerd needs no CRI schema change for the initial scope, only its
existing live-update path run on Windows.

## Motivation

Windows workers are excluded from in-place pod vertical scaling solely because the kubelet
hard-rejects every resize through a hand-written per-OS gate (pkg/kubelet/allocation/
features_windows.go). The underlying runtime (containerd on Windows, backed by HCS job
objects) already supports updating live resource limits for containers, and KEP-1287
explicitly intended UpdateContainerResources to work for Windows:

> "Modify UpdateContainerResources to allow it to work for Windows Containers, as well as
> Containers managed by other runtimes besides Linux." -- KEP-1287

That goal was never wired into the Windows kubelet. The parity gap is user-visible: Windows
workloads cannot apply CPU or memory resize to a running container without Pod recreation
and container restarts. This KEP closes that gap. Its primary motivations are:

- **On-line CPU vertical scaling.** Raise or lower a running Windows container's CPU count /
  quota (`cpuMaximum`, shares) without restarting it. Vertical pod autoscalers and operators can
  right-size CPU in place on Windows exactly as on Linux.
- **Commit-ceiling memory resize without container restart.** A Windows memory limit is
  not a kill: the HCS job object enforces a working-set limit (soft; induces working-set
  trimming) plus a commit ceiling (hard; surfaced as an allocation failure). Unlike Linux
  there is **no native OOM killer** that terminates the container. Changing
  `resources.memory.limit` in place moves that commit ceiling and working-set limit
  without a restart, so operators can adjust memory headroom for in-place-growing workloads
  such as .NET or Go services without dropping connections.
- **Resource and QoS consistency.** Keeping accounting, the scheduler, and the eviction manager
  in step with the limits actually enforced on the Windows node avoids drift between what is
  scheduled and what is applied, and keeps QoS semantics credible for Windows pods.
- **Operational convenience.** VPA in-place updates, StatefulSet resizes, and Job autoscaling
  become usable on Windows node pools, removing the recreate-and-restart tax and the resulting
  connection and state churn.

The enabler is unchanged from Linux: the CRI `UpdateContainerResources` (and the
pod-sandbox-level call) plus the existing feature-gate path (`InPlacePodVerticalScaling` /
`InPlacePodLevelResourcesVerticalScaling`), now honored by the
Windows kubelet. How the commit ceiling and memory limit map to HCS job objects is
detailed in Design Details.

### Goals

- Enable container-level in-place vertical scaling (resize of cpu and memory requests/limits)
  on Windows nodes, matching Linux behavior, without recreating the Pod or restarting the
  container.
- Provide on-line CPU vertical scaling by mapping a container's CPU request/limits to the
  Windows CPU-share/affinity model through `UpdateContainerResources`.
- Provide in-place memory sizing on the commit ceiling: lower or raise the working-set limit
  and commit ceiling enforced by the Windows runtime without container restart, documented as
  distinct from Linux cgroup `memory.max` enforcement (no native OOM kill on Windows).
- Make the Windows kubelet honor the `InPlacePodVerticalScaling` feature-gate path instead
  of hard-rejecting resize.
- Resolve CRI `UpdateContainerResources` / `UpdatePodSandbox` on Windows so accounting,
  scheduler, and eviction agree with the enforced limits.
- Keep the KEP-1287 CRI contract so no breaking CRI change is introduced in the container-level
  scope.

### Non-Goals

- Changing the PodSpec Resources API or QoS-class semantics.
- In-place vertical scaling for Hyper-V isolated pods in the Alpha milestone (initial scope
  targets process-isolated Windows containers).
- Any change to the Linux path.
- Reproducing Linux-memcg OOM-kill semantics on Windows, or full "OOM parity" between
  working-set trimming and cgroup `memory.max`: Windows enforces working-set trim
  plus commit-ceiling allocation failure (no native kill), and that divergence is documented
  rather than hidden.
- Pod-level-resource resize on Windows in Alpha; it follows in the Beta milestone.

## Proposal

### User Stories

#### Story 1: VPA in-place mode on a Windows StatefulSet

A Windows-hosted .NET workload is scaled by the Vertical Pod Autoscaler in in-place mode. Today
every resource update forces a Pod recreation, dropping connections and buffered writes. After
this KEP the VPA can resize requests and limits live without restarting containers, matching
the Linux experience.

#### Story 2: One feature gate across mixed-OS node pools

A cluster runs both Linux and Windows node pools behind the same InPlacePodVerticalScaling gate.
After this KEP the Windows node pool no longer silently refuses resize, so autoscaling policies
and operators become portable across OS pools.

### Notes/Constraints/Caveats

- Windows has no cgroups. Resource enforcement uses the HCS job object exposed through the
  runtime (containerd); the kubelet abstracts it behind the CRI update path.
- Memory limits are applied as working-set limits on Windows, not Linux-style commit limits.
  This can surface different OOM behavior and is recorded rather than hidden (see Design Details).
- The feature gates InPlacePodVerticalScaling (and the pod-level variant) already exist; no new
  feature gate is introduced in the Alpha milestone.

### Risks and Mitigations

- **Risk:** runtimes may not expose consistent live-update semantics across Windows Server
  versions.
  **Mitigation:** probe runtime capability at kubelet start and at resize; fail closed with a
  clear reason when the runtime reports no live-update support.
- **Risk:** users assume working-set memory limits behave like memcg commit limits and are
  surprised by OOM divergence.
  **Mitigation:** document the mapping, emit an event when a working-set limit is applied, and
  log the divergence so operators can plan capacity.
- **Risk:** scope creep into adjacent Windows parity items.
  **Mitigation:** keep strict non-goals; pod-level resources and OOM observability are fenced to
  their own follow-ups.

## Design Details

### Kubelet Gating Changes

On Windows the kubelet currently bypasses feature-gate checks and hard-rejects every resize. The
change replaces pkg/kubelet/allocation/features_windows.go so that it mirrors the Linux
behavior: gate on features.InPlacePodVerticalScaling (and the pod-level variant) instead of
returning false unconditionally, and reflect the runtime actual capability when known.

Sketch of the post-change Windows gate:

    // pkg/kubelet/allocation/features_windows.go (after)
    func IsInPlacePodVerticalScalingAllowed(_ *v1.Pod) (bool, string, string) {
        if !utilfeature.DefaultFeatureGate.Enabled(features.InPlacePodVerticalScaling) {
            return false, "InPlacePodVerticalScaling is disabled", "feature_gate_off"
        }
        return true, "", ""
    }

This is the minimal kubelet change; the real parity lives in the runtime update path.


### CRI Resource Update for Windows Containers

Keep using the existing UpdateContainerResources CRI call (and the pod-sandbox level call used
when pod-level resources change). The CRI spec already carries a Windows resource message with
fields for CPU shares/maximum and memory limits plus a pod-level Windows section. The kubelet
must:

1. populate that message from the requested cpu and memory resources,
2. invoke the existing UpdateContainerResources call, and
3. surface the runtime response; an unsupported response becomes a resize failure with an event
   reason rather than a hard gate rejection.

No CRI API addition is strictly required for the container-resource scope. If a follow-up
(pod-level or runtime-specific) needs a new field it will be a separate CRI change, not a new
kubelet contract.

### Working-Set vs Commit Memory Semantics

On Windows, memory limits control the container working set through the HCS compute-system
memory limit (job object). This differs from Linux cgroup v2 memory.max. The implementation
will:

- preserve the Linux meaning of resources.memory.limit,
- apply it as the job-object working-set limit on Windows,
- record an event when the working-set limit is applied so operators understand the divergence,
- document the OOM divergence in user-facing docs.

### CPU Resource Update

Windows CPU sizing uses a share model rather than Linux quota/periods. The implementation converts
container cpu requests and limits to the Windows CPU share value and submits it through
UpdateContainerResources. Where the CPU affinity gate is enabled on Windows nodes, the resize
path must stay consistent with the affinity engine; the WindowsCPUAndMemoryAffinity gate itself
is out of scope here.

### Pod-Level Resources

Pod-level in-place resize is supported only when restartPolicy is Always and containers can be
recreated. On Windows this is implemented by recreating the affected containers with the updated
resources through the existing CRI create path. This milestone wires the container-level path and
records a planned follow-up (section update plus implementation) for pod-level resize.

### Test Plan

[x] I/we understand the owners of the involved components may require updates to existing tests.

#### Unit tests
- pkg/kubelet/allocation: Windows build assertions that the feature-gate path is honored instead
  of the current unconditional rejection.
- pkg/kubelet/kuberuntime: tests mapping requests and limits to Windows CPU shares and memory
  working-set values.

#### Integration tests
- test/integration/kubelet: verify a Windows node honors the resize flow and reports success
  without a restart.
- test/integration/controlplane: confirm the API surface is unchanged (no control-plane impact).

#### e2e tests (Windows)
- [sig-windows] InPlacePodVerticalScaling: increase and decrease CPU and memory on a Windows
  container without restart.
- Process-isolated Windows containers are the initial scope; the Hyper-V case is a follow-up.
- Run in the periodic Windows conformance jobs with a stability window before alpha.

### Graduation Criteria

#### Alpha
- Windows kubelet honors the feature gate (no hard rejection).
- Container-level CPU and memory resize works for a Windows process-isolated container.
- A dedicated e2e test runs on a Windows node with no flakes for a two-week window.
- No known memory limit under- or over-allocation gaps remain open.

#### Beta
- Pod-level (Always) in-place resize works on Windows.
- CPU and memory parity with Linux is documented and tested (shares and working-set limit).
- Observability: metrics, events, and troubleshooting docs complete.

#### GA
- Conformance e2e for Windows in-place resize is present and passing.
- A maintained Windows CI job proves stability for more than two weeks.
- Only the existing feature gates remain; no per-OS default differences.

### Upgrade / Downgrade Strategy

No API changes; only kubelet code behind existing feature gates. Rolling upgrade of Windows
nodes: an old kubelet continues to refuse resize (current behavior) with a clear reason, and a
new kubelet accepts when the gate is on. Downgrade restores the hard rejection; no on-disk
state is introduced beyond existing pod status fields.

### Version Skew Strategy

The kubelet gate is per-node; the API server and scheduler are unchanged. Version skew between
the kubelet and the runtime (containerd) is handled by a runtime capability probe: if the runtime
cannot perform a live update, the kubelet fails the resize with an event, so old and new runtime
pairings degrade gracefully on either OS.


## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

- Enabled/disabled with the existing InPlacePodVerticalScaling feature gate on the kubelet;
  when disabled the Windows kubelet mirrors the current behavior (rejects resize).
- No new API object is introduced; the feature is runtime-capability facing.
- Disabling causes no effect beyond refusing resize; no data is mutated.

### Rollout, Upgrade and Rollback Planning

- The change ships in the kubelet binary (Windows); the API server and scheduler are unaffected.
- Rollback is a gate flip or a kubelet downgrade; no migration is needed.
- The e2e must run against every release in the Windows CI.

### Monitoring Requirements

- Collect kubelet_inplace_pod_resize_total with a label for the OS and outcome, so operators can
  observe accepted vs refused resize on Windows nodes.
- Emit a kubelet event reason (WindowsWorkingSetLimitApplied) when a working-set limit is
  applied, which assists OOM-path debugging.

### Dependencies

- k8s.io/cri-api (unchanged for the container scope); containerd with an existing Windows
  UpdateContainerResources implementation. No new third-party dependencies.

### Scalability

- No new API objects or control-plane channels. Per-node resize calls are the same as Linux; the
  existing kubelet rate limit bounds call volume.

### Troubleshooting

- Symptom: resize is refused on a Windows node even when the gate is on.
  - Check kubelet events for the reason indicating live updates are unsupported by the runtime.
  - Check kubelet and containerd logs for the UpdateContainerResources failure details.
  - Confirm the InPlacePodVerticalScaling feature gate is enabled on the node.

## Implementation History

- 2026-08-21: Initial provisional draft submitted. Authored with AI assistance; the human author
  remains responsible for the content.
- Tracking issue: kubernetes/enhancements#6303 (this is the KEP number).

## Drawbacks

- Working-set vs commit memory semantics add platform-specific behavior that needs clear
  documentation so users are not surprised.
- A runtime capability-probe path must be kept in parity with the Linux path over time.

## Alternatives

- **Just remove the rejection in features_windows.go:** unsafe without runtime capability
  support; KEP-1287 requires capability detection. Rejected as incomplete.
- **Extend KEP-1287 in place:** it is already marked implemented and stable; the remaining
  Windows work is best tracked as a new scoped KEP with clear reviewers, linking to KEP-1287 via
  see-also.
- **Candidate B (graduation of WindowsCPUAndMemoryAffinity):** a graduation-only change within
  an existing enhancement, not a new parity gap; out of scope here and tracked separately.

## Infrastructure Needed (Optional)

A Windows CI job that runs the new in-place resize e2e persistently in the sig-windows periodic
suite; existing jobs may need a new profile entry.
