---
name: k8s-csi-stale-mount-recovery
description: >
  DETERMINISTIC runbook for a Kubernetes pod stuck in CreateContainerError /
  ImagePullBackOff caused by a stale CSI mount ("host is down") and/or a
  digest-pinned image that was garbage-collected upstream. Activate when a
  workload is down and kubectl shows CreateContainerError with a message like
  `stat .../volumes/kubernetes.io~csi/...: host is down`, or ImagePullBackOff
  with `NotFound`. Kubernetes/containerd/CSI environment only.
license: MIT
metadata:
  author: hermes-incident-commander
  version: "1.0"
  origin_incident: forgejo-down-2026-07-13 (SMB storagebox stale mount + GC'd experimental image)
  tags: [kubernetes, csi, smb, cifs, longhorn, image-cache, deterministic-runbook]
---

# K8s CSI stale-mount + pinned-image recovery (DETERMINISTIC)

This is a deterministic runbook, not free-form reasoning. Follow the numbered
gates in order. Each REMEDIATE step names its safety tier. **Never reschedule a
pod off its current node until the image-pullability gate (step 3) passes** —
doing so is what turns a 2-minute mount fix into a multi-node outage.

## 0. SIGNATURE (when this applies)
- `kubectl get pod` shows `CreateContainerError` or `ImagePullBackOff`.
- Describe/events contain EITHER:
  - `failed to stat ".../volumes/kubernetes.io~csi/pvc-<id>/mount": host is down`  (stale CSI mount), OR
  - `failed to resolve reference "<img>@sha256:<digest>": ... not found`  (GC'd pinned image).
- Often both: a stale mount blocks the pod; a naive pod-delete reschedules it to
  a node lacking the (now-GC'd) image → ImagePullBackOff.

## 1. TRIAGE
- P0 if a singleton critical service (forge, auth, registry, DB) is fully down.
- Record: pod, node, namespace, the PVC id, the image ref, the failing container.

## 2. DIAGNOSE — stale mount vs backing-store-down (deterministic)
```bash
POD=<pod>; NS=<ns>
kubectl describe pod -n $NS $POD | grep -A3 -iE 'host is down|ErrImagePull|Multi-Attach'
# Identify the CSI volume + backing host:
PVC=$(kubectl get pod -n $NS $POD -o jsonpath='{.spec.volumes[*].persistentVolumeClaim.claimName}')
kubectl get pv $(kubectl get pvc -n $NS <claim> -o jsonpath='{.spec.volumeName}') \
  -o jsonpath='{.spec.csi.driver}{"  "}{.spec.csi.volumeAttributes.source}{"\n"}'
```
- **Is the backing store actually reachable?** (decides fix path — do NOT skip)
  ```bash
  # e.g. SMB storagebox: test 445; NFS: 2049; check the host from a node/devbox
  timeout 6 bash -c 'cat </dev/null >/dev/tcp/<store-host>/<port>' && echo UP || echo DOWN
  ```
  - Store DOWN → this runbook can't force a mount; escalate to restore the store
    (Hetzner Storage Box / NFS server / Longhorn node). Do not thrash pods.
  - Store UP → it is a **stale local mount**. Continue.
- Confirm a CSI node-plugin bounce lines up with the outage start:
  ```bash
  kubectl get pods -n kube-system | grep -E 'csi-.*-node'   # look for a RESTART ~ outage start
  ```

## 3. IMAGE-PULLABILITY GATE (do this BEFORE any pod delete)
The pinned image may have been GC'd upstream (esp. `-experimental`/rolling repos).
If so, only nodes that already pulled it can start the pod.
```bash
IMG=$(kubectl get pod -n $NS $POD -o jsonpath='{.spec.containers[0].image}')
# Is the pinned digest still upstream? A kubelet 'NotFound' in events is authoritative.
# Which nodes hold it cached? (the current node always does if it once ran there)
```
- **If the image is GC'd upstream AND cached on only one node:** the pod MUST run
  on that node. Do NOT let it reschedule elsewhere. Go to 4b.
- **If the image is pullable from any node:** a plain reschedule is safe. Go to 4a.

## 4. REMEDIATE (least-destructive first)
### 4a. Image pullable — stale mount only (Tier 2)
- Preferred: clear the stale mount in place (node access) so the SAME pod recovers:
  the CSI driver's `NodeUnstageVolume` cleans the global mount; a fresh
  NodeStage/Publish then succeeds. If in-place clear isn't available, delete the
  pod (it reschedules, image is pullable everywhere → safe).

### 4b. Image GC'd, cached on ONE node `N` (Tier 2, deterministic)
Keep the pod on the cached node; give it a fresh mount:
```bash
# Cordon every OTHER eligible node so the pod can only land on N:
kubectl get nodes -l <the-pod's-required-nodeSelector> -o name   # eligible set
kubectl cordon <each eligible node except N>
kubectl delete pod -n $NS $POD --wait=false   # reschedules to N (cached image, fresh CSI stage)
# watch it reach 1/1 Running on N, then:
kubectl uncordon <the nodes you cordoned>
```
- If a `Multi-Attach` error appears on an RWO data volume, it clears once the old
  pod fully terminates (volume detaches); wait, do not force-detach blindly.

## 5. VERIFY
```bash
kubectl get pod -n $NS -o wide | grep <deploy>       # 1/1 Running on the intended node
curl -s -o /dev/null -w '%{http_code}\n' https://<service>/healthz   # expect 200
```
Ensure every node you cordoned is `Ready`/schedulable again.

## 6. DOCUMENT + LEARN
Write the incident report. Then file the PREVENTION items below as tracked work.

## PREVENTION (root-cause fixes — file these, don't just restart)
1. **Cluster-wide image cache** so a GC'd upstream digest can't strand you on one
   node: deploy **Spegel** (P2P containerd image mirror) AND mirror all pinned
   images into the local registry / FlakeCache. Pin to the mirrored ref.
2. **Get off rolling/experimental image tags.** Pin a STABLE release by digest
   (e.g. Forgejo LTS v15, or v16 once GA) that won't be GC'd, mirrored locally.
3. **Replace fragile remote SMB/CIFS PVCs.** For Forgejo bulk/LFS/attachments,
   use Forgejo's native S3 backend (Garage `s3-eu1`) instead of an SMB
   Storage Box mount — removes the stale-CIFS failure mode entirely.
4. **CSI node-plugin bounce alert** → auto-run step 2's reachability check so the
   commander distinguishes "store down" from "stale mount" without human triage.
