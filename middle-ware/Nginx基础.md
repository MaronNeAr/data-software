#### Nginx配置与命令

###### Nginx 配置文件基础

- **全局块（Main Context）**：配置影响全局的参数，如用户、进程数、日志路径等。

  ```nginx
  user  nginx;                # 运行Nginx的用户和组
  worker_processes  auto;     # 工作进程数（通常设为CPU核心数）
  error_log  /var/log/nginx/error.log warn;  	# 错误日志路径及级别
  pid        /var/run/nginx.pid;     					# 进程PID文件路径
  ```

- **Events 块**：配置网络连接相关参数。

  ```nginx
  events {
      worker_connections  1024;   # 单个工作进程的最大并发连接数
      use epoll;                  # 使用高效的事件模型（Linux）
  }
  ```

- **HTTP 块**：定义 HTTP 服务的全局配置，可包含多个 `server` 块（虚拟主机）。

  ```nginx
  http {
      include       /etc/nginx/mime.types;   	# 包含MIME类型定义文件
      default_type  application/octet-stream; # 默认响应类型
  
      # 日志格式
      log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                        '$status $body_bytes_sent "$http_referer" '
                        '"$http_user_agent" "$http_x_forwarded_for"';
  
      access_log  /var/log/nginx/access.log  main; # 访问日志路径
  
      sendfile        on;          # 启用高效文件传输模式
      keepalive_timeout  65;       # 客户端长连接超时时间
  
      # 包含其他配置文件（如虚拟主机配置）
      include /etc/nginx/conf.d/*.conf;
  }
  ```

- **Server 块（虚拟主机）**：定义单个网站的监听端口、域名、路由规则等。

  ```nginx
  server {
      listen       80;            # 监听端口
      server_name  example.com;   # 域名或IP
  
      location / {                # 路由匹配规则
          root   /usr/share/nginx/html;  # 静态资源根目录
          index  index.html index.htm;   # 默认首页
      }
  
      # 错误页面配置
      error_page  500 502 503 504  /50x.html;
      location = /50x.html {
          root   /usr/share/nginx/html;
      }
  }
  ```

###### 常用配置指令

- **静态资源服务**

  ```nginx
  location /static/ {
      alias /data/static/;    # 路径映射（末尾必须加/）
      expires 30d;            # 设置缓存过期时间
      access_log off;         # 关闭访问日志
  }
  ```

- **反向代理**

  ```nginx
  location /api/ {
      proxy_pass http://backend_server;  # 转发到后端服务器
      proxy_set_header Host $host;       # 传递原始请求头
      proxy_set_header X-Real-IP $remote_addr;
  }
  ```

- **负载均衡**

  ```nginx
  upstream backend_server {
      server 10.0.0.1:8080 weight=3;  # 权重分配
      server 10.0.0.2:8080;
      server 10.0.0.3:8080 backup;    # 备用服务器
  }
  
  server {
      location / {
          proxy_pass http://backend_server;
      }
  }
  ```

- **HTTPS 配置**

  ```nginx
  server {
      listen 443 ssl;
      server_name example.com;
  
      ssl_certificate      /etc/ssl/certs/example.com.crt;  # 证书路径
      ssl_certificate_key  /etc/ssl/private/example.com.key;
  
      ssl_session_cache    shared:SSL:10m;   	# SSL会话缓存
      ssl_session_timeout  10m;             	# 会话超时时间
  
      # 安全协议配置
      ssl_protocols TLSv1.2 TLSv1.3;
      ssl_ciphers HIGH:!aNULL:!MD5;
      ssl_prefer_server_ciphers on;
  
      location / {
          root   /usr/share/nginx/html;
          index  index.html;
      }
  }
  ```

- **URL重写**

  ```nginx
  location /old/ {
      rewrite ^/old/(.*)$ /new/$1 permanent;  # 永久重定向
  }
  
  # 正则匹配示例
  location ~ ^/user/(\d+) {
      rewrite ^/user/(\d+) /profile?id=$1 last;
  }
  ```

###### Nginx 常用命令

- **启动与停止**

```cmd
nginx                     # 启动Nginx
nginx -s stop             # 立即停止
nginx -s quit             # 优雅停止（处理完当前请求后退出）
```

- **重载配置**

```cmd
nginx -s reload           # 重新加载配置文件（不中断服务）
nginx -t                  # 测试配置文件语法是否正确
```

- **日志管理**

```cmd
# 切割日志（需配合cron定时任务）
mv /var/log/nginx/access.log /var/log/nginx/access_$(date +%Y%m%d).log
nginx -s reopen           # 重新打开日志文件
```

- **查看进程与版本**

```cmd
ps aux | grep nginx       # 查看Nginx进程
nginx -v                  # 查看版本
nginx -V                  # 查看编译参数及模块信息
```

