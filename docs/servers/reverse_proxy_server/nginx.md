# Nginx 安装与反向代理配置完整指南

## 一、环境准备

### 1.1 系统环境

```bash
# 查看系统版本
cat /etc/centos-release
# 或
cat /etc/os-release

# 系统要求
# - CentOS 7/8/9 或 Ubuntu 18.04+
# - root 权限或 sudo 权限
# - 开放 80/443 端口
```



### 1.2 更新系统

```bash
# CentOS/RHEL
yum update -y

# Ubuntu/Debian
apt update && apt upgrade -y
```




## 二、安装 Nginx

### 2.1 CentOS/RHEL 系统

```bash
# 安装 EPEL 仓库（CentOS 7 需要）
yum install epel-release -y

# 安装 Nginx
yum install nginx -y

# 查看安装版本
nginx -v
```



**⚠️ 踩坑注意**：

- CentOS 7 默认仓库没有 Nginx，必须先安装 EPEL
- 如果想装最新版，需要添加官方源：

```bash
# 添加官方源（CentOS 7）
vim /etc/yum.repos.d/nginx.repo
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/7/$basearch/
gpgcheck=0
enabled=1

yum install nginx -y
```



### 2.2 Ubuntu/Debian 系统

```bash
# 安装 Nginx
apt install nginx -y

# 查看版本
nginx -v
```



### 2.3 启动与开机自启

```bash
# 启动 Nginx
systemctl start nginx

# 设置开机自启
systemctl enable nginx

# 检查状态
systemctl status nginx

# 防火墙开放端口（重要！）
# CentOS 使用 firewalld
firewall-cmd --permanent --add-port=80/tcp
firewall-cmd --permanent --add-port=443/tcp
firewall-cmd --reload

# Ubuntu 使用 ufw
ufw allow 80/tcp
ufw allow 443/tcp
ufw reload

# 或直接使用 iptables
iptables -I INPUT -p tcp --dport 80 -j ACCEPT
iptables -I INPUT -p tcp --dport 443 -j ACCEPT
service iptables save
```



**⚠️ 踩坑注意**：

- **云服务器**（阿里云/腾讯云等）除了系统防火墙，**还需要在控制台安全组中开放端口**！
- 检查端口是否真正开放：

```bash
netstat -tlnp | grep nginx
# 应该看到 0.0.0.0:80 和 0.0.0.0:443
```




## 三、Nginx 基础配置

### 3.1 配置文件结构

```bash
# CentOS/RHEL 结构（你的就是这种）
/etc/nginx/
├── nginx.conf              # 主配置文件
├── conf.d/                 # 自定义配置目录（推荐）
│   └── *.conf              # 所有 .conf 文件都会被加载
├── default.d/              # 备用目录
├── fastcgi_params          # FastCGI 参数
├── mime.types              # MIME 类型
└── ...

# Ubuntu/Debian 结构
/etc/nginx/
├── nginx.conf
├── sites-available/        # 可用站点配置
├── sites-enabled/          # 启用站点（软链接）
└── conf.d/
```



**⚠️ 踩坑注意**：

- 配置文件名**必须以 `.conf` 结尾**，否则不会被加载
- CentOS 默认 `conf.d/` 目录为空，需要自己创建配置
- 不要直接修改 `nginx.conf.default` 等 `.default` 文件

### 3.2 修改主配置（可选）

```bash
vim /etc/nginx/nginx.conf
```



```nginx
user nginx;                    # 运行用户
worker_processes auto;          # 自动匹配 CPU 核心数
error_log /var/log/nginx/error.log warn;
pid /run/nginx.pid;

events {
    worker_connections 1024;    # 每个进程最大连接数
    multi_accept on;
    use epoll;                  # Linux 高性能事件模型
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    # 日志格式
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';
    
    access_log /var/log/nginx/access.log main;
    
    # 性能优化
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    client_max_body_size 100M;   # ⚠️ 上传文件大小限制，按需调整
    
    # Gzip 压缩
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css text/xml text/javascript 
               application/javascript application/json;
    
    # 加载自定义配置
    include /etc/nginx/conf.d/*.conf;
}
```



**⚠️ 踩坑注意**：

- `client_max_body_size` 默认只有 1M，上传大文件必须修改
- 如果放在 `http` 块是全局生效，放在 `server` 或 `location` 块是局部生效


## 四、反向代理配置

### 4.1 单服务反向代理

在 `/etc/nginx/conf.d/` 下创建配置文件：

```bash
vim /etc/nginx/conf.d/proxy.conf
```



```nginx
server {
    listen 80;
    server_name your-domain.com;    # ⚠️ 改为你的域名或 IP
    
    # 访问日志
    access_log /var/log/nginx/proxy_access.log;
    error_log /var/log/nginx/proxy_error.log;
    
    location / {
        # 转发到后端服务
        proxy_pass http://127.0.0.1:3000;    # ⚠️ 改为你的后端地址
        
        # ⚠️ 必须设置这些头信息
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Connection "";
        
        # 超时设置
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```



### 4.2 多服务按路径分发

```bash
vim /etc/nginx/conf.d/multi-proxy.conf
```



```nginx
server {
    listen 80;
    server_name example.com;
    
    # API 服务
    location /api/ {
        proxy_pass http://127.0.0.1:3000/;    # ⚠️ 注意末尾斜杠
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
    
    # 管理后台
    location /admin/ {
        proxy_pass http://127.0.0.1:8000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
    
    # 静态文件（直接由 Nginx 提供）
    location /static/ {
        alias /var/www/static/;    # ⚠️ 物理路径
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
    
    # 默认
    location / {
        proxy_pass http://127.0.0.1:5000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```



**⚠️ 路径斜杠的坑**：

```nginx
# ❌ 错误：缺少斜杠会拼接路径
proxy_pass http://127.0.0.1:3000/api;
# 请求 /api/user → 转发到 /api/api/user

# ✅ 正确：末尾加斜杠
proxy_pass http://127.0.0.1:3000/api/;
# 请求 /api/user → 转发到 /api/user

# ✅ 正确：使用变量
proxy_pass http://127.0.0.1:3000;
# 请求 /api/user → 转发到 /api/user（完整 URI 保留）
```



### 4.3 负载均衡配置

```bash
vim /etc/nginx/conf.d/load-balance.conf
```



```nginx
upstream backend_servers {
    # 负载均衡算法
    # 默认轮询 (round-robin)
    # least_conn: 最少连接
    # ip_hash: IP 哈希（保持会话）
    # random: 随机
    
    least_conn;
    
    server 192.168.1.10:8080 weight=3;     # weight 权重
    server 192.168.1.11:8080;
    server 192.168.1.12:8080 backup;       # 备用服务器
    server 192.168.1.13:8080 max_fails=3 fail_timeout=30s;
    
    keepalive 32;    # 保持连接数
}

server {
    listen 80;
    server_name app.example.com;
    
    location / {
        proxy_pass http://backend_servers;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # ⚠️ 超时设置要合理
        proxy_connect_timeout 30s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        
        # 错误重试
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503;
        proxy_next_upstream_tries 3;
    }
}
```



### 4.4 WebSocket 反向代理

```bash
vim /etc/nginx/conf.d/websocket.conf
```



```nginx
upstream ws_backend {
    server 127.0.0.1:3001;
    keepalive 64;
}

server {
    listen 80;
    server_name ws.example.com;
    
    location /ws/ {
        proxy_pass http://ws_backend;
        proxy_http_version 1.1;
        
        # ⚠️ WebSocket 必须设置 Upgrade 头
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # ⚠️ WebSocket 长连接超时要设置大一些
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }
}
```



### 4.5 HTTPS 反向代理

```bash
vim /etc/nginx/conf.d/https-proxy.conf
```



```nginx
# HTTP 重定向到 HTTPS
server {
    listen 80;
    server_name secure.example.com;
    return 301 https://$server_name$request_uri;
}

# HTTPS 服务
server {
    listen 443 ssl http2;
    server_name secure.example.com;
    
    # ⚠️ SSL 证书路径
    ssl_certificate /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;
    
    # ⚠️ SSL 安全配置
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    
    # 安全响应头
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    
    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;    # ⚠️ 告诉后端是 HTTPS
    }
}
```




## 五、配置文件测试与生效

### 5.1 测试配置语法

```bash
# ⚠️ 每次修改配置后必须测试！
nginx -t

# 显示详细测试信息
nginx -t -c /etc/nginx/nginx.conf

# 查看所有加载的配置
nginx -T
```



**⚠️ 常见测试错误**：

```bash
# 错误1：缺少分号
nginx: [emerg] unexpected "}" in /etc/nginx/conf.d/proxy.conf:15

# 错误2：重复的 server 块
nginx: [emerg] duplicate "server" in /etc/nginx/conf.d/02-extra.conf:1

# 错误3：证书文件不存在
nginx: [emerg] BIO_new_file("/etc/nginx/ssl/fullchain.pem") failed

# 错误4：端口被占用
nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)
```



### 5.2 重载配置

```bash
# 优雅重载（推荐）
nginx -s reload

# 或使用 systemctl
systemctl reload nginx

# 完全重启（会短暂中断服务）
systemctl restart nginx

# 停止
nginx -s stop
# 或
systemctl stop nginx
```



### 5.3 查看日志

```bash
# 访问日志
tail -f /var/log/nginx/access.log

# 错误日志
tail -f /var/log/nginx/error.log

# 自定义日志
tail -f /var/log/nginx/proxy_access.log
tail -f /var/log/nginx/proxy_error.log

# 实时查看所有日志
tail -f /var/log/nginx/*.log
```




## 六、验证代理是否生效

### 6.1 本地测试

```bash
# 测试代理是否转发
curl -v http://your-domain.com/api/test

# 查看响应头
curl -I http://your-domain.com

# 测试 WebSocket
curl -H "Connection: Upgrade" -H "Upgrade: websocket" \
     -H "Sec-WebSocket-Key: x3JJHMbDL1EzLkh9GBhXDw==" \
     -H "Sec-WebSocket-Version: 13" \
     http://ws.example.com/ws/
```



### 6.2 查看转发状态

```bash
# 检查 Nginx 连接状态（需要开启 stub_status）
curl http://localhost/nginx_status

# 查看连接数
netstat -an | grep :80 | wc -l

# 查看后端服务是否正常
curl http://127.0.0.1:3000/health
```



### 6.3 调试工具

```bash
# 查看请求头
curl -v http://your-domain.com 2>&1 | grep -i "host\|x-forwarded"

# 使用 telnet 测试
telnet your-domain.com 80
GET / HTTP/1.1
Host: your-domain.com

# 使用 nc
echo -e "GET / HTTP/1.1\nHost: your-domain.com\n\n" | nc your-domain.com 80
```




## 七、常见踩坑点汇总

### 🔴 配置前必查

| 问题         | 检查命令                     | 解决方案                        |
| :----------- | :--------------------------- | :------------------------------ |
| 端口是否开放 | `netstat -tlnp | grep nginx` | 开放 80/443，云服务器检查安全组 |
| 配置文件语法 | `nginx -t`                   | 修正语法错误                    |
| 文件权限     | `ls -la /etc/nginx/conf.d/`  | 确保配置文件可读                |
| SELinux      | `getenforce`                 | 临时关闭: `setenforce 0`        |
| 后端服务     | `curl http://127.0.0.1:3000` | 确保后端服务正常运行            |

### 🔴 代理转发的坑

```nginx
# ❌ 错误：忘记设置 Host 头，后端可能无法正确识别域名
proxy_pass http://127.0.0.1:3000;
# ✅ 正确
proxy_pass http://127.0.0.1:3000;
proxy_set_header Host $host;

# ❌ 错误：后端拿不到真实 IP
proxy_set_header X-Real-IP $remote_addr;        # 要添加

# ❌ 错误：HTTPS 转发给 HTTP 后端时，后端不知道是 HTTPS
proxy_set_header X-Forwarded-Proto $scheme;     # 要添加

# ❌ 错误：路径斜杠问题
proxy_pass http://127.0.0.1:3000/api;           # 会拼接
proxy_pass http://127.0.0.1:3000/api/;          # 替换路径
```



### 🔴 性能配置的坑

```nginx
# ❌ 错误：连接数设置太小
worker_connections 1024;  # 建议 1024-65536

# ❌ 错误：没有开启 keepalive，每个请求都建立新连接
proxy_http_version 1.1;
proxy_set_header Connection "";

# ❌ 错误：超时时间太短，大文件/长连接被中断
proxy_read_timeout 60s;   # 根据业务调整
```



### 🔴 负载均衡的坑

```nginx
# ❌ 错误：没有配置健康检查，故障节点仍接收请求
# ✅ 使用 max_fails 和 fail_timeout
server 192.168.1.10:8080 max_fails=3 fail_timeout=30s;

# ❌ 错误：没有设置 backup，所有节点都不可用时无备用
server 192.168.1.11:8080 backup;

# ❌ 错误：没有配置 keepalive，连接开销大
keepalive 32;
```



### 🔴 WebSocket 的坑

```nginx
# ❌ 错误：忘记设置 Upgrade 头
proxy_set_header Upgrade $http_upgrade;      # 必须有
proxy_set_header Connection "upgrade";       # 必须有

# ❌ 错误：超时时间太小，WebSocket 连接被中断
proxy_read_timeout 3600s;   # 至少要 300s+
```



### 🔴 HTTPS 的坑

```nginx
# ❌ 错误：证书文件不存在或权限不足
ssl_certificate /etc/nginx/ssl/fullchain.pem;    # 检查路径
# chmod 644 fullchain.pem, chmod 600 privkey.pem

# ❌ 错误：只配置了 443，没有 80 重定向
server { listen 80; return 301 https://$server_name$request_uri; }

# ❌ 错误：没有配置 HSTS，容易被降级攻击
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```




## 八、完整配置文件模板

### 8.1 通用代理配置文件模板

```bash
vim /etc/nginx/conf.d/proxy-template.conf
```



```nginx
# ==========================================
# 通用反向代理配置模板
# ==========================================

upstream backend {
    server 127.0.0.1:3000;
    keepalive 32;
}

server {
    # 监听端口
    listen 80;
    listen [::]:80;
    server_name your-domain.com;
    
    # 日志
    access_log /var/log/nginx/proxy_access.log;
    error_log /var/log/nginx/proxy_error.log warn;
    
    # 上传大小限制
    client_max_body_size 50M;
    
    # 根目录（如果需要提供静态文件）
    root /var/www/html;
    index index.html index.htm;
    
    # 主要代理转发
    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port $server_port;
        
        # 超时配置
        proxy_connect_timeout 30s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        
        # 缓冲配置
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
        proxy_busy_buffers_size 8k;
    }
    
    # 静态文件缓存
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2|ttf|eot)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
        access_log off;
    }
    
    # 禁止访问隐藏文件
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }
}
```



### 8.2 通用 SSL 配置片段

```bash
vim /etc/nginx/conf.d/ssl-common.conf
```



```nginx
# ==========================================
# SSL 通用配置（被其他配置 include）
# ==========================================

ssl_certificate /etc/nginx/ssl/fullchain.pem;
ssl_certificate_key /etc/nginx/ssl/privkey.pem;

ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
ssl_prefer_server_ciphers off;

ssl_session_cache shared:SSL:10m;
ssl_session_timeout 10m;
ssl_session_tickets off;

# OCSP Stapling
ssl_stapling on;
ssl_stapling_verify on;
resolver 8.8.8.8 8.8.4.4 valid=300s;
resolver_timeout 5s;

# 安全头
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-XSS-Protection "1; mode=block" always;
```




## 九、快速排查命令

```bash
# 1. 测试配置
nginx -t

# 2. 重载配置
nginx -s reload

# 3. 查看错误日志
tail -20 /var/log/nginx/error.log

# 4. 检查端口监听
netstat -tlnp | grep -E ":(80|443)"

# 5. 检查后端服务
curl -I http://127.0.0.1:3000

# 6. 测试代理
curl -H "Host: your-domain.com" http://127.0.0.1/api/test

# 7. 查看进程
ps aux | grep nginx

# 8. 查看配置加载
nginx -T | grep -E "server_name|proxy_pass"
```




## 十、检查清单

配置完成后，逐项确认：

- [ ] Nginx 已安装并启动

- [ ] 80/443 端口已开放（系统防火墙 + 云平台安全组）

- [ ] 配置文件语法正确（`nginx -t` 通过）

- [ ] 配置文件已重载（`nginx -s reload`）

- [ ] 后端服务正常运行

- [ ] 代理转发路径正确（检查斜杠）

- [ ] 请求头设置完整（Host, X-Real-IP, X-Forwarded-For）

- [ ] 日志能正常输出

- [ ] 静态文件能正常访问（如有）

- [ ] HTTPS 证书有效（如使用）

- [ ] WebSocket 能正常连接（如有）