# Zeroboot 上手指南

> 基于 chaosreload/zeroboot（feat/virtio-blk-filesystem 分支）
> 环境：AWS c8i.xlarge，Ubuntu 22.04，嵌套虚拟化已开启

---

## 一、机器准备

### 1.1 启动新实例（推荐方式）

直接在 `run-instances` 时通过 `--cpu-options` 一步开启嵌套虚拟化，无需 stop/start 两次操作。

> ⚠️ 需要 AWS CLI >= v2.34，旧版本不支持 `NestedVirtualization` 参数

```bash
# 获取最新 Ubuntu 22.04 AMI（ap-southeast-1）
AMI_ID=$(aws ec2 describe-images \
  --owners 099720109477 \
  --filters 'Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*' \
            'Name=state,Values=available' \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' \
  --region ap-southeast-1 --output text)

# 启动实例，直接开嵌套虚拟化
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type c8i.xlarge \
  --key-name <your-key-name> \
  --security-group-ids <your-sg-id> \
  --cpu-options "NestedVirtualization=enabled" \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=zeroboot-fresh}]' \
  --region ap-southeast-1 \
  --query 'Instances[0].InstanceId' --output text)

echo "Instance ID: $INSTANCE_ID"

# 等待启动完成
aws ec2 wait instance-running --instance-ids $INSTANCE_ID --region ap-southeast-1
```

> ⚠️ 只有 c8i / m8i / r8i 系列支持嵌套虚拟化（均为 Intel 第 8 代平台）

### 1.2 已有实例启用嵌套虚拟化

如果你已有运行中的 C8i 实例，需要先停机再改配置：

```bash
aws ec2 stop-instances --region ap-southeast-1 --instance-ids $INSTANCE_ID
aws ec2 wait instance-stopped --region ap-southeast-1 --instance-ids $INSTANCE_ID

aws ec2 modify-instance-cpu-options \
  --region ap-southeast-1 \
  --instance-id $INSTANCE_ID \
  --nested-virtualization enabled

aws ec2 start-instances --region ap-southeast-1 --instance-ids $INSTANCE_ID
```

### 1.3 验证 KVM 可用

```bash
ssh ubuntu@<your-instance-ip>
ls -la /dev/kvm
# 期望：crw-rw-rw- 1 root kvm 10, 232 ...

# 如果权限不够：
sudo chmod 666 /dev/kvm
```

---

## 二、安装 Firecracker

```bash
# 下载 v1.15.0（x86_64）
curl -L -o fc.tgz https://github.com/firecracker-microvm/firecracker/releases/download/v1.15.0/firecracker-v1.15.0-x86_64.tgz
tar xzf fc.tgz
sudo mv release-v1.15.0-x86_64/firecracker-v1.15.0-x86_64 /usr/local/bin/firecracker
sudo mv release-v1.15.0-x86_64/jailer-v1.15.0-x86_64 /usr/local/bin/jailer
sudo chmod +x /usr/local/bin/firecracker /usr/local/bin/jailer
firecracker --version
# 期望：Firecracker v1.15.0
```

---

## 三、下载 Kernel

```bash
mkdir -p ~/fc-exp
cd ~/fc-exp

# Firecracker 官方 quickstart kernel（4.14.174，轻量启动快）
curl -fsSL -o vmlinux.bin \
  https://s3.amazonaws.com/spec.ccfc.min/img/quickstart_guide/x86_64/kernels/vmlinux.bin

ls -lh vmlinux.bin
# 期望：~21MB
```

---

## 四、用 Docker 构建 Rootfs

使用 Docker 构建 rootfs，比 debootstrap 更可复现、更易维护（类似 E2B sandbox template 的方式）。

> 需要先安装 Docker：

```bash
sudo apt remove $(dpkg --get-selections docker.io docker-compose docker-compose-v2 docker-doc podman-docker containerd runc | cut -f1)
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update

sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
```

### 4.1 创建 Dockerfile

```bash
mkdir -p ~/zeroboot-rootfs
cd ~/zeroboot-rootfs

# 下载 init.c（静态编译的 guest init）
curl -fsSL -o init.c \
  https://raw.githubusercontent.com/chaosreload/zeroboot/feat/virtio-blk-filesystem/guest/init.c

cat > Dockerfile << 'EOF'
FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update -qq && \
    apt-get install -y --no-install-recommends \
      python3 python3-pip gcc libc6-dev && \
    pip3 install --no-cache-dir numpy pandas && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

# 编译静态 init（guest 的 PID 1）
COPY init.c /init.c
RUN gcc -O2 -static -o /init /init.c && rm /init.c

EOF
```

### 4.2 构建并导出为 ext4

```bash
cd ~/zeroboot-rootfs

# 构建镜像（~3 分钟，主要是 pip install）
sudo docker build -t zeroboot-rootfs .

# 验证 init 编译结果
sudo docker run --rm zeroboot-rootfs ldd /init
# 期望：/init: ELF 64-bit LSB executable ... statically linked, stripped

# 导出为 tar
sudo docker create --name tmp-rootfs zeroboot-rootfs
sudo docker export tmp-rootfs -o rootfs.tar
sudo docker rm tmp-rootfs

# 打包成 ext4 镜像
cd ~/fc-exp
dd if=/dev/zero of=rootfs.ext4 bs=1M count=1500 status=progress
mkfs.ext4 -F rootfs.ext4

sudo mkdir -p /mnt/rootfs_out
sudo mount -o loop rootfs.ext4 /mnt/rootfs_out
sudo tar xf ~/zeroboot-rootfs/rootfs.tar -C /mnt/rootfs_out
sudo umount /mnt/rootfs_out

ls -lh rootfs.ext4
# 期望：~1.5GB 文件
```

### 4.3 验证 rootfs 内容

```bash
# 挂载检查
sudo mount -o loop,ro rootfs.ext4 /mnt/rootfs_out
sudo chroot /mnt/rootfs_out python3 -c "import numpy, pandas; print('numpy', numpy.__version__, 'pandas', pandas.__version__)"
sudo chroot /mnt/rootfs_out ldd /init
sudo umount /mnt/rootfs_out
```

---

## 五、编译 Zeroboot

```bash
# 安装依赖（C 编译器 + Rust）
sudo apt-get install -y build-essential
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source ~/.cargo/env

# 克隆代码
git clone -b feat/virtio-blk-filesystem \
  https://github.com/chaosreload/zeroboot.git ~/zeroboot
cd ~/zeroboot

# 编译（release 模式，~30 秒）
cargo build --release

ls -lh target/release/zeroboot
# 期望：~1.8MB ELF binary
```

---

## 六、创建 Template（拍 Snapshot）

```bash
# 释放内存缓存（避免 OOM）
echo 3 | sudo tee /proc/sys/vm/drop_caches > /dev/null

# 创建工作目录
mkdir -p ~/zeroboot-work

# 拍快照（约 10 秒）
# 参数：<kernel> <rootfs> <workdir> <wait_secs> <init_path> <mem_mib>
~/zeroboot/target/release/zeroboot template \
  ~/fc-exp/vmlinux.bin \
  ~/fc-exp/rootfs.ext4 \
  ~/zeroboot-work \
  10 /init 512

# 期望输出：
# Starting Firecracker...
# Firecracker VM started
# Waiting 10s for guest to boot...
# Pausing VM...
# Creating snapshot...
# Snapshot created: state=14312B, mem=512MB
# Template created in 13.xx s
```

查看产出：

```bash
ls -lh ~/zeroboot-work/snapshot/
# vmstate  (~14KB, CPU 寄存器状态)
# mem      (~512MB, 内存镜像)

cat ~/zeroboot-work/rootfs_path
# 应该显示 rootfs.ext4 的绝对路径
```

---

## 七、测试执行

### 7.1 基本 echo

```bash
echo 3 | sudo tee /proc/sys/vm/drop_caches > /dev/null

~/zeroboot/target/release/zeroboot test-exec ~/zeroboot-work "echo hello"
# 期望：
# Fork time: ~1ms
# === Output ===
# echo hello
# hello
# ZEROBOOT_DONE
```

### 7.2 读取文件（验证文件系统）

```bash
~/zeroboot/target/release/zeroboot test-exec ~/zeroboot-work "cat /etc/os-release"
# 期望：输出 Ubuntu 22.04 版本信息
```

### 7.3 执行 Python 代码

```bash
# 简单计算
~/zeroboot/target/release/zeroboot test-exec ~/zeroboot-work "CODE:print(1+1)"
# 期望：2

# 使用 numpy
~/zeroboot/target/release/zeroboot test-exec ~/zeroboot-work \
  "CODE:import numpy as np; print(np.array([1,2,3]).mean())"
# 期望：2.0

# 写文件（验证 CoW 隔离）
~/zeroboot/target/release/zeroboot test-exec ~/zeroboot-work \
  "CODE:open('/tmp/test','w').write('hello'); print(open('/tmp/test').read())"
# 期望：hello
```

---

## 八、性能 Benchmark

```bash
echo 3 | sudo tee /proc/sys/vm/drop_caches > /dev/null
~/zeroboot/target/release/zeroboot bench ~/zeroboot-work 2>/dev/null
# 期望数据（c8i.xlarge）：
# Fork P50: ~655µs  ← 亚毫秒！
# Fork P99: ~996µs
# Fork + echo P50: ~5.8ms
# 内存/fork（100并发）: ~169KB
```

---

## 九、启动 API Server

```bash
~/zeroboot/target/release/zeroboot serve ~/zeroboot-work 8080
# Zeroboot API server listening on port 8080
```

另开一个终端测试：

```bash
# 健康检查
curl localhost:8080/v1/health

# 执行 Python
curl -X POST localhost:8080/v1/exec \
  -H 'Content-Type: application/json' \
  -d '{"code": "print(1+1)"}'

# 期望响应：
# {"id":"...","stdout":"2\n","stderr":"","exit_code":0,"fork_time_ms":0.65,...}
```

---

## 十、理解核心流程

```
你运行 test-exec/serve
    │
    ├─ load_snapshot()
    │    ├─ sendfile(mem_file → memfd)  [512MB, kernel-to-kernel, no user buffer]
    │    └─ parse_vmstate()             [解析 CPU 寄存器、virtio queue 地址]
    │
    └─ fork_cow()  [~1ms]
         ├─ KVM: create_vm + create_irq_chip
         ├─ mmap(memfd, MAP_PRIVATE)     [CoW: 读共享，写触发 page fault]
         ├─ 恢复 CPU 状态: sregs→XCRS→XSAVE→regs→LAPIC→MSRs
         ├─ 创建 OverlayBlockDevice      [per-fork 内存 CoW 层]
         └─ 运行 VM loop
              ├─ IoOut/IoIn  → 16550 UART (serial 通信)
              └─ MmioWrite   → VirtioBlk (文件系统 I/O)
```

---

## 常见问题

**Q: `ls: cannot access '/dev/kvm'`**  
A: 机器没开嵌套虚拟化，参考第一步。

**Q: `OOM Killed`**  
A: 先运行 `echo 3 | sudo tee /proc/sys/vm/drop_caches > /dev/null` 清理 page cache。

**Q: `echo hello` 正常但 `CODE:` 卡住**  
A: Snapshot 等待时间太短，Python 还没启动就拍快照了。增大 `wait_secs`（第六步最后一个参数）到 15 秒。

**Q: `Warning: snapshot CPUID rejected`**  
A: 嵌套虚拟化环境限制，不影响功能，用 `2>/dev/null` 过滤即可。

**Q: `Too many open files (os error 24)`（1000 并发 bench 时）**  
A: `ulimit -n 65535` 增大文件句柄限制。

**Q: AWS CLI 报 `Unknown parameter in CpuOptions: NestedVirtualization`**  
A: 升级 AWS CLI 到 v2.34+，旧版本不支持该参数。
