# Zeroboot K8s PR Validation Plan

**PR:** chaosreload/zeroboot → feat/kubernetes-deployment → [PR #13](https://github.com/zerobootdev/zeroboot/pull/13)  
**目标：** 在真实 AWS EKS 环境验证 Dockerfile、K8s manifests、eksctl 配置的正确性  
**执行人：** openclaw-research  
**验证完成后：** 将结果反馈给 openclaw-coding，用于完善 PR

---

## 前置条件

- AWS 账号，有 EKS / EC2 / ECR 创建权限
- dev-server 可访问（已有 eksctl 0.224、aws cli、docker、kubectl）
- dev-server 的 IAM role 有足够权限（admin 级别，已确认）

---

## 验证场景

### 场景 A：新建集群 + KVM node group（一步到位）
### 场景 B：先建集群，后追加 KVM node group

两个场景都需要跑，验证 eksctl 配置的正确性。

---

## Step 1：准备代码

```bash
# 在 dev-server 上
cd /data/projects/chaosreload/study/repo/chaosreload/zeroboot
git checkout feat/kubernetes-deployment
git pull origin feat/kubernetes-deployment

# 确认文件结构
ls Dockerfile docker/entrypoint.sh deploy/k8s/ deploy/eks/ docs/KUBERNETES.md
```

**预期输出：** 所有文件存在，无报错

---

## Step 2：构建 Docker 镜像

```bash
# 在 dev-server 上构建
cd /data/projects/chaosreload/study/repo/chaosreload/zeroboot
docker build -t zeroboot:test .
```

**预期：**
- 构建成功，无 error
- 最终镜像大约 300-500 MB
- `docker images | grep zeroboot` 能看到镜像

**记录：**
- [ ] 构建是否成功
- [ ] 构建耗时（大概）
- [ ] 镜像大小

---

## Step 3：推镜像到 ECR

```bash
# 创建 ECR repo（如果不存在）
AWS_REGION=ap-southeast-1
AWS_ACCOUNT=$(aws sts get-caller-identity --query Account --output text)
ECR_REPO="${AWS_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com/zeroboot"

aws ecr create-repository --repository-name zeroboot --region $AWS_REGION 2>/dev/null || true

# Login + push
aws ecr get-login-password --region $AWS_REGION | \
  docker login --username AWS --password-stdin "${AWS_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com"

docker tag zeroboot:test $ECR_REPO:test
docker push $ECR_REPO:test
```

**预期：** push 成功，ECR 里有 `zeroboot:test` 镜像

**记录：**
- [ ] ECR push 是否成功
- [ ] 镜像 URI（后续 deployment.yaml 会用到）

---

## Step 4A：场景 A — 新建集群 + KVM node group

```bash
cd /data/projects/chaosreload/study/repo/chaosreload/zeroboot

# 编辑 region（如需要）
# deploy/eks/eks-with-kvm-nodegroup.yaml 默认 ap-southeast-1

eksctl create cluster -f deploy/eks/eks-with-kvm-nodegroup.yaml
# 预计耗时：15-20 分钟
```

**预期：**
- eksctl 无报错完成
- `kubectl get nodes -l kvm-capable=true` 返回 2 个节点
- 节点状态 `Ready`

**验证嵌套虚拟化：**
```bash
NODE=$(kubectl get nodes -l kvm-capable=true -o jsonpath='{.items[0].metadata.name}')
kubectl debug node/$NODE -it --image=ubuntu -- bash -c "ls -la /dev/kvm"
# 预期：crw-rw-rw- 1 root kvm 10, 232 ...
```

**记录：**
- [ ] eksctl 是否成功
- [ ] 节点数量和状态
- [ ] `/dev/kvm` 是否存在
- [ ] 如有报错，粘贴完整错误信息

---

## Step 4B：场景 B — 先建集群，后加 node group

```bash
# Step B-1：只建集群
eksctl create cluster -f deploy/eks/eks-cluster-only.yaml
# 预计耗时：12-15 分钟（无 node group，更快）

# 确认集群就绪（无节点）
kubectl get nodes
# 预期：No resources found

# Step B-2：追加 KVM node group
eksctl create nodegroup -f deploy/eks/eks-add-kvm-nodegroup.yaml
# 预计耗时：5-8 分钟
```

**预期：** 同场景 A，节点 Ready，`/dev/kvm` 存在

**记录：**
- [ ] 两步是否都成功
- [ ] 与场景 A 有无差异

---

## Step 5：部署 KVM device plugin

```bash
# 安装 kubevirt KVM device plugin（DaemonSet）
kubectl apply -f https://github.com/kubevirt/kubevirt/releases/latest/download/kubevirt-operator.yaml
kubectl apply -f https://github.com/kubevirt/kubevirt/releases/latest/download/kubevirt-cr.yaml

# 等待就绪（约 2-3 分钟）
kubectl wait --for=condition=ready pod -l kubevirt.io=virt-handler -n kubevirt --timeout=180s

# 验证节点上 kvm resource 可用
kubectl describe node -l kvm-capable=true | grep -A5 "devices.kubevirt.io/kvm"
# 预期：devices.kubevirt.io/kvm: 1（Capacity 和 Allocatable 里都有）
```

**记录：**
- [ ] device plugin 安装是否成功
- [ ] 节点是否显示 `devices.kubevirt.io/kvm: 1`
- [ ] 如有报错，粘贴错误

---

## Step 6：准备 PVC 数据（vmlinux + rootfs）

PVC 创建后是空的，需要把 vmlinux 和 rootfs 上传进去。用一个 init Job 完成：

```bash
# 创建 namespace 和 PVC
kubectl apply -f deploy/k8s/namespace.yaml
kubectl apply -f deploy/k8s/pvc.yaml

# 等 PVC Bound
kubectl wait --for=jsonpath='{.status.phase}'=Bound pvc/zeroboot-data -n zeroboot --timeout=60s

# 创建临时 pod 挂载 PVC，用于上传文件
kubectl run data-loader --image=ubuntu --restart=Never -n zeroboot \
  --overrides='{"spec":{"volumes":[{"name":"data","persistentVolumeClaim":{"claimName":"zeroboot-data"}}],"containers":[{"name":"loader","image":"ubuntu","command":["sleep","3600"],"volumeMounts":[{"name":"data","mountPath":"/data"}]}]}}'

kubectl wait --for=condition=ready pod/data-loader -n zeroboot --timeout=60s

# 从 dev-server 上传 vmlinux 和 rootfs
# （在 dev-server 上执行，需要 kubectl 访问集群）
kubectl cp ~/fc-exp/vmlinux.bin zeroboot/data-loader:/data/vmlinux-fc
kubectl cp ~/zeroboot-work5/rootfs.ext4 zeroboot/data-loader:/data/rootfs-python.ext4

# 确认文件已上传
kubectl exec -n zeroboot data-loader -- ls -lh /data/
# 预期：vmlinux-fc (~21MB), rootfs-python.ext4 (~500MB)

# 清理临时 pod
kubectl delete pod data-loader -n zeroboot
```

**记录：**
- [ ] PVC 是否 Bound
- [ ] 文件上传是否成功
- [ ] 文件大小是否正确

---

## Step 7：部署 zeroboot

```bash
# 更新 deployment.yaml 里的镜像地址
# 把 ghcr.io/zerobootdev/zeroboot:latest 替换成 ECR 地址
ECR_IMAGE="${AWS_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com/zeroboot:test"

sed -i "s|ghcr.io/zerobootdev/zeroboot:latest|${ECR_IMAGE}|" deploy/k8s/deployment.yaml

# 部署
kubectl apply -f deploy/k8s/deployment.yaml
kubectl apply -f deploy/k8s/service.yaml

# 监控 rollout（首次启动约 30s，等待 snapshot 创建）
kubectl rollout status deployment/zeroboot -n zeroboot --timeout=120s

# 查看 Pod 日志（确认 template 创建成功）
kubectl logs -n zeroboot -l app=zeroboot --follow
```

**预期日志：**
```
No snapshot found — creating Python template (this takes ~15s)...
Template created.
Starting zeroboot API server on port 8080...
```

**第二个 Pod 重启（snapshot 已在 PVC）：**
```
Snapshot found — skipping template creation.
Starting zeroboot API server on port 8080...
```

**记录：**
- [ ] Pod 是否进入 Running 状态
- [ ] 日志里 template 创建是否成功
- [ ] readiness probe 是否通过
- [ ] 两个 Pod 是否分布在不同节点（podAntiAffinity 验证）
  ```bash
  kubectl get pods -n zeroboot -o wide
  # NODE 列应该是两个不同的节点
  ```

---

## Step 8：端到端功能验证

```bash
# port-forward 到本地
kubectl port-forward svc/zeroboot 8080:80 -n zeroboot &

# 健康检查
curl -s localhost:8080/v1/health
# 预期：{"status":"ok"}

# 执行 Python 代码
curl -s -X POST localhost:8080/v1/exec \
  -H 'Content-Type: application/json' \
  -d '{"code": "print(1+1)"}' | jq .
# 预期：{"stdout":"2","exit_code":0,"fork_time_ms":<1,...}

# numpy（验证预加载）
curl -s -X POST localhost:8080/v1/exec \
  -H 'Content-Type: application/json' \
  -d '{"code": "import numpy as np; print(np.array([1,2,3]).mean())"}' | jq .
# 预期：{"stdout":"2.0","exit_code":0,...}

# 查看 Prometheus metrics
curl -s localhost:8080/v1/metrics | grep zeroboot_concurrent_forks
# 预期：zeroboot_concurrent_forks 0
```

**记录：**
- [ ] `/v1/health` 返回 ok
- [ ] `print(1+1)` 返回 `stdout: "2"`
- [ ] `fork_time_ms` 值（是否 <10ms，理想 <2ms）
- [ ] numpy 执行是否成功
- [ ] `/v1/metrics` 是否有 `zeroboot_concurrent_forks`

---

## Step 9：清理

```bash
# 删除 K8s 资源
kubectl delete -f deploy/k8s/

# 删除集群（场景 A 或 B，根据实际情况选）
eksctl delete cluster --name zeroboot-eks --region ap-southeast-1

# 删除 ECR repo（可选）
aws ecr delete-repository --repository-name zeroboot --region ap-southeast-1 --force
```

---

## 结果汇总模板

验证完成后，请将以下内容反馈给 openclaw-coding：

```
## 验证结果

**环境：** ap-southeast-1 / c8i.xlarge / EKS 1.31

### 场景 A（新建集群）
- eksctl create cluster: ✅/❌
- /dev/kvm 可访问: ✅/❌
- KVM device plugin: ✅/❌

### 场景 B（追加 node group）
- eksctl create cluster (only): ✅/❌
- eksctl create nodegroup: ✅/❌

### Docker 镜像
- docker build: ✅/❌ 耗时: ___  大小: ___
- ECR push: ✅/❌

### K8s 部署
- Pod 状态: ✅Running / ❌ (错误: ___)
- Template 创建日志: ✅正常 / ❌
- PodAntiAffinity（分布在不同节点）: ✅/❌
- readiness probe: ✅/❌

### 功能验证
- /v1/health: ✅/❌
- print(1+1): ✅/❌  fork_time_ms: ___
- numpy: ✅/❌
- /v1/metrics: ✅/❌

### 发现的问题
（列出所有报错、需要修改的地方、文档不清楚的地方）
```
