---
title: Windows系统安全基线与加固指南
date: 2026-07-16 12:10:00 +0800
categories: [经验分享]
tags: [安全加固, Windows, 基线检查, 等保]
description: 本文档从安全服务（安服）专业视角出发，依据等级保护相关标准与行业最佳实践，围绕身份鉴别、访问控制、安全审计、资源控制、剩余信息保护、入侵防范、恶意代码防范七大维度，系统性梳理Windows操作系统的安全基线检查与加固方案。
---

# Windows系统安全基线与加固指南

# 一、概述

本文档从安全服务（安服）专业视角出发，依据等级保护相关标准与行业最佳实践，围绕身份鉴别、访问控制、安全审计、资源控制、剩余信息保护、入侵防范、恶意代码防范七大维度，系统性梳理Windows操作系统的安全基线检查与加固方案。适用于Windows Server 2016/2019/2022及Windows 10/11企业版终端的安全配置合规性检查与加固实施。

**安服实施原则：**所有加固操作前必须完成基线快照备份，优先通过组策略（GPO）批量下发，单台服务器采用本地安全策略与注册表结合方式；变更前需客户确认并留存回滚方案。

自动化检测脚本：[tangjie1/-Baseline-check: 唐门·心法 windows和linux基线检查，配套自动化检查脚本。](https://github.com/tangjie1/-Baseline-check)

# 二、身份鉴别

## 2\.1 账户管理规范

**默认账户清理。**重命名默认管理员账户Administrator，禁用Guest来宾账户，删除或禁用所有测试账户、共享账户、过期账户。安服检查时需逐一核对账户列表，确认不存在匿名账户与弱命名管理员账户。

```powershell
# 重命名默认管理员账户
Rename-LocalUser -Name "Administrator" -NewName "Admin_Sec01"

# 禁用Guest账户
net user Guest /active:no

# 查看所有本地账户
net user

```

## 2\.2 密码策略强化

**强口令策略配置。**强制开启密码复杂度要求，密码最小长度不少于12位，包含大小写字母、数字与特殊字符三类组合；密码最长使用期限不超过90天，禁止复用最近5次历史密码。安服核查时需确认密码策略已应用，不存在空密码或弱口令账户。

```powershell
# 密码复杂度启用
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Netlogon\Parameters" -Name "RequireStrongKey" -Value 1

# 本地安全策略路径：账户策略 → 密码策略
# 密码必须符合复杂性要求：已启用
# 密码长度最小值：12个字符
# 密码最长使用期限：90天
# 强制密码历史：5个记住的密码

```

## 2\.3 账户锁定策略

**防暴力破解配置。**设置登录失败锁定阈值，连续5次无效登录后锁定账户30分钟，复位账户锁定计数器为30分钟。安服视角需重点核查远程桌面（RDP）、SSH等远程接入端口的 brute\-force 防护是否到位。

```powershell
# 账户锁定阈值：5次无效登录
# 账户锁定时间：30分钟
# 重置账户锁定计数器：30分钟后
# 配置路径：本地安全策略 → 账户策略 → 账户锁定策略

```

## 2\.4 身份认证增强

**NTLM协议降级防护。**强制使用NTLMv2认证，禁用LM与NTLMv1哈希存储，防止哈希传递攻击（Pass\-the\-Hash）；核心服务器建议部署双因素认证（2FA），对管理员账户增加动态口令验证环节。

```powershell
# 强制NTLMv2，拒绝LM和NTLMv1
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Lsa" /v LmCompatibilityLevel /t REG_DWORD /d 5 /f

# 禁用可逆加密存储密码
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Lsa" /v "NoDefaultAdminOwner" /t REG_DWORD /d 1 /f

```

# 三、访问控制

## 3\.1 用户权限分配

**最小权限原则落地。**从远端系统强制关机仅指派给Administrators组；关闭系统权限同样仅限管理员组；拒绝本地登录包含Guest、匿名账户；通过终端服务登录需严格限制授权用户组。安服检查时需逐项核对用户权限分配清单，确认越权分配项。

```cmd
配置路径：本地安全策略 → 本地策略 → 用户权限分配

1. 从远程系统强制关机：只指派给 Administrators 组
2. 关闭系统：只指派给 Administrators 组
3. 拒绝本地登录：Guest、匿名账户
4. 允许通过终端服务登录：仅限授权管理员组
5. 取得文件或其他对象的所有权：Administrators

```

## 3\.2 共享权限管控

**默认共享清理。**禁用系统默认共享（ADMIN$、C$、IPC$等），业务共享文件夹需配置独立权限，禁止Everyone组拥有写入权限。安服实施时需逐一核查共享列表，确认匿名用户无法枚举共享资源。

```powershell
# 查看所有共享
Get-SmbShare

# 禁用默认共享（重启后生效）
reg add "HKLM\SYSTEM\CurrentControlSet\Services\LanmanServer\Parameters" /v AutoShareServer /t REG_DWORD /d 0 /f
reg add "HKLM\SYSTEM\CurrentControlSet\Services\LanmanServer\Parameters" /v AutoShareWks /t REG_DWORD /d 0 /f

# 限制匿名访问共享
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Lsa" /v RestrictAnonymous /t REG_DWORD /d 1 /f

```

## 3\.3 注册表与文件权限

**敏感目录权限收敛。**系统盘根目录、Windows目录、System32目录权限需严格控制，普通用户仅拥有读取与执行权限；注册表关键启动项（Run、RunOnce）需限制写入权限，防止恶意程序持久化植入。

```cmd
重点核查路径：
C:\Windows\System32\
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce
C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp\

```

## 3\.4 UAC用户账户控制

**UAC最高级别启用。**确保用户账户控制（UAC）设置为始终通知级别，管理员审批模式开启，提升权限时需进行安全桌面提示。安服视角下UAC是拦截未授权提权的重要防线，不得随意关闭。

```powershell
# 启用UAC最高级别
Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" -Name "ConsentPromptBehaviorAdmin" -Value 2
Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" -Name "EnableLUA" -Value 1

```

# 四、安全审计

## 4\.1 全量审计策略启用

**高级审核策略全覆盖。**启用登录/注销、账户管理、策略更改、对象访问、特权使用、目录服务访问、系统事件等全类别审核，同时记录成功与失败事件。安服基线核查需确认审计日志覆盖关键操作行为，满足溯源取证需求。

```cmd
# 命令行启用所有类别审核（成功+失败）
auditpol /set /category:* /success:enable /failure:enable

# 查看当前审计策略
auditpol /get /category:*

```

## 4\.2 日志容量与留存

**日志存储扩容。**安全日志、系统日志、应用程序日志单文件容量建议不低于200MB，日志满时按需要覆盖旧事件；核心服务器建议配置日志转发（SIEM/日志服务器），本地留存周期不少于180天。安服检查时需确认日志不会因容量不足被覆盖丢失。

```powershell
# 设置安全日志最大大小200MB
wevtutil sl Security /ms:209715200

# 设置系统日志最大大小200MB
wevtutil sl System /ms:209715200

# 设置应用程序日志最大大小200MB
wevtutil sl Application /ms:209715200

```

## 4\.3 关键事件审计清单

|事件ID|事件类型|安服关注要点|
|---|---|---|
|4624|登录成功|异常时间、异常来源IP登录|
|4625|登录失败|暴力破解攻击识别|
|4672|管理员登录|特权账户使用监控|
|4720|创建用户账户|未授权账户创建告警|
|4723/4724|密码更改/重置|异常密码操作溯源|
|4688|进程创建|恶意程序执行追踪|
|4698|计划任务创建|持久化攻击检测|

## 4\.4 时间同步配置

**NTP时间校准。**配置系统与内部NTP服务器同步，确保日志时间戳准确统一，为跨设备日志关联分析与事件溯源提供时间基准。安服核查时需确认时间偏差不超过5分钟。

```cmd
# 设置NTP服务器
w32tm /config /manualpeerlist:"ntp.aliyun.com" /syncfromflags:manual /reliable:yes /update

# 立即同步
w32tm /resync

```

# 五、资源控制

## 5\.1 服务与端口精简

**攻击面缩减。**禁用所有非业务必需的系统服务与端口，包括Telnet、FTP、RemoteRegistry、SSDP Discovery、UPnP设备主机等高危服务；关闭SMBv1协议，防范永恒之蓝类勒索病毒利用。安服基线检查需逐一核对开放端口与运行服务清单。

```cmd
# 禁用高危服务
sc config Telnet start= disabled
sc config RemoteRegistry start= disabled
sc config FTP start= disabled
sc config SSDPSRV start= disabled
sc config upnphost start= disabled

# 禁用SMBv1
Disable-WindowsOptionalFeature -Online -FeatureName SMB1Protocol

```

## 5\.2 防火墙策略加固

**白名单准入原则。**Windows Defender防火墙默认拒绝所有入站连接，仅放行业务必需端口（如443、3389管理端口建议绑定指定管理IP）；出站规则按需收紧，防止服务器被用作横向跳板。安服实施时入站规则需逐条审核，禁用所有非必要放行规则。

```powershell
# 启用防火墙（域、专用、公用配置文件）
Set-NetFirewallProfile -Profile Domain,Private,Public -Enabled True

# 默认入站阻止、出站允许
Set-NetFirewallProfile -DefaultInboundAction Block -DefaultOutboundAction Allow

# RDP仅允许指定管理IP段（示例）
New-NetFirewallRule -DisplayName "Allow RDP from Mgmt" -Direction Inbound -Protocol TCP -LocalPort 3389 -RemoteAddress 192.168.1.0/24 -Action Allow

```

## 5\.3 远程桌面安全加固

**RDP安全收敛。**修改默认3389端口为非知名端口，启用网络级别身份验证（NLA），设置最大空闲断开时间30分钟；限制单用户最大并发会话数，禁止空密码远程登录。安服视角需重点核查RDP是否暴露公网及是否存在弱口令。

```powershell
# 启用NLA网络级别身份验证
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" -Name "UserAuthentication" -Value 1

# 设置空闲断开时间30分钟
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services" -Name "MaxIdleTime" -Value 1800000

# 禁止空密码远程登录
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Lsa" /v LimitBlankPasswordUse /t REG_DWORD /d 1 /f

```

## 5\.4 USB与外设管控

**移动介质准入控制。**通过组策略禁用USB存储设备读写权限，或配置只读模式；关闭自动播放功能，防范U盘病毒传播。安服检查时需确认Autorun与自动播放已全部禁用。

```cmd
# 关闭所有驱动器自动播放
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\Explorer" /v NoDriveTypeAutoRun /t REG_DWORD /d 255 /f

# 禁用USB存储设备
reg add "HKLM\SYSTEM\CurrentControlSet\Services\USBSTOR" /v Start /t REG_DWORD /d 4 /f

```

# 六、剩余信息保护

## 6\.1 内存与页面文件清理

**关机清除虚拟内存。**配置系统关机时自动清除页面文件（pagefile\.sys），防止物理内存中的敏感数据残留在虚拟内存文件中被离线提取。安服核查项：关机时清除虚拟内存页面文件策略是否启用。

```cmd
# 启用关机清除页面文件
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Memory Management" /v ClearPageFileAtShutdown /t REG_DWORD /d 1 /f

```

## 6\.2 临时文件与回收站清理

**临时目录定期清理。**配置系统临时文件夹、用户临时目录（%TEMP%）定期自动清理；启用回收站直接删除模式或设置容量上限，避免删除文件可被轻易恢复。核心业务服务器建议部署定期清理脚本。

```powershell
# 清理系统临时文件
Remove-Item -Path "$env:TEMP\*" -Recurse -Force -ErrorAction SilentlyContinue

# 清理Windows更新缓存
Remove-Item -Path "C:\Windows\SoftwareDistribution\Download\*" -Recurse -Force -ErrorAction SilentlyContinue

```

## 6\.3 远程会话残留清理

**断开会话自动注销。**配置终端服务断开会话后自动注销用户，释放内存资源并清除会话上下文；设置空闲会话超时断开，防止遗忘的远程会话遗留敏感操作界面。

```cmd
配置路径：本地组策略 → 计算机配置 → 管理模板 → Windows组件 → 远程桌面服务 → 远程桌面会话主机 → 会话时间限制

1. 设置活动但空闲的远程桌面服务会话的时间限制：30分钟
2. 设置已断开连接的会话的时间限制：15分钟后结束会话

```

## 6\.4 凭据残留防护

**内存凭据保护。**启用WDigest身份验证禁用，防止明文密码存储在LSASS进程中；启用Credential Guard（需硬件支持）隔离凭据存储，抵御Mimikatz等凭据转储工具。

```cmd
# 禁用WDigest，防止LSASS明文凭据
reg add "HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest" /v UseLogonCredential /t REG_DWORD /d 0 /f

# 启用LSASS保护进程模式（PPL）
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Lsa" /v RunAsPPL /t REG_DWORD /d 1 /f

```

# 七、入侵防范

## 7\.1 系统补丁管理

**漏洞修复闭环。**启用Windows自动更新服务，配置自动下载并安装安全更新；高危漏洞补丁（如永恒之蓝MS17\-010、PrintNightmare等）需在72小时内完成修复。安服视角需建立补丁分级机制，区分紧急、重要、一般等级别，确保关键漏洞优先修复。

```powershell
# 启用自动更新
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" -Name "NoAutoUpdate" -Value 0
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" -Name "AUOptions" -Value 4

# 查看已安装补丁
wmic qfe list brief

```

## 7\.2 漏洞与攻击面缩减

**高危组件禁用。**卸载或禁用不必要的系统组件，如旧版\.NET Framework、SMBv1、Telnet客户端、TFTP等；启用攻击面减少规则（ASR），拦截Office宏衍生子进程、脚本解释器执行可执行文件、LSASS凭据转储等典型攻击行为。

```powershell
# 启用关键攻击面减少规则（Defender for Endpoint）
# 阻止Office应用创建子进程
Set-MpPreference -AttackSurfaceReductionRules_Ids D4F940AB-401B-4EFC-AADC-AD5F3C50688A -AttackSurfaceReductionRules_Actions Enabled

# 阻止从电子邮件客户端运行的可执行内容
Set-MpPreference -AttackSurfaceReductionRules_Ids 3B576869-A4EC-4529-8536-B80A7769E899 -AttackSurfaceReductionRules_Actions Enabled

# 阻止凭据窃取行为
Set-MpPreference -AttackSurfaceReductionRules_Ids 9e6c4e1f-7d60-472f-ba1a-a39ef669e4b2 -AttackSurfaceReductionRules_Actions Enabled

```

## 7\.3 持久化入口排查

**启动项与计划任务审计。**定期排查注册表Run/RunOnce启动项、计划任务、服务启动项、WMI事件订阅等常见持久化入口；安服应急响应场景下需重点核查异常计划任务与新增服务，识别后门植入痕迹。

```powershell
# 查看所有计划任务
Get-ScheduledTask | Where-Object {$_.State -eq "Ready"} | Select-Object TaskName,TaskPath

# 查看注册表启动项
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run"
Get-ItemProperty "HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run"

# 查看所有服务
Get-Service | Where-Object {$_.StartType -eq "Automatic"} | Select-Object Name,DisplayName

```

## 7\.4 网络入侵检测

**异常连接监控。**启用防火墙高级安全日志，记录所有丢弃的数据包与成功连接；核心服务器建议部署HIDS（主机入侵检测系统），监控异常端口监听、外连C2服务器、横向移动特征等行为。安服基线核查需确认不存在异常监听端口与可疑外连会话。

```cmd
# 查看所有监听端口与对应进程
netstat -ano | findstr "LISTENING"

# 查看已建立的外部连接
netstat -ano | findstr "ESTABLISHED"

# 查看路由表与ARP缓存
route print & arp -a

```

# 八、恶意代码防范

## 8\.1 杀毒软件部署与更新

**终端防护全覆盖。**确保Windows Defender防病毒启用并运行在实时保护模式，病毒定义库保持7天内更新；企业环境建议统一部署EDR/XDR终端检测与响应平台。安服检查时需确认杀毒软件未被禁用、排除项未被滥用、特征库版本合规。

```powershell
# 查看Defender状态
Get-MpComputerStatus

# 查看病毒定义版本
(Get-MpComputerStatus).AntivirusSignatureVersion

# 启用实时保护
Set-MpPreference -DisableRealtimeMonitoring $false

# 启用云提供保护
Set-MpPreference -DisableBlockAtFirstSeen $false

```

## 8\.2 扫描策略配置

**定期全盘扫描。**配置每周至少一次全盘快速扫描，每日快速扫描；启用行为监控、启发式检测与云保护级别；对下载文件、附件、可移动介质执行自动扫描。安服视角需确认扫描日志无大量未处理威胁。

```powershell
# 设置每日快速扫描时间（凌晨2点）
Set-MpPreference -ScanScheduleDay 0 -ScanScheduleTime 120

# 设置每周全盘扫描（周日凌晨3点）
Set-MpPreference -ScanParameters 2 -RemediationScheduleDay 0 -RemediationScheduleTime 180

# 启用PUA潜在不受欢迎应用检测
Set-MpPreference -PUAProtection 1

```

## 8\.3 勒索病毒专项防护

**多层防御体系。**启用受控文件夹访问（Controlled Folder Access），保护文档、桌面等关键目录不被未授权程序修改；启用网络勒索防护，阻断SMB横向传播；配置文件服务器卷影副本定期快照，作为数据恢复最后防线。

```powershell
# 启用受控文件夹访问
Set-MpPreference -EnableControlledFolderAccess Enabled

# 添加受保护目录
Add-MpPreference -ControlledFolderAccessProtectedFolders "C:\Users\Public\Documents"

# 启用网络勒索防护
Set-MpPreference -EnableNetworkProtection Enabled

```

## 8\.4 宏与脚本防护

**Office宏管控。**配置Office应用默认禁用所有宏，仅允许受信任位置与数字签名的宏运行；启用PowerShell约束语言模式，限制恶意脚本执行；禁用WSH脚本宿主或限制VBS/JS脚本运行权限。

```cmd
# Office宏禁用（全局）
reg add "HKLM\SOFTWARE\Policies\Microsoft\Office\16.0\Common\Security" /v DisableAllFromOfficeTemplatesValue /t REG_DWORD /d 4 /f

# 启用PowerShell脚本执行策略（仅允许本地签名脚本）
Set-ExecutionPolicy RemoteSigned -Scope LocalMachine

```

# 九、附录：安服基线核查速查表

|加固维度|核心检查项|合规标准|
|---|---|---|
|身份鉴别|默认管理员重命名|已重命名，非Administrator|
||Guest账户状态|已禁用|
||密码复杂度与长度|启用，≥12位|
||账户锁定策略|5次锁定30分钟|
|访问控制|远程关机权限|仅限Administrators|
||默认共享状态|已禁用ADMIN$/C$/IPC$|
||UAC级别|始终通知级别|
|安全审计|审计策略覆盖|全类别成功\+失败|
||安全日志容量|≥200MB|
||NTP时间同步|偏差≤5分钟|
|资源控制|高危服务状态|Telnet/RemoteRegistry禁用|
||防火墙状态|已启用，默认入站阻止|
||SMBv1协议|已卸载/禁用|
|剩余信息保护|关机清除页面文件|已启用|
||WDigest凭据|已禁用|
|入侵防范|自动更新状态|已启用自动安装|
||高危补丁安装|MS17\-010等已修复|
||异常启动项|无未授权持久化入口|
|恶意代码防范|杀毒软件状态|实时保护已启用|
||病毒库更新|7天以内|
||受控文件夹访问|核心目录已启用|

**安服实施注意事项：**所有加固操作前必须备份当前配置（注册表导出、组策略备份、系统还原点）；变更需走客户审批流程，优先在测试环境验证后再批量推广；加固完成后需执行功能回归测试，确保业务系统正常运行。
