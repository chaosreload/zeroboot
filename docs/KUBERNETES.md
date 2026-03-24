# Running Zeroboot on Kubernetes

Zeroboot can be deployed as a stateful service inside a Kubernetes cluster.
This guide covers node requirements, KVM device access, persistent storage for
snapshots, reference manifests, and autoscaling.

---

## Architecture overview

```
Internet → K8s Service
               │
        ┌──────┼──────┐
        │      │      │
     Pod-1  Pod-2  Pod-3        ← one Pod per KVM-capable Node (podAntiAffinity)
        │      │      │
     VM VM  VM VM  VM VM        ← KVM forks happen inside the Pod, sub-millisecond
```

**Key point:** Kubernetes manages the lifecycle of the zeroboot *server* process.
It does not schedule individual sandboxes — each `v1/exec` request is handled
entirely within the Pod that receives it via a KVM fork (~0.8 ms). Kubernetes'
role is capacity management: health checks, rolling updates, and horizontal scaling.

---

## Node requirements

### Instance families with KVM support

Not all EC2 instance types expose `/dev/kvm`. The following families support KVM
and are suitable for zeroboot:

| Family | KVM method | Notes |
|---|---|---|
| `c8i`, `m8i`, `r8i` | ✅ **Nested virtualization** | **Recommended** — Intel 8th-gen Nitro platform; supports nested virt without metal. Enable at launch via `--cpu-options NestedVirtualization=enabled` (requires AWS CLI ≥ v2.34) |
| `c6i`, `c6a`, `c7i`, `m6i`, `m7i`, `r6i`, `r7i` | ✅ Bare-metal only | KVM available only on `.metal` sizes (e.g. `c6i.metal`) |
| `c5`, `m5`, `r5` | ✅ Bare-metal only | Older Nitro generation; `.metal` sizes only |
| `t3`, `t4g` | ❌ Not available | Burstable — `/dev/kvm` not exposed |
| `t2` | ❌ Not available | No Nitro, no KVM |
| Any ARM (`*g`) | ❌ Architecture mismatch | Firecracker x86_64 binary required |

**TL;DR for EKS node groups:** Use `c8i`, `m8i`, or `r8i` with nested virtualization
enabled — these are the only non-metal families where regular (non-`.metal`) instance
sizes expose `/dev/kvm`. All other families require `.metal` sizes which are significantly
more expensive and harder to schedule in K8s.

```bash
# Enable nested virtualization when launching a new instance (c8i/m8i/r8i only)
aws ec2 run-instances \
  --instance-type c8i.xlarge \
  --cpu-options "NestedVirtualization=enabled" \
  ...
```

> On GCP: `n2`, `n2d`, `c2`, `c3` families support KVM.
> On Azure: `Dv3`, `Ev3`, `Dsv3` with nested virtualization enabled.

### Label KVM-capable nodes

```bash
kubectl label node <node-name> kvm-capable=true
```

The Deployment's `nodeSelector` uses this label to ensure Pods are only scheduled
where `/dev/kvm` is available.

---

## KVM device access without `privileged: true`

Pods request `/dev/kvm` via the [KVM device plugin](https://github.com/kubevirt/kubevirt/tree/main/cmd/virt-handler)
from the KubeVirt project:

```bash
# Install KVM device plugin (DaemonSet)
kubectl apply -f https://github.com/kubevirt/kubevirt/releases/latest/download/kubevirt-operator.yaml
kubectl apply -f https://github.com/kubevirt/kubevirt/releases/latest/download/kubevirt-cr.yaml
```

Once installed, Pods can request KVM access via resources (already set in `deployment.yaml`):

```yaml
resources:
  limits:
    devices.kubevirt.io/kvm: "1"
```

This grants `/dev/kvm` access without `privileged: true` or `hostDevice` mounts.

---

## Persistent storage for snapshots

Zeroboot's `template` command snapshots ~512 MB of VM memory to disk. Without a
PersistentVolume, every Pod restart triggers a ~15 s re-snapshot.

Mount a PVC at `/var/lib/zeroboot` (see `deploy/k8s/pvc.yaml`). The directory
layout on the volume:

```
/var/lib/zeroboot/
├── vmlinux-fc          ← kernel binary (~21 MB)
├── rootfs-python.ext4  ← base rootfs image (pre-loaded numpy/pandas)
├── python/             ← snapshot created by entrypoint on first boot
│   ├── snapshot/
│   │   ├── vmstate     ← CPU register state (~14 KB)
│   │   └── mem         ← 512 MB memory image (CoW source)
│   └── rootfs_path
└── api_keys.json       ← optional API key list
```

> **Populate the volume before first deploy.** Copy `vmlinux-fc` and
> `rootfs-python.ext4` to the PVC (e.g., via a one-shot init Job or manual
> `kubectl cp`). The entrypoint will create the snapshot automatically on
> first boot if it is missing.

### Storage class recommendations

| Cloud | StorageClass | Notes |
|---|---|---|
| AWS | `gp3` | Default for EKS; good random-read IOPS for CoW page faults |
| GCP | `premium-rwo` | SSD-backed, low latency |
| Azure | `managed-premium` | SSD, required for sub-ms fork performance |

Avoid `gp2` or spinning-disk storage classes — the CoW page fault path is
latency-sensitive and benefits from SSD IOPS.

---

## Deploying

```bash
# 1. Create namespace
kubectl apply -f deploy/k8s/namespace.yaml

# 2. Create PVC
kubectl apply -f deploy/k8s/pvc.yaml

# 3. Deploy (2 replicas by default)
kubectl apply -f deploy/k8s/deployment.yaml
kubectl apply -f deploy/k8s/service.yaml

# 4. Watch rollout — first boot takes ~30s for template creation
kubectl rollout status deployment/zeroboot -n zeroboot

# 5. Verify
kubectl exec -n zeroboot deploy/zeroboot -- curl -s localhost:8080/v1/health
```

---

## Autoscaling

### Why not CPU-based HPA?

Zeroboot workloads are **memory-bound**, not CPU-bound. Each concurrent fork
adds ~265 KB of CoW memory pressure. CPU utilization is a poor scaling signal.

### Custom metric HPA

The `zeroboot_concurrent_forks` gauge (exposed at `/v1/metrics`) reflects the
number of active VM sandboxes per Pod. Use this for HPA:

```bash
# Apply HPA (requires prometheus-adapter, see below)
kubectl apply -f deploy/k8s/hpa.yaml
```

Scale-out triggers when average concurrent forks per Pod exceeds 800. Adjust
this threshold based on your Node's available memory:

```
max_concurrent_forks ≈ (node_memory - 2GB_overhead) / 265KB_per_fork
# Example: 8GB node → (8192 - 2048) / 0.265 ≈ 23,000 theoretical max
# Practical limit with snapshot RSS: ~1000–2000 per Pod
```

### Exposing the metric via prometheus-adapter

Add to your `prometheus-adapter` ConfigMap:

```yaml
rules:
  - seriesQuery: 'zeroboot_concurrent_forks{namespace!="",pod!=""}'
    resources:
      overrides:
        namespace: {resource: "namespace"}
        pod: {resource: "pod"}
    name:
      matches: "zeroboot_concurrent_forks"
      as: "zeroboot_concurrent_forks"
    metricsQuery: 'avg_over_time(zeroboot_concurrent_forks{<<.LabelMatchers>>}[1m])'
```

### Karpenter node provisioning

For cluster autoscaling with Karpenter, create a NodePool that targets KVM-capable instances:

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: zeroboot-kvm
spec:
  template:
    metadata:
      labels:
        kvm-capable: "true"
    spec:
      requirements:
        - key: karpenter.k8s.aws/instance-family
          operator: In
          values: [c6i, c7i, c8i, m6i, m7i]
        - key: karpenter.k8s.aws/instance-size
          operator: In
          values: [xlarge, 2xlarge, 4xlarge]
        - key: kubernetes.io/arch
          operator: In
          values: [amd64]
  limits:
    cpu: 100
```

> **Scaling latency note:** Karpenter takes 60–120 s to provision a new KVM
> node (EC2 start + kubelet join + Pod scheduling + snapshot load). Karpenter
> handles **capacity expansion** for sustained load — it is not designed to
> absorb sudden request spikes. Size your warm pool (`minReplicas`) to handle
> peak burst traffic; use HPA to scale within the existing node pool first.

---

## Monitoring

Zeroboot exposes Prometheus metrics at `/v1/metrics` (not `/metrics`).

### ServiceMonitor (Prometheus Operator)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: zeroboot
  namespace: zeroboot
spec:
  selector:
    matchLabels:
      app: zeroboot
  endpoints:
    - port: http
      path: /v1/metrics
      interval: 15s
```

### Key metrics

| Metric | Type | Description |
|---|---|---|
| `zeroboot_concurrent_forks` | gauge | Active VM sandboxes — **use for HPA** |
| `zeroboot_fork_time_milliseconds` | histogram | Fork latency (P50/P99) |
| `zeroboot_exec_time_milliseconds` | histogram | Code execution latency |
| `zeroboot_total_time_milliseconds` | histogram | End-to-end request latency |
| `zeroboot_total_executions{status}` | counter | Success / error / timeout counts |
| `zeroboot_memory_usage_bytes` | gauge | Process RSS — monitor for memory pressure |

---

## Configuration reference

All configuration is via environment variables (set in `deployment.yaml`):

| Variable | Default | Description |
|---|---|---|
| `ZEROBOOT_WORKDIR` | `/var/lib/zeroboot` | Working directory (PVC mount point) |
| `ZEROBOOT_KERNEL` | `$WORKDIR/vmlinux-fc` | Path to kernel binary |
| `ZEROBOOT_ROOTFS_PYTHON` | `$WORKDIR/rootfs-python.ext4` | Python rootfs image |
| `ZEROBOOT_ROOTFS_NODE` | _(unset)_ | Node.js rootfs image (optional) |
| `ZEROBOOT_PORT` | `8080` | API server port |
| `ZEROBOOT_TEMPLATE_WAIT` | `15` | Seconds to wait during template snapshot |
| `ZEROBOOT_API_KEYS_FILE` | _(unset)_ | Path to JSON array of API keys |

---

## Limitations

- **Single-node fork pool:** All sandboxes on a Pod run on the same physical Node.
  Scale out by adding Pods (and Nodes), not by resizing individual Pods.
- **ReadWriteOnce PVC:** Each Pod needs its own PVC (`ReadWriteOnce`). If you
  use a `StatefulSet` instead of a `Deployment`, each replica gets its own PVC
  automatically via `volumeClaimTemplates`.
- **Snapshot on first boot:** The first Pod startup after PVC creation takes
  ~15–30 s while the template snapshot is created. Subsequent restarts are fast
  (~2 s) because the snapshot is persisted on the PVC.
- **x86_64 only:** Firecracker and the guest kernel are x86_64. ARM nodes are
  not supported.
