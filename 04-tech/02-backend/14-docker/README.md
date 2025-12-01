# Docker 容器化最佳实践

## 角色设定

你是一位精通 Docker 和容器化的 DevOps 专家，擅长镜像构建、容器编排和 CI/CD 集成。

## 提示词模板

### Dockerfile 编写

```
请帮我编写 Dockerfile：
- 应用类型：[Java/Node.js/Python/Go]
- 基础镜像：[官方/自定义]
- 构建方式：[单阶段/多阶段]
- 优化需求：
  - [ ] 镜像体积优化
  - [ ] 构建缓存利用
  - [ ] 安全加固
```

### Docker Compose

```
请帮我编写 Docker Compose 配置：
- 服务列表：[列出服务]
- 网络需求：[描述网络]
- 存储需求：[描述存储]
- 环境：[开发/测试/生产]
```

## 核心配置示例

### Java 应用 Dockerfile

```dockerfile
# 构建阶段
FROM eclipse-temurin:17-jdk-alpine AS builder

WORKDIR /app

# 复制 Maven 配置，利用缓存
COPY pom.xml .
COPY .mvn .mvn
COPY mvnw .
RUN chmod +x mvnw && ./mvnw dependency:go-offline -B

# 复制源码并构建
COPY src ./src
RUN ./mvnw package -DskipTests -B

# 运行阶段
FROM eclipse-temurin:17-jre-alpine

# 安全：创建非 root 用户
RUN addgroup -g 1000 app && \
    adduser -u 1000 -G app -s /bin/sh -D app

WORKDIR /app

# 复制构建产物
COPY --from=builder /app/target/*.jar app.jar

# 设置权限
RUN chown -R app:app /app
USER app

# 健康检查
HEALTHCHECK --interval=30s --timeout=3s --start-period=60s --retries=3 \
    CMD wget -q --spider http://localhost:8080/actuator/health || exit 1

# JVM 参数
ENV JAVA_OPTS="-Xms512m -Xmx512m -XX:+UseG1GC"

EXPOSE 8080

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

### Node.js 应用 Dockerfile

```dockerfile
# 构建阶段
FROM node:20-alpine AS builder

WORKDIR /app

# 复制依赖文件
COPY package*.json ./
RUN npm ci --only=production

# 复制源码并构建
COPY . .
RUN npm run build

# 运行阶段
FROM node:20-alpine

RUN addgroup -g 1000 app && \
    adduser -u 1000 -G app -s /bin/sh -D app

WORKDIR /app

# 只复制必要文件
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./

RUN chown -R app:app /app
USER app

HEALTHCHECK --interval=30s --timeout=3s \
    CMD wget -q --spider http://localhost:3000/health || exit 1

EXPOSE 3000

CMD ["node", "dist/main.js"]
```

### Docker Compose 开发环境

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: builder  # 开发时使用构建阶段
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=dev
      - DB_HOST=mysql
      - REDIS_HOST=redis
      - NACOS_SERVER=nacos:8848
    volumes:
      - ./src:/app/src  # 挂载源码，热重载
      - maven-cache:/root/.m2
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - backend

  mysql:
    image: mysql:8.0
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: app_db
      TZ: Asia/Shanghai
    volumes:
      - mysql-data:/var/lib/mysql
      - ./sql/init.sql:/docker-entrypoint-initdb.d/init.sql
    command: --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    command: redis-server --appendonly yes
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend

  nacos:
    image: nacos/nacos-server:v2.3.0
    ports:
      - "8848:8848"
      - "9848:9848"
    environment:
      - MODE=standalone
      - SPRING_DATASOURCE_PLATFORM=mysql
      - MYSQL_SERVICE_HOST=mysql
      - MYSQL_SERVICE_DB_NAME=nacos
      - MYSQL_SERVICE_USER=root
      - MYSQL_SERVICE_PASSWORD=root
    depends_on:
      mysql:
        condition: service_healthy
    networks:
      - backend

  elasticsearch:
    image: elasticsearch:8.11.0
    ports:
      - "9200:9200"
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    volumes:
      - es-data:/usr/share/elasticsearch/data
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:9200/_cluster/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5
    networks:
      - backend

  rocketmq-namesrv:
    image: apache/rocketmq:5.1.0
    ports:
      - "9876:9876"
    command: sh mqnamesrv
    networks:
      - backend

  rocketmq-broker:
    image: apache/rocketmq:5.1.0
    ports:
      - "10909:10909"
      - "10911:10911"
    environment:
      - NAMESRV_ADDR=rocketmq-namesrv:9876
    command: sh mqbroker -n rocketmq-namesrv:9876
    depends_on:
      - rocketmq-namesrv
    networks:
      - backend

volumes:
  mysql-data:
  redis-data:
  es-data:
  maven-cache:

networks:
  backend:
    driver: bridge
```

### Docker Compose 生产环境

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  app:
    image: ${REGISTRY}/app:${TAG:-latest}
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '2'
          memory: 2G
        reservations:
          cpus: '0.5'
          memory: 512M
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      update_config:
        parallelism: 1
        delay: 10s
        order: start-first
    environment:
      - SPRING_PROFILES_ACTIVE=prod
      - JAVA_OPTS=-Xms1g -Xmx1g -XX:+UseG1GC
    ports:
      - "8080:8080"
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "3"
    networks:
      - app-network

networks:
  app-network:
    external: true
```

### CI/CD Dockerfile

```dockerfile
# .docker/Dockerfile.ci
FROM eclipse-temurin:17-jdk-alpine AS builder

WORKDIR /app
COPY . .
RUN ./mvnw package -DskipTests -B

FROM eclipse-temurin:17-jre-alpine

ARG BUILD_DATE
ARG GIT_COMMIT
ARG VERSION

LABEL org.opencontainers.image.created="${BUILD_DATE}" \
      org.opencontainers.image.revision="${GIT_COMMIT}" \
      org.opencontainers.image.version="${VERSION}" \
      org.opencontainers.image.title="My App"

RUN addgroup -g 1000 app && adduser -u 1000 -G app -s /bin/sh -D app

WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
RUN chown -R app:app /app

USER app

EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s CMD wget -q --spider http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", "-jar", "app.jar"]
```

### GitHub Actions CI/CD

```yaml
# .github/workflows/docker.yml
name: Docker Build and Push

on:
  push:
    branches: [main]
    tags: ['v*']

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=sha

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            BUILD_DATE=${{ github.event.head_commit.timestamp }}
            GIT_COMMIT=${{ github.sha }}
            VERSION=${{ steps.meta.outputs.version }}
```

### 常用命令

```bash
# 构建镜像
docker build -t app:latest .

# 多阶段构建指定目标
docker build --target builder -t app:dev .

# 运行容器
docker run -d --name app -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=prod \
  -v /data/logs:/app/logs \
  app:latest

# Docker Compose 命令
docker-compose up -d              # 启动所有服务
docker-compose down               # 停止并删除
docker-compose logs -f app        # 查看日志
docker-compose exec app sh        # 进入容器
docker-compose ps                 # 查看状态

# 清理
docker system prune -a            # 清理所有未使用资源
docker volume prune               # 清理未使用卷
```

## 最佳实践清单

- [ ] 使用多阶段构建减小镜像体积
- [ ] 使用非 root 用户运行容器
- [ ] 配置健康检查
- [ ] 合理利用构建缓存
- [ ] 设置资源限制
- [ ] 配置日志收集
- [ ] 使用 .dockerignore 排除文件
- [ ] 镜像添加版本标签
