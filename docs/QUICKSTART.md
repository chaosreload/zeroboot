# Zeroboot 上手指南

> 基于 chaosreload/zeroboot（feat/virtio-blk-filesystem 分支）
> 环境：AWS c8i.xlarge，Ubuntu 22.04，嵌套虚拟化已开启

---

## 一、机器准备

### 1.1 开启嵌套虚拟化（已有 EC2 实例）

```bash
# 先停止实例（在 AWS Console 或 CLI）
aws ec2 stop-instances --region ap-southeast-1 --instance-ids i-XXXXXXXXX

# 开启嵌套虚拟化（需要 AWS CLI >= 2.34）
aws ec2 modify-instance-cpu-options \
  --region ap-southeast-1 \
  --instance-id i-XXXXXXXXX \
  --nested-virtualization enabled

# 重新启动
aws ec2 start-instances --region ap-southeast-1 --instance-ids i-XXXXXXXXX
```

> ⚠️ 只有 c8i / m8i / r8i 系列支持嵌套虚拟化

### 1.2 验证 KVM 可用

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

## 四、制作 Ubuntu 22.04 Rootfs

这一步需要 ~5 分钟，制作一个含 Python 3.10 + numpy + pandas 的根文件系统。

```bash
# 安装 debootstrap（如果没有）
sudo apt-get install -y debootstrap

# Step 1：创建 Ubuntu 22.04 最小系统
sudo mkdir -p /tmp/rootfs_build
sudo debootstrap --arch=amd64 jammy /tmp/rootfs_build http://archive.ubuntu.com/ubuntu/

# Step 2：添加 universe 源（pip 在这里）
echo "deb http://archive.ubuntu.com/ubuntu jammy main universe" | \
  sudo tee /tmp/rootfs_build/etc/apt/sources.list

# Step 3：安装 Python + 科学包
sudo chroot /tmp/rootfs_build apt-get update -qq
sudo chroot /tmp/rootfs_build apt-get install -y python3 python3-pip gcc
sudo chroot /tmp/rootfs_build pip3 install numpy pandas
# 验证
sudo chroot /tmp/rootfs_build python3 -c "import numpy; print('numpy', numpy.__version__)"
sudo chroot /tmp/rootfs_build python3 -c "import pandas; print('pandas', pandas.__version__)"
```

---

## 五、编译 Guest Init

```bash
# 下载 init.c（从我们的 fork）
curl -fsSL -o /tmp/rootfs_build/init.c \
  https://raw.githubusercontent.com/chaosreload/zeroboot/feat/virtio-blk-filesystem/guest/init.c

# 编译（静态链接，无依赖）
sudo chroot /tmp/rootfs_build gcc -O2 -static -o /init /init.c
sudo rm /tmp/rootfs_build/init.c

# 验证
sudo file /tmp/rootfs_build/init
# 期望：ELF 64-bit LSB executable ... statically linked
```

---

## 六、打包 Rootfs

```bash
cd ~/fc-exp

# 创建 1.5GB ext4 镜像
dd if=/dev/zero of=rootfs.ext4 bs=1M count=1500 status=progress
mkfs.ext4 -F rootfs.ext4

# 写入内容
sudo mkdir -p /mnt/rootfs_out
sudo mount -o loop rootfs.ext4 /mnt/rootfs_out
sudo cp -a /tmp/rootfs_build/. /mnt/rootfs_out/
sudo umount /mnt/rootfs_out

ls -lh rootfs.ext4
# 期望：~1.5GB 文件
```

---

## 七、编译 Zeroboot

```bash
# 安装 Rust（如果没有）
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

## 八、创建 Template（拍 Snapshot）

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

## 九、测试执行

### 9.1 基本 echo

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

### 9.2 读取文件（验证文件系统）

```bash
~/zeroboot/target/release/zeroboot test-exec ~/zeroboot-work "cat /etc/os-release"
# 期望：输出 Ubuntu 22.04 版本信息
```

### 9.3 执行 Python 代码

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

## 十、性能 Benchmark

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

## 十一、启动 API Server

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

## 十二、理解核心流程

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
A: Snapshot 等待时间太短，Python 还没启动就拍快照了。增大 `wait_secs`（第八步最后一个参数）到 15 秒。

**Q: `Warning: snapshot CPUID rejected`**  
A: 嵌套虚拟化环境限制，不影响功能，用 `2>/dev/null` 过滤即可。

**Q: `Too many open files (os error 24)`（1000 并发 bench 时）**  
A: `ulimit -n 65535` 增大文件句柄限制。
