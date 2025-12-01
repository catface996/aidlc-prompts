# Nginx 反向代理最佳实践

## 角色设定

你是一位精通 Nginx 的 Web 服务器专家，擅长反向代理、负载均衡、SSL配置和性能优化。

## 提示词模板

### Nginx 配置

```
请帮我配置 Nginx：
- 使用场景：[反向代理/负载均衡/静态资源/API网关]
- 后端服务：[服务列表和端口]
- SSL 需求：[是否需要 HTTPS]
- 负载策略：[轮询/权重/IP哈希/最少连接]
- 优化需求：[缓存/压缩/限流]

请提供完整的配置文件。
```

## 核心配置示例

### 主配置文件

```nginx
# nginx.conf
user nginx;
worker_processes auto;
worker_rlimit_nofile 65535;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 65535;
    use epoll;
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # 日志格式
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for" '
                    'rt=$request_time uct="$upstream_connect_time" '
                    'uht="$upstream_header_time" urt="$upstream_response_time"';

    log_format json escape=json '{'
        '"time":"$time_iso8601",'
        '"remote_addr":"$remote_addr",'
        '"request":"$request",'
        '"status":"$status",'
        '"body_bytes_sent":"$body_bytes_sent",'
        '"request_time":"$request_time",'
        '"upstream_response_time":"$upstream_response_time",'
        '"http_referer":"$http_referer",'
        '"http_user_agent":"$http_user_agent"'
    '}';

    access_log /var/log/nginx/access.log json;

    # 基础优化
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    # Gzip 压缩
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_min_length 1000;
    gzip_types text/plain text/css text/xml application/json application/javascript
               application/xml application/xml+rss text/javascript application/x-javascript;

    # 缓冲区设置
    client_body_buffer_size 16k;
    client_max_body_size 50m;
    client_header_buffer_size 1k;
    large_client_header_buffers 4 32k;

    # 代理缓冲区
    proxy_buffer_size 4k;
    proxy_buffers 8 32k;
    proxy_busy_buffers_size 64k;
    proxy_temp_file_write_size 64k;

    # 超时设置
    proxy_connect_timeout 60s;
    proxy_send_timeout 60s;
    proxy_read_timeout 60s;

    # 限流配置
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=100r/s;
    limit_conn_zone $binary_remote_addr zone=conn_limit:10m;

    # SSL 配置
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;

    # 引入其他配置
    include /etc/nginx/conf.d/*.conf;
}
```

### 负载均衡配置

```nginx
# conf.d/upstream.conf

# 轮询 (默认)
upstream order_service {
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
    server 10.0.0.3:8080;

    keepalive 32;
}

# 权重
upstream user_service {
    server 10.0.0.11:8080 weight=5;
    server 10.0.0.12:8080 weight=3;
    server 10.0.0.13:8080 weight=2 backup;

    keepalive 32;
}

# IP Hash
upstream cart_service {
    ip_hash;
    server 10.0.0.21:8080;
    server 10.0.0.22:8080;
    server 10.0.0.23:8080;
}

# 最少连接
upstream payment_service {
    least_conn;
    server 10.0.0.31:8080 max_fails=3 fail_timeout=30s;
    server 10.0.0.32:8080 max_fails=3 fail_timeout=30s;

    keepalive 32;
}

# 一致性哈希
upstream session_service {
    hash $request_uri consistent;
    server 10.0.0.41:8080;
    server 10.0.0.42:8080;
    server 10.0.0.43:8080;
}
```

### 反向代理配置

```nginx
# conf.d/api.conf
server {
    listen 80;
    server_name api.example.com;

    # 强制跳转 HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name api.example.com;

    # SSL 证书
    ssl_certificate /etc/nginx/ssl/api.example.com.crt;
    ssl_certificate_key /etc/nginx/ssl/api.example.com.key;

    # 安全头
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # 访问日志
    access_log /var/log/nginx/api.access.log json;
    error_log /var/log/nginx/api.error.log warn;

    # 健康检查
    location /health {
        access_log off;
        return 200 'healthy\n';
        add_header Content-Type text/plain;
    }

    # API 路由
    location /api/v1/orders {
        limit_req zone=api_limit burst=50 nodelay;
        limit_conn conn_limit 20;

        proxy_pass http://order_service;
        proxy_http_version 1.1;
        proxy_set_header Connection "";

        # 请求头传递
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Request-ID $request_id;

        # 超时设置
        proxy_connect_timeout 30s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;

        # 重试配置
        proxy_next_upstream error timeout http_500 http_502 http_503;
        proxy_next_upstream_tries 3;
        proxy_next_upstream_timeout 10s;
    }

    location /api/v1/users {
        proxy_pass http://user_service;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /api/v1/payments {
        # 更长的超时时间
        proxy_connect_timeout 60s;
        proxy_send_timeout 120s;
        proxy_read_timeout 120s;

        proxy_pass http://payment_service;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # WebSocket 支持
    location /ws {
        proxy_pass http://websocket_service;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }

    # 默认返回 404
    location / {
        return 404 '{"error": "Not Found"}';
        add_header Content-Type application/json;
    }
}
```

### 静态资源配置

```nginx
# conf.d/static.conf
server {
    listen 80;
    server_name static.example.com;

    root /var/www/static;
    index index.html;

    # 缓存控制
    location ~* \.(jpg|jpeg|png|gif|ico|svg|webp)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    location ~* \.(css|js)$ {
        expires 7d;
        add_header Cache-Control "public";
        access_log off;
    }

    location ~* \.(woff|woff2|ttf|eot)$ {
        expires 365d;
        add_header Cache-Control "public, immutable";
        add_header Access-Control-Allow-Origin *;
        access_log off;
    }

    # HTML 不缓存
    location ~* \.html$ {
        expires -1;
        add_header Cache-Control "no-store, no-cache, must-revalidate";
    }

    # 文件不存在时返回 index.html (SPA)
    location / {
        try_files $uri $uri/ /index.html;
    }

    # 禁止访问隐藏文件
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }
}
```

### 限流配置

```nginx
# conf.d/rate_limit.conf
http {
    # 按 IP 限流
    limit_req_zone $binary_remote_addr zone=ip_limit:10m rate=10r/s;

    # 按 API Key 限流
    map $http_x_api_key $api_key_limit {
        default         $binary_remote_addr;
        ~^.+$           $http_x_api_key;
    }
    limit_req_zone $api_key_limit zone=api_key_limit:10m rate=100r/s;

    # 按 URI 限流
    limit_req_zone $uri zone=uri_limit:10m rate=50r/s;

    server {
        location /api/login {
            # 登录接口更严格的限流
            limit_req zone=ip_limit burst=5 nodelay;
            limit_req_status 429;
            limit_req_log_level warn;

            proxy_pass http://auth_service;
        }

        location /api/ {
            # API 限流
            limit_req zone=api_key_limit burst=50 nodelay;
            limit_conn conn_limit 20;

            # 超出限制返回 429
            error_page 429 = @rate_limited;

            proxy_pass http://api_service;
        }

        location @rate_limited {
            return 429 '{"error": "Too Many Requests", "retry_after": 60}';
            add_header Content-Type application/json;
            add_header Retry-After 60;
        }
    }
}
```

### 缓存配置

```nginx
# conf.d/cache.conf
http {
    # 缓存路径配置
    proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=api_cache:100m
                     max_size=10g inactive=60m use_temp_path=off;

    server {
        location /api/products {
            # 启用缓存
            proxy_cache api_cache;
            proxy_cache_valid 200 10m;
            proxy_cache_valid 404 1m;
            proxy_cache_key "$scheme$request_method$host$request_uri";

            # 缓存头
            add_header X-Cache-Status $upstream_cache_status;

            # 缓存绕过条件
            proxy_cache_bypass $http_cache_control;
            proxy_no_cache $http_pragma;

            # 后端不可用时使用过期缓存
            proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
            proxy_cache_background_update on;
            proxy_cache_lock on;

            proxy_pass http://product_service;
        }

        # 缓存清除
        location ~ /purge(/.*) {
            allow 127.0.0.1;
            allow 10.0.0.0/8;
            deny all;
            proxy_cache_purge api_cache "$scheme$request_method$host$1";
        }
    }
}
```

### CORS 配置

```nginx
# conf.d/cors.conf
map $http_origin $cors_origin {
    default "";
    "~^https://.*\.example\.com$" $http_origin;
    "https://app.example.com" $http_origin;
}

server {
    location /api/ {
        # CORS 预检请求
        if ($request_method = 'OPTIONS') {
            add_header 'Access-Control-Allow-Origin' $cors_origin;
            add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS';
            add_header 'Access-Control-Allow-Headers' 'Authorization, Content-Type, X-Requested-With';
            add_header 'Access-Control-Allow-Credentials' 'true';
            add_header 'Access-Control-Max-Age' 86400;
            add_header 'Content-Type' 'text/plain charset=UTF-8';
            add_header 'Content-Length' 0;
            return 204;
        }

        # 实际请求
        add_header 'Access-Control-Allow-Origin' $cors_origin always;
        add_header 'Access-Control-Allow-Credentials' 'true' always;

        proxy_pass http://api_service;
    }
}
```

### Docker 部署

```dockerfile
# Dockerfile
FROM nginx:1.25-alpine

# 删除默认配置
RUN rm -rf /etc/nginx/conf.d/*

# 复制配置
COPY nginx.conf /etc/nginx/nginx.conf
COPY conf.d/ /etc/nginx/conf.d/
COPY ssl/ /etc/nginx/ssl/

# 创建缓存目录
RUN mkdir -p /var/cache/nginx

# 健康检查
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget -q --spider http://localhost/health || exit 1

EXPOSE 80 443

CMD ["nginx", "-g", "daemon off;"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  nginx:
    build: .
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./conf.d:/etc/nginx/conf.d:ro
      - ./ssl:/etc/nginx/ssl:ro
      - ./logs:/var/log/nginx
      - nginx-cache:/var/cache/nginx
    networks:
      - backend
    depends_on:
      - app
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '1'
          memory: 512M

volumes:
  nginx-cache:

networks:
  backend:
    driver: bridge
```

## 常用命令

```bash
# 测试配置
nginx -t

# 重新加载配置
nginx -s reload

# 查看状态
nginx -V

# 查看连接状态
curl http://localhost/nginx_status

# 日志分析
tail -f /var/log/nginx/access.log | jq .

# 清理缓存
rm -rf /var/cache/nginx/*
```

## 最佳实践清单

- [ ] 启用 Gzip 压缩
- [ ] 配置 SSL/TLS
- [ ] 设置安全响应头
- [ ] 配置访问限流
- [ ] 启用代理缓存
- [ ] 配置健康检查
- [ ] 日志格式标准化
- [ ] 超时时间合理设置
