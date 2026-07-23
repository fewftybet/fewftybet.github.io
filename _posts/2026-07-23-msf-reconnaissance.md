---
title: MSF信息收集与扫描模块利用
date: 2026-07-23 11:00:00 +0800
categories: [工具速查, MSF利用]
tags: [Metasploit, 信息收集, 端口扫描, 漏洞扫描]
description: MSF 辅助模块实战，涵盖端口扫描、服务识别、SMB/SSH/HTTP/FTP 信息收集与漏洞扫描技巧。
---

# MSF信息收集与扫描模块利用

## 一、信息收集概述

信息收集是渗透测试的第一步。MSF 提供了丰富的 auxiliary 模块用于各协议的信息收集，配合数据库功能可以系统化地管理扫描结果。

### 核心思路

1. **存活探测**：确定目标主机是否在线
2. **端口扫描**：发现开放端口
3. **服务识别**：确定端口上运行的服务及版本
4. **漏洞扫描**：匹配已知漏洞

## 二、主机发现

### ARP 扫描（内网）

```bash
msf6 > use auxiliary/scanner/discovery/arp_sweep
msf6 auxiliary(scanner/discovery/arp_sweep) > set RHOSTS 192.168.1.0/24
msf6 auxiliary(scanner/discovery/arp_sweep) > set THREADS 10
msf6 auxiliary(scanner/discovery/arp_sweep) > run
```

### UDP 探测

```bash
msf6 > use auxiliary/scanner/discovery/udp_probe
msf6 auxiliary(scanner/discovery/udp_probe) > set RHOSTS 192.168.1.100
msf6 auxiliary(scanner/discovery/udp_probe) > run
```

## 三、端口扫描

### TCP Portscan

```bash
msf6 > use auxiliary/scanner/portscan/tcp
msf6 auxiliary(scanner/portscan/tcp) > set RHOSTS 192.168.1.100
msf6 auxiliary(scanner/portscan/tcp) > set PORTS 1-65535
msf6 auxiliary(scanner/portscan/tcp) > set THREADS 100
msf6 auxiliary(scanner/portscan/tcp) > run
```

### SYN 扫描（需要 root 权限）

```bash
msf6 > use auxiliary/scanner/portscan/syn
msf6 auxiliary(scanner/portscan/syn) > set RHOSTS 192.168.1.100
msf6 auxiliary(scanner/portscan/syn) > set PORTS 1-10000
msf6 auxiliary(scanner/portscan/syn) > run
```

### 使用 Nmap 集成

```bash
msf6 > db_nmap -sS -sV -O 192.168.1.100
msf6 > hosts        # 查看扫描结果
msf6 > services     # 查看服务信息
msf6 > vulns        # 查看关联漏洞
```

## 四、服务扫描

### SMB

```bash
# SMB 版本识别
msf6 > use auxiliary/scanner/smb/smb_version
msf6 auxiliary(scanner/smb/smb_version) > set RHOSTS 192.168.1.100
msf6 auxiliary(scanner/smb/smb_version) > run

# SMB 共享枚举
msf6 > use auxiliary/scanner/smb/smb_enumshares
msf6 auxiliary(scanner/smb/smb_enumshares) > set RHOSTS 192.168.1.100
msf6 auxiliary(scanner/smb/smb_enumshares) > run

# SMB 用户枚举
msf6 > use auxiliary/scanner/smb/smb_enumusers
```

### SSH

```bash
# SSH 版本识别
msf6 > use auxiliary/scanner/ssh/ssh_version
msf6 auxiliary(scanner/ssh/ssh_version) > set RHOSTS 192.168.1.100
msf6 auxiliary(scanner/ssh/ssh_version) > run

# SSH 登录爆破
msf6 > use auxiliary/scanner/ssh/ssh_login
msf6 auxiliary(scanner/ssh/ssh_login) > set RHOSTS 192.168.1.100
msf6 auxiliary(scanner/ssh/ssh_login) > set USER_FILE /usr/share/wordlists/users.txt
msf6 auxiliary(scanner/ssh/ssh_login) > set PASS_FILE /usr/share/wordlists/pass.txt
msf6 auxiliary(scanner/ssh/ssh_login) > set STOP_ON_SUCCESS true
msf6 auxiliary(scanner/ssh/ssh_login) > run
```

### HTTP

```bash
# Web 服务识别
msf6 > use auxiliary/scanner/http/http_version
msf6 auxiliary(scanner/http/http_version) > set RHOSTS 192.168.1.100
msf6 auxiliary(scanner/http/http_version) > run

# 目录扫描
msf6 > use auxiliary/scanner/http/dir_scanner
msf6 auxiliary(scanner/http/dir_scanner) > set RHOSTS 192.168.1.100
msf6 auxiliary(scanner/http/dir_scanner) > set DICTIONARY /usr/share/wordlists/dir.txt
msf6 auxiliary(scanner/http/dir_scanner) > run

# robots.txt 获取
msf6 > use auxiliary/scanner/http/robots_txt

# WordPress 扫描
msf6 > use auxiliary/scanner/http/wordpress_scanner
```

### FTP

```bash
# FTP 版本识别
msf6 > use auxiliary/scanner/ftp/ftp_version

# 匿名登录检测
msf6 > use auxiliary/scanner/ftp/anonymous
msf6 auxiliary(scanner/ftp/anonymous) > set RHOSTS 192.168.1.100
msf6 auxiliary(scanner/ftp/anonymous) > run

# FTP 登录爆破
msf6 > use auxiliary/scanner/ftp/ftp_login
```

## 五、漏洞扫描

### MS17-010 永恒之蓝

```bash
msf6 > use auxiliary/scanner/smb/smb_ms17_010
msf6 auxiliary(scanner/smb/smb_ms17_010) > set RHOSTS 192.168.1.100
msf6 auxiliary(scanner/smb/smb_ms17_010) > run
```

### SSL/TLS 漏洞

```bash
msf6 > use auxiliary/scanner/ssl/openssl_ccs
msf6 > use auxiliary/scanner/ssl/openssl_heartbleed
```

## 六、结果管理

```bash
# 查看扫描到的主机
msf6 > hosts -c address,name,os_name,vulns

# 查看开放端口
msf6 > services -S http

# 查看发现的凭证
msf6 > creds

# 导出结果
msf6 > db_export -f xml /tmp/scan_results.xml
```

---

> 下一篇文章将介绍如何利用收集到的信息匹配 Exploit 模块进行漏洞利用。
