---
title: Linux系统安全基线与加固指南（安服视角）
date: 2026-07-16 12:00:00 +0800
categories: [经验分享]
tags: [安全加固, Linux, 基线检查, 等保]
description: 本文档从安全服务（安服）专业视角出发，依据等级保护2.0三级标准与CIS Benchmark最佳实践，围绕身份鉴别、访问控制、安全审计、资源控制、剩余信息保护、入侵防范、恶意代码防范七大维度，系统性梳理Linux操作系统的安全基线检查与加固方案。
---

# Linux系统安全基线与加固指南（安服视角）

# 一、概述

本文档从安全服务（安服）专业视角出发，依据等级保护2\.0三级标准与CIS Benchmark最佳实践，围绕身份鉴别、访问控制、安全审计、资源控制、剩余信息保护、入侵防范、恶意代码防范七大维度，系统性梳理Linux操作系统的安全基线检查与加固方案。适用于CentOS 7/8、RHEL、Ubuntu、银河麒麟、UOS等主流发行版的安全配置合规性检查与加固实施。

**安服实施原则：**所有加固操作前必须完成配置文件备份（/etc目录全量备份），优先通过Ansible批量下发，单台服务器采用配置文件修改\+命令行方式；变更前需客户确认并留存回滚脚本。

自动化检测脚本：[tangjie1/-Baseline-check: 唐门·心法 windows和linux基线检查，配套自动化检查脚本。](https://github.com/tangjie1/-Baseline-check)

# 二、身份鉴别

## 2\.1 账户管理规范

**异常账户清理。**删除或锁定所有无用账户、测试账户、共享账户，禁用或删除UID为0的非root账户；检查空口令账户与无需登录的系统账户shell设置。安服检查时需逐一核对/etc/passwd与/etc/shadow，确认不存在匿名账户与弱UID越权账户。

```bash
# 查看UID为0的账户（应仅root）
awk -F: '$3==0{print $1}' /etc/passwd

# 查看空口令账户
awk -F: '$2=="!"{print $1}' /etc/shadow

# 锁定无用账户
passwd -l testuser
# 或删除账户
userdel -r testuser

# 系统账户设置为nologin
usermod -s /sbin/nologin daemon

```

## 2\.2 密码策略强化

**PAM密码复杂度配置。**通过pam\_pwquality模块配置密码复杂度，要求密码长度不少于12位，包含大小写字母、数字与特殊字符三类组合；密码最长使用期限不超过90天，禁止复用最近5次历史密码。安服核查时需确认/etc/login\.defs与PAM配置文件策略一致。

```bash
# /etc/login.defs 密码老化配置
PASS_MAX_DAYS   90
PASS_MIN_DAYS   7
PASS_MIN_LEN    12
PASS_WARN_AGE   7

# /etc/pam.d/system-auth 密码复杂度配置（CentOS）
password requisite pam_pwquality.so try_first_pass retry=3 minlen=12 ucredit=-1 lcredit=-1 dcredit=-1 ocredit=-1 difok=5

# Ubuntu对应配置文件：/etc/pam.d/common-password
# 安装pam_pwquality模块：apt install libpam-pwquality

```

## 2\.3 登录失败锁定

**防暴力破解配置。**通过pam\_faillock或pam\_tally2模块配置账户锁定策略，连续5次登录失败后锁定账户30分钟。安服视角需重点核查SSH远程登录的暴力破解防护是否生效，同时配合fail2ban实现IP级封禁。

```bash
# CentOS 7+ 使用pam_faillock
# /etc/pam.d/system-auth 添加：
auth required pam_faillock.so preauth silent deny=5 unlock_time=1800
auth [default=die] pam_faillock.so authfail deny=5 unlock_time=1800
account required pam_faillock.so

# 查看锁定状态
faillock --user username

# 手动解锁
faillock --user username --reset

```

## 2\.4 SSH身份认证加固

**SSH密钥登录优先。**禁用root远程登录，禁用密码认证，强制使用SSH密钥对登录；修改默认22端口为非知名端口；配置登录横幅（Banner）警示未授权访问法律后果。核心服务器建议部署双因素认证（Google Authenticator）。

```bash
# /etc/ssh/sshd_config
Port 22222                          # 修改默认端口
PermitRootLogin no                  # 禁止root远程登录
PasswordAuthentication no           # 禁用密码登录
PubkeyAuthentication yes            # 启用密钥登录
PermitEmptyPasswords no             # 禁止空密码
MaxAuthTries 3                      # 最大认证尝试次数
LoginGraceTime 60                   # 登录宽限期
ClientAliveInterval 300             # 空闲超时断开
ClientAliveCountMax 2               # 最大空闲检测次数
AllowUsers admin ops                # 白名单用户（按需配置）

# 重启生效
systemctl restart sshd

```

# 三、访问控制

## 3\.1 文件权限最小化

**敏感文件权限收敛。**严格控制关键配置文件权限：/etc/shadow为600、/etc/gshadow为600、/root目录为700、/etc/passwd为644；检查SUID/SGID文件列表，移除不必要的特殊权限文件，防止提权漏洞利用。

```bash
# 设置敏感文件权限
chmod 600 /etc/shadow /etc/gshadow
chmod 644 /etc/passwd /etc/group
chmod 700 /root

# 查找所有SUID文件
find / -perm -4000 -type f 2>/dev/null

# 查找所有SGID文件
find / -perm -2000 -type f 2>/dev/null

# 移除不必要的SUID位（如非必要）
chmod u-s /usr/bin/passwd

```

## 3\.2 sudo权限管控

**最小权限原则落地。**禁止普通用户直接su到root，通过sudo分配精细化管理权限；审计sudoers配置，确认无NOPASSWD免密越权项；启用sudo操作日志记录，所有提权操作留痕可追溯。

```bash
# /etc/sudoers 配置示例（使用visudo编辑）
# 禁止su切换root，仅允许sudo
auth required pam_wheel.so use_uid

# 用户精细化授权（仅允许执行特定命令）
admin ALL=(ALL) /usr/bin/systemctl, /usr/bin/nginx

# 禁用空密码sudo
Defaults !authenticate

# 启用sudo日志
Defaults logfile=/var/log/sudo.log

# 核查免密sudo项（高危）
grep NOPASSWD /etc/sudoers /etc/sudoers.d/*

```

## 3\.3 强制访问控制（SELinux/AppArmor）

**MAC强制访问控制启用。**CentOS/RHEL系统启用SELinux并设置为enforcing模式，Ubuntu系统启用AppArmor；安服检查时需确认强制访问控制未被随意关闭，这是等保三级合规的关键检查项。

```bash
# 查看SELinux状态
getenforce
sestatus

# 设置为enforcing模式（临时）
setenforce 1

# 永久生效 /etc/selinux/config
SELINUX=enforcing
SELINUXTYPE=targeted

# Ubuntu AppArmor状态检查
aa-status
systemctl status apparmor

```

## 3\.4 共享与目录访问控制

**NFS/Samba权限收紧。**如启用NFS共享，配置no\_root\_squash禁止root映射，限制可访问IP段；Samba共享禁用guest访问，配置独立用户认证。安服核查时需确认共享目录不包含系统敏感路径。

```bash
# /etc/exports NFS配置示例
/data/share 192.168.1.0/24(rw,sync,no_root_squash)

# Samba配置禁用guest
map to guest = Never
guest ok = no

```

# 四、安全审计

## 4\.1 auditd审计服务启用

**全量系统调用审计。**确保auditd服务开机自启并正常运行，配置对敏感文件写入、账户管理、权限变更、系统调用等关键行为的审计规则。安服基线核查需确认审计规则覆盖等保要求的全部审计点。

```bash
# 安装并启用
yum install audit -y  # apt install auditd
systemctl enable --now auditd

# /etc/audit/rules.d/audit.rules 核心规则
# 监控账户相关文件
-w /etc/passwd -p wa -k identity
-w /etc/shadow -p wa -k identity
-w /etc/group -p wa -k identity
-w /etc/sudoers -p wa -k sudo

# 监控登录日志
-w /var/log/lastlog -p wa -k logins
-w /var/run/faillock -p wa -k logins

# 监控权限变更
-a always,exit -F arch=b64 -S chmod,fchmod,chown,fchown -k perm_mod

# 时间修改审计
-a always,exit -S adjtimex,settimeofday -k time-change

```

## 4\.2 日志服务与留存

**rsyslog日志规范化。**确保rsyslog服务正常运行，配置日志轮转（logrotate），单日志文件设置容量上限与留存周期；核心服务器配置远程日志转发至集中日志服务器（SIEM），本地安全日志留存不少于180天。

```bash
# 查看rsyslog状态
systemctl status rsyslog

# /etc/logrotate.conf 关键配置
weekly          # 每周轮转
rotate 12       # 保留12份
compress        # 压缩旧日志
dateext         # 日期后缀

# 远程日志转发（/etc/rsyslog.conf）
*.* @@192.168.1.100:514    # TCP方式转发

# 查看审计日志
ausearch -k identity
aureport -au    # 认证报告

```

## 4\.3 关键审计事件清单

|日志文件|记录内容|安服关注要点|
|---|---|---|
|/var/log/secure|SSH登录、sudo、认证事件|暴力破解、异常登录IP|
|/var/log/messages|系统全局事件|服务异常、内核告警|
|/var/log/audit/audit\.log|auditd系统调用审计|敏感文件篡改、提权操作|
|/var/log/cron|计划任务执行|恶意计划任务持久化|
|/var/log/sudo\.log|sudo提权操作|越权操作溯源|
|/var/log/btmp|失败登录记录|暴力破解攻击统计|

## 4\.4 时间同步配置

**NTP时间校准。**配置chronyd或ntpd服务与内部NTP服务器同步，确保日志时间戳准确统一，为跨设备日志关联分析提供时间基准。安服核查时确认时间偏差不超过5分钟。

```bash
# chrony配置（CentOS 7+推荐）
yum install chrony -y
# /etc/chrony.conf
server ntp.aliyun.com iburst
systemctl enable --now chronyd

# 查看同步状态
chronyc sources
timedatectl status

```

# 五、资源控制

## 5\.1 服务与端口精简

**攻击面缩减。**关闭所有非业务必需的系统服务，包括telnet、rsh、rlogin、talk、ntalk等高危旧协议服务；禁用不必要的自启动服务。安服基线检查需逐一核对开放端口与运行服务清单，确认端口与业务一一对应。

```bash
# 查看所有监听端口
ss -tulnp
netstat -tulnp

# 查看开机自启服务
systemctl list-unit-files --type=service | grep enabled

# 禁用高危服务
systemctl disable --now telnet.socket
systemctl disable --now rsh.socket
systemctl disable --now rexec.socket

# 关闭不必要服务示例
systemctl disable --now avahi-daemon
systemctl disable --now cups

```

## 5\.2 防火墙策略加固

**白名单准入原则。**启用firewalld或iptables防火墙，默认拒绝所有入站流量，仅放行业务必需端口；管理端口（SSH）绑定指定管理IP段。安服实施时入站规则需逐条审核，遵循最小开放原则。

```bash
# 启用firewalld
systemctl enable --now firewalld

# 默认拒绝入站
firewall-cmd --set-default-zone=drop

# 放行业务端口
firewall-cmd --permanent --add-port=443/tcp
firewall-cmd --permanent --add-port=80/tcp

# SSH仅允许管理网段
firewall-cmd --permanent --zone=trusted --add-source=192.168.1.0/24
firewall-cmd --permanent --zone=trusted --add-port=22222/tcp

# 重载生效
firewall-cmd --reload
firewall-cmd --list-all

```

## 5\.3 系统资源限制

**资源滥用防护。**通过/etc/security/limits\.conf配置用户级资源限制，包括最大进程数、最大文件句柄数、核心转储文件大小等，防止fork炸弹、文件句柄耗尽等DoS攻击。

```bash
# /etc/security/limits.conf
# 核心转储禁用（防止敏感信息泄露）
* soft core 0
* hard core 0

# 最大进程数限制
* soft nproc 4096
* hard nproc 8192

# 最大文件句柄数
* soft nofile 65535
* hard nofile 65535

# root不受限
root soft nproc unlimited
root hard nproc unlimited

```

## 5\.4 内核参数加固

**网络内核安全调优。**配置sysctl内核参数，开启SYN Cookies防DDoS、禁用ICMP重定向、禁用源路由、开启反向路径过滤等。安服核查时重点确认网络层防护参数是否生效。

```bash
# /etc/sysctl.conf 网络安全加固
# 开启SYN Cookie防护SYN Flood
net.ipv4.tcp_syncookies = 1

# 禁用IP源路由
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0

# 启用反向路径过滤（防IP欺骗）
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# 禁用ICMP重定向
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0

# 忽略广播ICMP请求
net.ipv4.icmp_echo_ignore_broadcasts = 1

# 生效
sysctl -p

```

# 六、剩余信息保护

## 6\.1 内存与缓存清理

**敏感数据残留防护。**配置定期清理页面缓存与swap空间；核心数据处理服务器建议禁用swap分区或设置swappiness=0，防止内存数据写入磁盘被离线提取；注销时清除用户历史命令记录。

```bash
# 降低swap使用倾向
echo "vm.swappiness = 0" >> /etc/sysctl.conf
sysctl -p

# 用户注销清空历史命令
echo "history -c && rm -f ~/.bash_history" >> ~/.bash_logout

# 限制历史命令记录数量
echo "HISTSIZE=50" >> /etc/profile
echo "HISTFILESIZE=50" >> /etc/profile

# 临时清理缓存
sync && echo 3 > /proc/sys/vm/drop_caches

```

## 6\.2 临时目录与回收站

**/tmp目录安全加固。**为/tmp目录挂载独立分区并设置nosuid、noexec、nodev选项，防止临时目录提权；配置systemd\-tmpfiles定期清理临时文件，默认清理超过10天的临时文件。

```bash
# fstab挂载选项（独立分区时）
UUID=xxx /tmp ext4 defaults,nosuid,noexec,nodev 0 0

# 无独立分区时创建tmpfs挂载
echo "tmpfs /tmp tmpfs defaults,nosuid,noexec,nodev 0 0" >> /etc/fstab

# /var/tmp同样加固
mount -o bind,nosuid,noexec,nodev /tmp /var/tmp

```

## 6\.3 会话超时自动注销

**空闲会话自动断开。**配置Shell空闲超时自动注销，设置TMOUT环境变量为300秒（5分钟），防止管理员遗忘的终端会话被未授权使用。SSH层面同步配置ClientAliveInterval实现双保险。

```bash
# /etc/profile 全局空闲超时（5分钟）
export TMOUT=300
readonly TMOUT

# 各用户bashrc同步配置
echo "export TMOUT=300" >> /etc/bashrc
echo "readonly TMOUT" >> /etc/bashrc

# 屏幕自动锁定（图形界面）
gsettings set org.gnome.desktop.session idle-delay 300

```

## 6\.4 数据删除与介质清理

**彻底删除防止恢复。**敏感文件删除使用shred命令覆写，而非普通rm；磁盘退役或报废前需执行全盘安全擦除（dd或专用擦除工具），确保数据不可恢复。安服数据销毁场景需出具擦除报告。

```bash
# 安全删除文件（覆写3次）
shred -u -n 3 /path/to/sensitive_file

# 全盘擦除（谨慎使用！）
dd if=/dev/urandom of=/dev/sda bs=4M status=progress

# 清空内存数据（重启前）
sync && echo 3 > /proc/sys/vm/drop_caches && swapoff -a && swapon -a

```

# 七、入侵防范

## 7\.1 补丁与漏洞管理

**安全补丁闭环。**配置自动安全更新（yum\-cron/unattended\-upgrades），高危漏洞补丁72小时内完成修复；建立补丁分级机制，区分紧急、重要、一般等级别。安服视角需定期执行漏洞扫描，确保无CVE高危漏洞遗留。

```bash
# CentOS yum-cron自动安全更新
yum install yum-cron -y
# /etc/yum/yum-cron.conf
update_cmd = security
apply_updates = yes
systemctl enable --now yum-cron

# Ubuntu无人值守升级
apt install unattended-upgrades -y
dpkg-reconfigure -plow unattended-upgrades

# 查看可更新安全包
yum updateinfo list security
apt list --upgradable

```

## 7\.2 暴力破解防护

**fail2ban自动封禁。**部署fail2ban工具，监控SSH、FTP等服务登录日志，连续失败自动封禁IP；配置封禁时长与白名单网段。安服基线核查需确认暴力破解防护已部署，封禁策略有效。

```bash
# 安装
yum install fail2ban -y  # apt install fail2ban

# /etc/fail2ban/jail.local SSH防护
[sshd]
enabled = true
port = 22222
maxretry = 3
bantime = 86400
findtime = 600
ignoreip = 127.0.0.1 192.168.1.0/24

systemctl enable --now fail2ban

# 查看封禁状态
fail2ban-client status sshd
fail2ban-client banned

```

## 7\.3 文件完整性监控

**AIDE入侵检测。**部署AIDE（Advanced Intrusion Detection Environment）建立系统文件基线数据库，定期扫描比对文件哈希、权限、大小变化，检测木马后门与未授权文件篡改。安服应急响应场景下AIDE数据库是重要取证依据。

```bash
# 安装
yum install aide -y

# 初始化基线数据库
aide --init
mv /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz

# 执行完整性检查
aide --check

# 更新基线（确认变更合法后）
aide --update

# 配置定期检查（cron每日执行）
0 2 * * * /usr/sbin/aide --check | mail -s "AIDE Report" admin@example.com

```

## 7\.4 持久化入口排查

**后门与持久化审计。**定期排查计划任务（crontab）、系统服务、rc\.local启动脚本、/etc/profile\.d、bashrc、SSH authorized\_keys等常见持久化入口；检查rootkit与后门程序（chkrootkit/rkhunter）。安服应急响应需全面排查所有持久化点位。

```bash
# 查看所有用户crontab
for user in $(cut -f1 -d: /etc/passwd); do echo "=== $user ==="; crontab -u $user -l 2>/dev/null; done

# 查看系统级计划任务
ls -la /etc/cron.* /var/spool/cron/

# 检查rc.local与启动脚本
cat /etc/rc.d/rc.local
ls -la /etc/profile.d/

# SSH密钥核查
cat /root/.ssh/authorized_keys
find /home -name authorized_keys -exec cat {} \;

# rootkit检测
yum install rkhunter -y
rkhunter --check

```

# 八、恶意代码防范

## 8\.1 杀毒软件部署

**终端防护全覆盖。**服务器部署ClamAV开源杀毒软件或商业EDR产品，配置每日全盘扫描与实时监控；病毒库保持自动更新。企业环境建议统一部署终端安全管理平台，集中管控与告警。

```bash
# 安装
yum install clamav clamav-update -y

# 更新病毒库
freshclam

# 全盘扫描
clamscan -r / --bell -i

# 定时扫描（cron每日凌晨3点）
0 3 * * * /usr/bin/clamscan -r / --exclude-dir=/sys --exclude-dir=/proc -l /var/log/clamav/scan.log

# 实时监控（clamonacc）
systemctl enable --now clamav-freshclam

```

## 8\.2 WebShell与挖矿检测

**常见恶意代码专项排查。**Web服务器定期扫描WebShell（使用find与特征匹配或专用工具）；检查异常挖矿进程（CPU占用异常、可疑矿池连接）；定时任务与启动项中排查挖矿程序持久化痕迹。安服应急响应需重点关注CPU异常升高的服务器。

```bash
# 查找可疑PHP WebShell特征
find /var/www/html -name "*.php" -exec grep -l "eval\|base64_decode\|assert\|system(" {} \;

# 查看CPU占用异常进程
top -bn1 | head -20
ps aux --sort=-%cpu | head -10

# 查看可疑外连（矿池常用端口：3333, 8899等）
ss -tulnp | grep -E "3333|8899|14444"
netstat -antp | grep ESTABLISHED

# 检查异常定时任务挖矿
grep -r "curl\|wget\|python\|miner" /etc/cron* /var/spool/cron/

```

## 8\.3 病毒库更新与告警

**特征库持续更新。**配置杀毒软件自动每日更新病毒特征库；扫描发现威胁自动告警至安全管理员；隔离区文件定期审核，确认误报后再处理。安服检查时需确认病毒库版本不超过7天，隔离区无积压未处理威胁。

```bash
# ClamAV自动更新配置 /etc/freshclam.conf
Checks 24                # 每日检查24次
DatabaseMirror db.cn.clamav.net
NotifyClamd /etc/clamd.d/scan.conf

# 告警脚本示例（发现病毒发送邮件）
OnVirusFound /usr/local/bin/virus_alert.sh

```

## 8\.4 脚本与可执行文件管控

**未知程序执行限制。**通过SELinux/AppArmor限制进程执行范围；Web目录禁止脚本执行权限；临时目录/tmp设置noexec挂载选项。核心服务器建议部署应用白名单机制，仅允许授权程序运行。

```bash
# Web目录移除执行权限（上传目录）
chmod -R 644 /var/www/html/upload
find /var/www/html/upload -type d -exec chmod 755 {} \;

# 查找全局可写目录（高危）
find / -type d -perm -o+w 2>/dev/null | grep -v proc | grep -v sys

# 检查隐藏可疑文件
find /tmp /var/tmp /dev/shm -name ".*" -type f 2>/dev/null

```

# 九、附录：安服基线核查速查表

|加固维度|核心检查项|合规标准|
|---|---|---|
|身份鉴别|root远程登录|已禁用（PermitRootLogin no）|
||SSH密码登录|已禁用，密钥登录|
||密码复杂度|≥12位，三类字符组合|
||登录失败锁定|5次失败锁定30分钟|
|访问控制|敏感文件权限|/etc/shadow为600|
||SELinux/AppArmor|enforcing/enabled模式|
||sudo免密项|无NOPASSWD越权配置|
|安全审计|auditd服务|已启用，规则覆盖敏感文件|
||日志留存周期|≥180天|
||NTP时间同步|偏差≤5分钟|
|资源控制|防火墙状态|已启用，默认入站拒绝|
||高危服务|telnet/rsh等已禁用|
||内核网络参数|SYN Cookie、反向过滤已开启|
|剩余信息保护|会话超时|5分钟自动注销|
||/tmp挂载选项|nosuid,noexec,nodev|
|入侵防范|自动安全更新|已启用|
||fail2ban防护|已部署，SSH封禁生效|
||AIDE完整性监控|基线已建立，定期扫描|
|恶意代码防范|杀毒软件|已安装，实时监控启用|
||病毒库更新|7天以内|
||定时全盘扫描|每日/每周执行|

**安服实施注意事项：**所有加固操作前必须备份/etc全目录与关键配置文件；SELinux变更需在测试环境充分验证，避免业务中断；防火墙规则修改建议先临时添加测试，确认无问题再permanent永久保存；加固完成后执行业务功能回归测试与远程连通性验证。
