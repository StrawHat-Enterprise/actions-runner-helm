# ARC Runner Scale Set Container Modes: Setup-Verified Guide

## 1. Purpose

This document describes container execution modes for ARC runner scale sets and is aligned to the current deployed behavior in this workspace.

It explains:

- Which modes are currently deployed
- How jobs run in each mode
- Which image is the runner image vs workload image
- When child pods and PVCs are created
- How to verify behavior with kubectl

---

## 2. Scope of this repo (important)

Current dev runner scale sets in this setup:

- straw-hat-runners-linux-dev-dind: dind mode
- straw-hat-runners-linux-dev-kubernetes-pvc: kubernetes mode with PVC
- straw-runners-linux-dev-k8s-novol: dind mode without PVC

Important clarification:

- The scale set named k8s-novol is currently configured as dind without PVC in GitOps.
- It is not currently configured as ARC type kubernetes-novolume.

---

## 3. Core mental model

ARC runs self-hosted runners in Kubernetes.

Two image knobs are different:

- Runner pod image: configured in ARC Helm values (runner template)
- Job workload image: configured by the workflow job pattern

Examples:

- dind jobs use the dind sidecar to pull and run workload images
- kubernetes container jobs use child pod images from workflow container image

---

## 4. Quick comparison (this setup)

| Mode in this setup | Runner pod shape | Child job pod | PVC | How workload image is pulled |
|---|---|---|---|---|
| dind | runner + docker:dind sidecar | No (for standard shell steps) | No | Pulled by docker daemon in dind sidecar |
| kubernetes-pvc | runner + kubernetes hook flow | Yes (for container jobs) | Yes | Pulled by kubelet for child pod |
| k8s-novol (current) | runner + docker:dind sidecar | No (same as dind) | No | Pulled by docker daemon in dind sidecar |

---

## 5. Mode details

### 5.1 dind

Type:

```yaml
containerMode:
  type: dind
```

Behavior:

- ARC creates runner pod with runner container and docker:dind container.
- Workflow shell steps run in the runner container.
- Docker commands talk to dind daemon.
- Workload images are pulled inside dind.

Used for:

- docker build
- docker run
- docker compose

### 5.2 kubernetes-pvc

Type:

```yaml
containerMode:
  type: kubernetes
  kubernetesModeWorkVolumeClaim:
    accessModes:
      - ReadWriteOnce
    storageClassName: managed-csi
    resources:
      requests:
        storage: 10Gi
```

Behavior:

- Runner receives job and uses Kubernetes hooks for container-style execution.
- A child job pod is created when workflow uses container/job container patterns.
- A PVC is created and mounted for workspace sharing.

Used for:

- containerized jobs where Kubernetes child pod execution is desired
- workflows that need shared workspace via PVC

### 5.3 k8s-novol (as currently deployed)

Current behavior in this repo:

- This scale set is configured as dind, not kubernetes-novolume.
- No shared PVC is created.
- No separate Kubernetes child job pod is created for standard shell workflows.
- Additional workload images are pulled by docker engine inside dind.

This is why you can observe:

- runner pod starts with actions-runner and docker:dind images
- docker pull inside the dind container succeeds for additional images (for example alpine)

---

## 6. Image responsibility

### 6.1 Runner image

Configured in Helm values under runner template container image.

Current setup uses public image source for runner:

- ghcr.io/actions/actions-runner:latest

### 6.2 Workload image

Depends on mode:

- dind and current k8s-novol: pulled by docker daemon in dind
- kubernetes-pvc container-style execution: pulled by Kubernetes for child pod

Examples:

- docker run alpine:3.20 in dind mode pulls from docker.io
- container image in workflow for kubernetes mode is pulled for child pod

---

## 7. Verification commands

### 7.1 Check scale sets

```bash
kubectl get autoscalingrunnersets.actions.github.com -n arc-runners
```

### 7.2 Check ephemeral runners

```bash
kubectl get ephemeralrunners.actions.github.com -n arc-runners
```

### 7.3 Check runner pods and images

```bash
kubectl get pods -n arc-runners -o wide
kubectl get pod -n arc-runners <runner-pod> -o jsonpath='{.spec.initContainers[*].image}{"\n"}{.spec.containers[*].image}{"\n"}'
```

### 7.4 Check pvc lifecycle (kubernetes-pvc set)

```bash
kubectl get pvc -n arc-runners -w
kubectl get pods -n arc-runners -w
```

### 7.5 Confirm fresh no-volume spin-up image pulls

```bash
kubectl delete pod -n arc-runners <k8s-novol-runner-pod>
kubectl describe pod -n arc-runners <new-k8s-novol-runner-pod> | sed -n '/Events:/,$p'
```

Expected for current k8s-novol:

- pull events for ghcr.io/actions/actions-runner:latest
- docker:dind container start
- no pvc creation for this set

## 7.6 Workflow tests to run

The following workflow files in InfraCreator are mode-specific and can be run with workflow_dispatch:

- .github/workflows/test-runner-dev-dind.yml
  - validates dind docker daemon path and workload image pull/run
- .github/workflows/test-runner-dev-kubernetes-pvc.yml
  - uses job container image to validate kubernetes child pod flow plus pvc workspace behavior
- .github/workflows/test-runner-dev-k8s-novol.yml
  - validates current k8s-novol behavior as dind without pvc

---

## 8. Common confusion to avoid

1. Runner image and workload image are different concerns.
2. A name containing kubernetes or novol does not guarantee ARC kubernetes-novolume mode.
3. In current repo state, k8s-novol behaves like dind-no-pvc.
4. If job shows Waiting for a runner to pick up this job, validate runner group access at GitHub org/repo level even if pods are healthy.

---

## 9. If true kubernetes-novolume is needed later

To switch from current dind-no-pvc behavior to true ARC kubernetes-novolume behavior:

1. Change containerMode type to kubernetes-novolume in the relevant overlay.
2. Ensure runner image and hooks are compatible with no-volume execution.
3. Test with workflow container job and verify child pod lifecycle.
4. Re-validate filesystem behavior across steps and actions.

---

## 10. Reference links

- ARC overview: https://docs.github.com/en/actions/concepts/runners/actions-runner-controller
- Runner scale sets: https://docs.github.com/en/actions/how-tos/manage-runners/use-actions-runner-controller/deploy-runner-scale-sets
- ARC repository: https://github.com/actions/actions-runner-controller
