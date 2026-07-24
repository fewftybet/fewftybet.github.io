---
title: Web中间件安全基线与加固指南（安服视角）
date: 2026-07-23 09:00:00 +0800
categories: [经验分享, 安全基线与加固]
tags: [安全加固, Web中间件, Nginx, Tomcat, Apache, IIS, Redis, RabbitMQ]
description: 从安服视角出发，系统梳理Nginx、Tomcat、Apache、IIS、Redis、RabbitMQ等常见Web中间件的安全基线与加固方案，涵盖配置规范、访问控制、日志审计等核心维度。
---

# Web中间件安全基线与加固指南（安服视角）

## 中间件部署规范

### 0x01 Nginx安全部署规范

> **配置文件路径**：主配置文件为 `/etc/nginx/nginx.conf`（CentOS/RHEL）或 `/etc/nginx/nginx.conf`（Debian/Ubuntu），站点配置文件位于 `/etc/nginx/conf.d/` 或 `/etc/nginx/sites-available/` 目录下。
>
> **修改后执行**：`nginx -t && systemctl reload nginx`

#### 1. 隐藏版本号

在 `nginx.conf` 的 `http` 块中添加：

```coffeescript
http {
    server_tokens off;
}
```

#### 2. 开启HTTPS

在 `nginx.conf` 的 `server` 块中配置：

> `ssl on`:开启https
> `ssl_certificate`:配置nginx ssl证书的路径
> `ssl_certificate_key`:配置nginx ssl证书key的路径
> `ssl_protocols`:指定客户端建立连接时使用的ssl协议版本
> `ssl_ciphers`:指定客户端连接时所使用的加密算法

```cobol
server {
    listen 443;
    server_name <xxx>;
 
    ssl on;
    ssl_certificate <pem路径>;
    ssl_certificate_key <key路径>;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers HIGH:!aNULL:!MD%

}
```

#### 3. 限制请求方法

在 `nginx.conf` 的 `server` 块中配置：

> $request_method能获取到请求时所使用的method，应该配置只使用GET/POST方法访问，其他的method返回405

```cobol
if ($request_method !~ ^(GET|POST)$ ){
    return 405;
}
```

#### 4. 拒绝某些User-Agent

在 `nginx.conf` 的 `server` 块中配置：

> 禁止一些爬虫的扫描

```ruby
if ($http_user_agent ~* LWP::Simple|BBBike|wget|curl){
    return 444;
}
```

#### 5. 利用referer图片防盗链

在 `nginx.conf` 的 `server` 块中添加 `location` 配置：

```cobol
location ~* \.(gif|jpg|png|jpeg|bmp)$ {
    valid_referers none blocked <domain_name> <domain_name>;
    if ($invalid_referer){
        return 403;
    }
}
```

> `valid_referers`:验证referer
> none：允许referer为空
> blocked：允许不带协议的请求

#### 6. 控制并发连接数

在 `nginx.conf` 的 `http` 块中定义连接限制区域，在 `server` 或 `location` 块中应用：

```cobol
http {
    limit_conn_zone $binary_remote_addr zone=ops:10m;
    limit_conn_zone $server_name zone=coffee:10m;
 
    server {
        listen 80
        server_name <server_name>;
        ...
        location / {
            limit_conn opos 10; # 限制单一IP来源的连接数为10
            limit_conn coffee 2000; # 限制单一虚拟服务器的总连接数为2000 
            limit_rate 500k; # 限制单个连接使用的带宽
        }
    }
}
```

> `limit_conn_zone`:设定保存各个属性状态的共享内存空间的参数
> `limit_conn`:为已经设定zone的属性设置最大连接数

#### 7. 设置缓冲区大小防止缓冲区溢出攻击

在 `nginx.conf` 的 `http` 或 `server` 块中配置：

```cobol
client_body_buffer_size 1K;
client_header_buffer_size 1K;
client_max_body_size 1K;
large_client_header_buffers 2 1K;
```

> `client_body_buffer_size`:默认8k或16k，标识客户端请求body占用缓冲区的大小。如果连接请求超过指定缓冲区大小的值，尼玛这些请求实体的整体或者部分将尝试写入一个临时文件
> `client_header_buffer_size`:表示客户端请求头部的缓冲区大小。绝大多数情况下一个请求头不会大于1K，如果大于1K，Nginx将分配给它一个更大的缓冲区，，这个值可以在`large_client_header_buffers`中设置
> `client_max_body_size`:标识客户端请求的最大可接受的body大小。如果请求头部的Content-Length字段的值大于该值，客户端将收到一个(413)状态码的错误。【会影响上传文件的功能】
> `large_client_header_buffers`:表示一些比较大的请求头使用的缓冲区数量和大小，默认一个缓冲区大小为操作系统中分页文件大小，通常是4k或8k。请求字段不能大于一个缓冲区的大小，若大于，则nginx将返回400状态码的错误

**设置超时时间**

在 `nginx.conf` 的 `http` 块中配置：

```cobol
client_body_timeout 10
client_header_timeout 10;
keepalive_timeout 5 5;
send_timeout 10;
```

> `client_body_timeout`：表示读取请求body的超时时间，如果连接超过这个时间而客户端没有任何响应，Nginx将返回**"Request time out" (408)**错误
> `client_header_timeout`：表示读取客户端请求头的超时时间，如果连接超过这个时间而客户端没有任何响应，Nginx将返回**"Request time out" (408)**错误
> `keepalive_timeout`：参数的第一个值表示客户端与服务器长连接的超时时间，超过这个时间，服务器将关闭连接，可选的第二个参数参数表示Response头中Keep-Alive: timeout=time的time值，这个值可以使一些浏览器知道什么时候关闭连接，以便服务器不用重复关闭，如果不指定这个参数，nginx不会在应Response头中发送Keep-Alive信息
> `send_timeout`：表示发送给客户端应答后的超时时间，Timeout是指没有进入完整**established**状态，只完成了两次握手，如果超过这个时间客户端没有任何响应，nginx将关闭连接

#### 8. 添加Header头防止XSS攻击

在 `nginx.conf` 的 `server` 或 `location` 块中添加：

```cobol
add_header X-Frame-Options "SAMEORIGIN";
add_header X-XSS-Protection "1; mode=block";
add_header X-Content-Type-Options "nosniff";
```

> `X-Frame-Options`:标识是否允许浏览器加载frame等属性。
> `X-XSS-Protection`:启用XSS过滤。mode=block标识若检查到XSS攻击则停止渲染页面
> `X-Content-Type-Options`:用来指定浏览器对未指定或错误指定Content-Type资源真正类型的猜测行为

#### 9. 添加其他Header头

在 `nginx.conf` 的 `server` 或 `location` 块中添加：

```cobol
add_header Content-Security-Policy "default-src 'self'";
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains";
```

> `Content-Security-Policy`:表示页面可以加载哪些资源
> `Strict-Transport-Security`:表示告诉浏览器要用HTTPS协议代替HTTP来访问目标站点

### 0x02 Tomcat安全部署规范

> **配置文件路径**：主配置文件为 `$CATALINA_HOME/conf/server.xml`，Web应用配置为 `$CATALINA_HOME/conf/web.xml`，用户配置为 `$CATALINA_HOME/conf/tomcat-users.xml`。
>
> **修改后执行**：`systemctl restart tomcat` 或 `$CATALINA_HOME/bin/shutdown.sh && $CATALINA_HOME/bin/startup.sh`

#### 1. 更改Server Header

在 `$CATALINA_HOME/conf/server.xml` 的 `<Connector>` 标签中添加 `server` 属性：

```cobol
<Connector port="8080" server="webserver" />\n
```

#### 2. 保护telnet管理端口

在 `$CATALINA_HOME/conf/server.xml` 中修改根 `<Server>` 元素的 `port` 和 `shutdown` 属性：

- 修改默认的8005管理端口或禁用
- 修改`SHUTDOWN`命令为其他字符串

> 若开启，则可以使用nc或者telnet发送指令直接关闭tomcat

```cobol
<Server port="8578" shutdown="close" > # 更改
<Server port="-1" shutdown="close" >   # 禁用
```

#### 3. 保护AJP连接端口

在 `$CATALINA_HOME/conf/server.xml` 中修改或注释 `<Connector>` 的 AJP 配置：

- 修改或禁用默认的AJP8009端口，但配置在8000-8999之间

```cobol
<Connector port="8349" protocol="AJP/1.3" /> # 修改
<!--<Connector port="8349" protocol="AJP/1.3" /> --> # 禁用
```

#### 4. 删除默认文档和示例程序

- 删除默认的 `$CATALINA_HOME/conf/tomcat-users.xml` 文件（或清空其中的用户配置）
- 删除 `$CATALINA_HOME/webapps` 下的默认目录和文件：`docs`、`examples`、`host-manager`、`manager`、`ROOT`
- 将tomcat应用根目录配置为tomcat安装目录以外的目录

```bash
rm -rf $CATALINA_HOME/webapps/docs
rm -rf $CATALINA_HOME/webapps/examples
rm -rf $CATALINA_HOME/webapps/host-manager
rm -rf $CATALINA_HOME/webapps/manager
rm -rf $CATALINA_HOME/webapps/ROOT
```

#### 5. 隐藏版本信息（需要解jar包）

> 针对该信息的显示是由一个jar包控制的，该jar包存放在 `$CATALINA_HOME/lib` 目录下，名称为 `catalina.jar`，通过 `jar xf` 命令解压这个jar包会得到两个目录 `META-INF` 和 `org`，修改 `org/apache/catalina/util/ServerInfo.properties` 文件中的 `server.info` 和 `server.number` 字段来实现更改tomcat版本信息的目的。

```bash
cd $CATALINA_HOME/lib
jar xf catalina.jar
# 查看当前版本信息
cat org/apache/catalina/util/ServerInfo.properties |grep -v "^$|#"
# 修改版本信息
vim org/apache/catalina/util/ServerInfo.properties
# 将以下值修改为自定义内容：
# server.info=WebServer
# server.number=1.0.0
# 修改后重新打包
jar uf catalina.jar org/apache/catalina/util/ServerInfo.properties
```

#### 6. 专职低权限用户启动tomcat

- 创建专门的用户来启动tomcat：`useradd -s /sbin/nologin tomcat`
- 修改安装目录属主：`chown -R tomcat:tomcat $CATALINA_HOME`
- 修改 `bin/` 目录权限：`chmod -R 750 $CATALINA_HOME/bin`
- 将Tomcat和web项目的属主分离

#### 7. 文件列表访问控制

在 `$CATALINA_HOME/conf/web.xml` 中，找到 `<servlet>` 下name为 `default` 的Servlet，将 `init-param` 中的 `listings` 参数值改为 `false`：

```cobol
<servlet>
    <servlet-name>default</servlet-name>
    <servlet-class>org.apache.catalina.servlets.DefaultServlet</servlet-class>
    <init-param>
        <param-name>listings</param-name>
        <param-value>false</param-value>
    </init-param>
</servlet>
```

#### 8. 脚本权限回收

- `$CATALINA_HOME/bin`目录下的`startup.sh,catalina.sh,shutdown.sh`的权限不高于750

```cobol
chmod 750 $CATALINA_HOME/bin/*.sh
```

### 0x03 Apache安全部署规范

> **配置文件路径**：主配置文件为 `/etc/httpd/conf/httpd.conf`（CentOS/RHEL）或 `/etc/apache2/apache2.conf`（Debian/Ubuntu），额外配置目录为 `/etc/httpd/conf.d/` 或 `/etc/apache2/conf-enabled/`，虚拟主机配置在 `/etc/httpd/conf.d/vhost.conf` 或 `/etc/apache2/sites-available/`。
>
> **修改后执行**：`apachectl configtest && systemctl restart httpd`（或 `apache2ctl configtest && systemctl restart apache2`）

#### 1. 专职低权限用户运行Apache服务

在终端创建用户后，在 `httpd.conf` 中修改运行用户和组：

```sql
# 创建用户和组
groupadd apache
useradd apache -g apache

# 在 httpd.conf 中修改：
User apache
Group apache
```

#### 2. 目录访问权限设置

**2.1 非超级用户不能修改Apache主目录中的内容**

在 `httpd.conf` 中设置 `ServerRoot` 并确保目录权限正确：

```perl
# httpd.conf
ServerRoot /usr/local/apache # 主目录
```

```bash
# 设置Apache安装目录权限（CentOS/RHEL）
chown -R root:root /usr/local/apache
chmod -R 755 /usr/local/apache
```

**2.2 配置文件和日志文件权限**

- 配置文件 `/etc/httpd/conf/httpd.conf`（或 `/etc/apache2/apache2.conf`）的权限不高于 `600`
- 日志文件 `/var/log/httpd/*.log`（或 `/var/log/apache2/*.log`）日志文件的权限不高于 644

```bash
chmod 600 /etc/httpd/conf/httpd.conf
chmod 644 /var/log/httpd/*.log
```

#### 3. 日志设置

在 `httpd.conf` 中配置日志相关指令：

**3.1 错误日志**

```cobol
LogLevel notice # 日志的级别
ErrorLog /var/log/httpd/error_log
```

**3.2 访问日志**

```cobol
LogFormat %h %l %u %t \"%r\" %>s %b \"%{Accept}i\" \"%{Referer}i\" \"%{User-Agent}i\"" combined
CustomLog /var/log/httpd/access_log combined
```

> `ErrorLog`：设置错误日志文件名和位置。
> `CustomLog`: 指定保存日志文件的具体位置和日志的格式。
> `LogFormat`: 设置日志格式，建议设置为combined格式
> `LogLevel`: 用于调整记录在错误日志中的信息的详细程度

#### 4. 只允许访问web目录下的文件

在 `httpd.conf` 中配置 `<Directory>` 指令，限制Apache只能访问web目录：

```cobol
# httpd.conf
<Directory />
    Order Deny,Allow
    Deny from all
    Options None
    AllowOverride None
</Directory>
<Directory /var/www/html>
    Order Allow,Deny
    Allow from all
</Directory>
```

#### 5. 禁止列出目录

在 `httpd.conf` 的 `<Directory>` 块中修改 `Options` 指令，删除 `Indexes`：

```csharp
# httpd.conf
# Options Indexes FollowSymLinks
Options FollowSymlinks # 删去Indexes
```

#### 6. 防范拒绝服务

在 `httpd.conf` 中设置超时和连接相关参数：

```cobol
# httpd.conf
Timeout 10
KeepAlive On
KeepAliveTimeout 15    # 限制每个session的保持时间为15秒
MaxKeepAliveRequests 100
```

#### 7. 隐藏版本号

在 `httpd.conf` 中添加或修改：

```csharp
# httpd.conf
ServerSignature Off 
ServerTokens Prod
```

#### 8. 关闭TRACE功能

在 `httpd.conf` 中添加：

```csharp
# httpd.conf
TraceEnable Off
```

#### 9. 绑定监听地址

在 `httpd.conf` 中修改 `Listen` 指令，将Apache绑定到指定IP和端口：

```ruby
# httpd.conf
# 原来是 Listen 80，改为只监听内网IP
Listen 192.168.1.10:80
```

#### 10. 删除默认安装的无用文件

删除以下默认文件，减少信息泄露和攻击面：

**10.1 删除默认的html文件**

```bash
# 默认网站根目录：/usr/local/apache2/htdocs/（源码安装）或 /var/www/html/（包管理器安装）
rm -rf /usr/local/apache2/htdocs/*
```

**10.2 删除默认的CGI脚本**

```bash
# CGI目录：/usr/local/apache2/cgi-bin/（源码安装）或 /usr/lib/cgi-bin/（包管理器安装）
rm -rf /usr/local/apache2/cgi-bin/*
```

**10.3 删除Apache的说明文件**

```bash
rm -rf /usr/local/apache2/manual
```

**10.4 删除源代码文件**

```bash
rm -rf /path/to/httpd-2.2.4*
```

#### 11. 限定可以使用的HTTP方法

在 `httpd.conf` 中使用 `<LimitExcept>` 指令限制HTTP方法：

- 禁用PUT,DELETE等危险的HTTP方法
- 只允许GET，POST方法

```cobol
# httpd.conf
<Location />
<LimitExcept GET POST CONNECT OPTIONS>
    Order Allow,Deny
    Deny from all
</LimitExcept>
</Location>
```

### 0x04 IIS安全部署规范

> **配置文件路径**：
> - 全局配置：`%SystemRoot%\System32\inetsrv\Config\applicationHost.config`（IIS 7+）
> - 站点配置：各站点根目录下的 `web.config`
> - 也可以通过 **IIS管理器**（`inetmgr`）图形界面进行配置
>
> **修改后执行**：在IIS管理器中点击"应用"或命令行执行 `iisreset` / `net stop w3svc && net start w3svc`

#### 1. 限制目录的执行权限

- **操作方式**：在IIS管理器中选择上传目录 → 功能视图中双击"处理程序映射" → 编辑功能权限 → 取消"脚本"执行权限
- **或在 `web.config` 中配置**：

```csharp
<configuration>
  <system.webServer>
    <handlers>
      <clear />
      <!-- 只保留静态文件映射 -->
      <add name="StaticFile" path="*" verb="GET,HEAD" modules="StaticFileModule" resourceType="Either" />
    </handlers>
  </system.webServer>
</configuration>
```

> 可以做到即使上传了shell文件，也无法解析执行

#### 2. 开启日志记录功能

- **操作方式**：在IIS管理器中选择站点 → 双击"日志" → 启用日志记录
- **日志存放路径**：默认 `%SystemDrive%\inetpub\logs\LogFiles\`
- **设置日志文件的权限**：只允许 `SYSTEM` 和 `Administrators` 组读写

#### 3. 自定义错误页面

- **操作方式**：在IIS管理器中选择站点 → 双击"错误页" → 编辑功能设置 → 选择"自定义错误页"
- **在 `web.config` 中配置**：

```csharp
<configuration>
  <system.webServer>
    <httpErrors errorMode="Custom">
      <remove statusCode="404" />
      <error statusCode="404" path="/404.html" responseMode="ExecuteURL" />
    </httpErrors>
  </system.webServer>
</configuration>
```

#### 4. 关闭目录浏览功能

- **操作方式**：在IIS管理器中选择站点 → 双击"目录浏览" → 点击"禁用"

#### 5. 停用或删除默认站点

- **操作方式**：在IIS管理器中右键"Default Web Site" → 停止 或 删除
- 删除IIS安装时默认的`Default Web Site`

#### 6. 删除不必要的脚本映射

- **操作方式**：在IIS管理器中选择服务器节点 → 双击"处理程序映射" → 删除不需要的映射（如 `.ida`、`.idq`、`.htw` 等）
- 只保留网站需要的脚本映射（如 `.aspx`、`.asp` 等）

#### 7. 专职低权限用户运行网站

- **操作方式**：
  1. 创建Windows本地用户：`net user iisuser <password> /add`
  2. 在IIS管理器中选择应用程序池 → 高级设置 → 标识（Identity） → 选择 `iisuser`
  3. 或将站点的身份验证的匿名用户指定为低权限用户

#### 8. 在独立的应用程序池中运行网站

- **操作方式**：在IIS管理器中右键"应用程序池" → 添加应用程序池 → 为每个网站指定独立的应用程序池
- 若同一台IIS中运行多个网站，可以每个网站都运行在单独的应用程序池中，避免相互影响

### 0x05 Redis安全部署规范

> **配置文件路径**：`/etc/redis/redis.conf`（通过包管理器安装）或自定义路径 `/usr/local/redis/redis.conf`。
>
> **修改后执行**：`systemctl restart redis` 或 `redis-cli shutdown && redis-server /etc/redis/redis.conf`

#### 1. 专职低权限用户启动redis

- 创建专门的redis用户和组来运行redis：

```bash
groupadd redis
useradd -r -g redis -s /sbin/nologin redis
chown -R redis:redis /var/lib/redis
chown -R redis:redis /var/log/redis
```

- 修改服务文件 `/etc/systemd/system/redis.service`（或 `/usr/lib/systemd/system/redis.service`）中的 `User` 和 `Group`：

```ini
[Service]
User=redis
Group=redis
```

#### 2. 限制redis配置文件的访问权限

- 配置文件 `/etc/redis/redis.conf` 的权限设置为 `600`，只允许redis用户读取：

```bash
chown redis:redis /etc/redis/redis.conf
chmod 600 /etc/redis/redis.conf
```

#### 3. 更改默认端口

在 `/etc/redis/redis.conf` 中修改 `port` 指令：

```csharp
# redis.conf
port 8379
```

#### 4. 开启redis密码认证

在 `/etc/redis/redis.conf` 中取消注释并设置 `requirepass`：

```csharp
# redis.conf
requirepass YourStrongPassword123!
```

#### 5. 禁用或者重命名危险命令

在 `/etc/redis/redis.conf` 中添加或修改 `rename-command` 指令：

```perl
# redis.conf
rename-command FLUSHALL ""
rename-command FLUSHDB ""
rename-command CONFIG ""
rename-command KEYS ""
rename-command SHUTDOWN ""
rename-command DEL ""
rename-command EVAL ""
```

#### 6. 禁止监听在公网IP

在 `/etc/redis/redis.conf` 中修改 `bind` 指令：

```perl
# redis.conf
bind 127.0.0.1
# OR
bind <内网IP>
```

#### 7. 开启保护模式

在 `/etc/redis/redis.conf` 中确保 `protected-mode` 为 `yes`：

```csharp
# redis.conf
protected-mode yes
```

> 开启保护模式后，若没有指定bind和密码，则只能本地访问redis

### 0x06 RabbitMQ规范

> **配置文件路径**：
> - 主配置文件：`/etc/rabbitmq/rabbitmq.conf`（RabbitMQ 3.7+）或 `/etc/rabbitmq/rabbitmq.config`（旧版Erlang格式）
> - 环境配置文件：`/etc/rabbitmq/rabbitmq-env.conf`
> - 插件列表：`/etc/rabbitmq/enabled_plugins`
>
> **修改后执行**：`systemctl restart rabbitmq-server`

#### 1. 专职低权限用户启动RabbitMQ

- 创建专门的用户和组来启动RabbitMQ：

```bash
groupadd rabbitmq
useradd -r -g rabbitmq -s /sbin/nologin rabbitmq
chown -R rabbitmq:rabbitmq /var/lib/rabbitmq
chown -R rabbitmq:rabbitmq /var/log/rabbitmq
```

- 修改服务配置中的运行用户（路径因安装方式而异）

#### 2.配置SSL证书

在 `/etc/rabbitmq/rabbitmq.conf` 中添加SSL监听配置：

```erlang
# rabbitmq.conf（新版ini格式）
listeners.ssl.default = 5671

ssl_options.cacertfile = /etc/rabbitmq/ssl/ca.crt
ssl_options.certfile   = /etc/rabbitmq/ssl/server.crt
ssl_options.keyfile    = /etc/rabbitmq/ssl/server.key
ssl_options.verify     = verify_peer
ssl_options.fail_if_no_peer_cert = false
```

#### 3. 开启HTTP后台认证

- 启用 HTTP 后台认证需要使用 `rabbitmq_auth_backend_http` 插件：

```bash
rabbitmq-plugins enable rabbitmq_auth_backend_http
rabbitmq-plugins enable rabbitmq_auth_backend_cache
```

- 在 `/etc/rabbitmq/rabbitmq.conf` 中配置：

```erlang
auth_backends.1 = http
auth_http.http_method   = post
auth_http.user_path     = http://localhost:8080/auth/user
auth_http.vhost_path    = http://localhost:8080/auth/vhost
auth_http.resource_path = http://localhost:8080/auth/resource
auth_http.topic_path    = http://localhost:8080/auth/topic
```

#### 4. 删除或修改默认的guest用户和密码

```bash
# 检查是否有默认用户名guest
sudo rabbitmqctl list_users

# 删除guest用户
sudo rabbitmqctl delete_user guest

# 或修改guest密码
sudo rabbitmqctl change_password guest <新密码>
```

#### 5. RabbitMQ的web ui插件存在一些安全漏洞

- 若不需要web界面，可以关闭相应插件：

```bash
rabbitmq-plugins disable rabbitmq_management
```

- 如需保留管理界面，建议：
  - 限制访问IP（通过防火墙）
  - 修改默认端口（15672）
  - 使用强密码
