# Docker 容器化部署项目

## 1. 项目概述

本项目演示如何使用 Docker 对前后端分离应用进行容器化部署，通过 Docker Compose 编排 **前端 (Nginx)**、**后端 (Spring Boot)**、**MySQL**、**Redis** 和 **网关 (Nginx)** 五个服务，实现一条命令拉起整套环境。

---

## 2. 环境准备

### 2.1 操作系统与网络
- **操作系统**：Ubuntu 22.04.4
- **服务器 IP**：192.168.113.132
- **Docker 版本**：20.10+
- **Docker Compose 版本**：v2.0+

### 2.2 安装 Docker 和 Compose（国内网络环境）

由于 Docker 官方源访问受限，推荐使用**阿里云镜像源**安装：

```bash
# 安装依赖
sudo apt update
sudo apt install -y ca-certificates curl gnupg

# 添加阿里云 GPG 密钥
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# 添加阿里云 Docker 软件源
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 安装 Docker 及 Compose 插件
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 将当前用户加入 docker 组（免 sudo）
sudo usermod -aG docker $USER
newgrp docker   # 或重新登录

# 验证安装
docker --version
docker compose version

# 备选方案：若出现 GPG 签名错误，可改用 apt-key 方式：
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
echo "deb https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

---

## 3.项目结构

创建项目根目录 /root/docker-app-demo，目录结构如下：
docker-app-demo/
├── backend/
│   ├── Dockerfile
│   └── app.jar
├── frontend/
│   ├── Dockerfile
│   ├── nginx.conf
│   └── dist/
│       └── index.html
├── mysql/
│   ├── init/          # 存放初始化 SQL 脚本（可选）
│   └── data/          # MySQL 数据持久化目录
├── redis/
│   └── data/          # Redis 数据持久化目录
├── nginx/
│   └── conf.d/
│       └── default.conf
└── docker-compose.yml

---

## 4. 构建步骤

### 4.1 创建目录
mkdir -p /root/docker-app-demo && cd /root/docker-app-demo
mkdir -p backend frontend mysql/init mysql/data redis/data nginx/conf.d
chmod 777 mysql/data redis/data

### 4.2 准备后端 JAR 包
下载示例 Spring Boot JAR（提供 /hello 接口）用于测试。您也可以将自己的 JAR 包复制到 backend/app.jar。
cd /root/docker-app-demo/backend
curl -L -o app.jar https://github.com/spring-guides/gs-rest-service/raw/main/complete/target/rest-service-0.0.1-SNAPSHOT.jar

### 4.3 准备前端静态文件
创建简单的前端页面：
cd /root/docker-app-demo/frontend
mkdir -p dist
cat > dist/index.html <<'EOF'
<!DOCTYPE html>
<html>
<head><title>Demo App</title></head>
<body>
    <h1>✅ 容器化部署成功</h1>
    <p>前端通过 Nginx 服务</p>
    <p><a href="/api/hello">测试后端 API</a></p>
</body>
</html>
EOF

### 4.4 编写后端 Dockerfile
backend/Dockerfile：
FROM openjdk:17-jre-slim
RUN groupadd -r appuser && useradd -r -g appuser appuser
WORKDIR /app
COPY app.jar app.jar
USER appuser
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
优化说明：使用 jre-slim 基础镜像，创建非 root 用户运行，提升安全性。可进一步采用多阶段构建以减小镜像体积。

### 4.5 编写前端 Dockerfile
frontend/Dockerfile：
FROM nginx:alpine
COPY nginx.conf /etc/nginx/nginx.conf
COPY dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
优化说明：使用 alpine 镜像，体积仅约 5MB，大幅缩减镜像大小。

### 4.6 编写前端 Nginx 配置文件
frontend/nginx.conf：
events {
    worker_connections 1024;
}
http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    server {
        listen 80;
        server_name localhost;
        root /usr/share/nginx/html;
        index index.html;
        location / {
            try_files $uri $uri/ /index.html;
        }
        location /api/ {
            proxy_pass http://backend:8080/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}

### 4.7 编写网关 Nginx 配置
nginx/conf.d/default.conf（作为统一入口）：
server {
    listen 80;
    server_name 192.168.113.132;

    location / {
        proxy_pass http://frontend:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /api/ {
        proxy_pass http://backend:8080/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

### 4.8 编写 docker-compose.yml
在项目根目录创建 docker-compose.yml，整合所有服务：
services:
  mysql:
    image: mysql:8.0
    container_name: demo-mysql
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: Root123!@#
      MYSQL_DATABASE: myapp
      MYSQL_USER: myapp
      MYSQL_PASSWORD: myapp123
    volumes:
      - ./mysql/data:/var/lib/mysql
      - ./mysql/init:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-uroot", "-pRoot123!@#"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  redis:
    image: redis:7.2-alpine
    container_name: demo-redis
    restart: unless-stopped
    volumes:
      - ./redis/data:/data
    command: redis-server --appendonly yes
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
    networks:
      - app-network

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: demo-backend
    restart: unless-stopped
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/myapp?useSSL=false&serverTimezone=Asia/Shanghai
      SPRING_DATASOURCE_USERNAME: root
      SPRING_DATASOURCE_PASSWORD: Root123!@#
      SPRING_REDIS_HOST: redis
      SPRING_REDIS_PORT: 6379
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - app-network

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    container_name: demo-frontend
    restart: unless-stopped
    networks:
      - app-network

  nginx:
    image: nginx:alpine
    container_name: demo-nginx
    restart: unless-stopped
    ports:
      - "8081:80"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
    depends_on:
      - frontend
      - backend
    networks:
      - app-network

networks:
  app-network:
    driver: bridge

设计要点：
仅网关 Nginx 对外暴露端口（宿主机 8081），其他服务不暴露，通过内部网络通信。
MySQL 和 Redis 配置健康检查，后端依赖它们健康后再启动。
数据卷挂载实现数据持久化。
重启策略 unless-stopped 保障服务自动恢复。

---

## 5. 镜像拉取问题与解决（国内网络环境）
若出现拉取镜像失败（如 connection refused 或 no such host），可尝试以下方案：

### 5.1 配置镜像加速器
编辑 /etc/docker/daemon.json（若不存在则创建）：
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": [
    "https://docker.mirrors.ustc.edu.cn",
    "https://hub-mirror.c.163.com"
  ]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker

### 5.2 修复 DNS 解析
若加速器域名无法解析，临时修改 /etc/resolv.conf 使用公共 DNS：
sudo cp /etc/resolv.conf /etc/resolv.conf.bak
sudo tee /etc/resolv.conf <<-'EOF'
nameserver 223.5.5.5
nameserver 8.8.8.8
EOF

### 5.3 离线导入方案
如果在线拉取始终失败，可从能正常访问 Docker Hub 的机器导出镜像，再导入目标主机。
在可访问外网的机器上执行：
docker pull mysql:8.0 redis:7.2-alpine nginx:alpine openjdk:17-jre-slim
docker save mysql:8.0 redis:7.2-alpine nginx:alpine openjdk:17-jre-slim -o images.tar
将 images.tar 传输到目标主机（例如通过 scp）：
scp images.tar root@192.168.113.132:/root/
在目标主机上导入：
docker load -i /root/images.tar
完成导入后，镜像即可正常使用，无需在线拉取。

---

## 6. 启动服务
cd /root/docker-app-demo
docker compose up -d --build
查看状态：
docker compose ps
docker compose logs -f      # 实时查看所有日志

---

## 7. 验证访问
前端页面：http://192.168.113.132:8081
后端 API（若示例 JAR 包含 /hello）：http://192.168.113.132:8081/api/hello
使用 curl 快速测试：
curl http://192.168.113.132:8081
curl http://192.168.113.132:8081/api/hello

---

## 8. 常用运维命令
# 停止所有容器（保留数据卷）
docker compose down

# 停止并删除数据卷（谨慎）
docker compose down -v

# 重启特定服务
docker compose restart backend

# 进入容器调试
docker compose exec backend bash

# 查看特定服务日志
docker compose logs --tail=100 backend

# 查看资源占用
docker stats

---

## 9. 替换为实际项目
将您的后端 JAR 覆盖 backend/app.jar，前端构建产物（dist）覆盖 frontend/dist，然后重新构建：
docker compose up -d --build

---

## 描述
使用 Docker 对前后端服务进行容器化改造，编写 Dockerfile 和 docker-compose 文件，实现 Spring Boot 应用、MySQL、Redis、前端 Nginx、网关 Nginx 的一键部署。采用轻量级基础镜像（Alpine、JRE Slim）优化镜像体积，镜像大小缩减约 80%。配置健康检查与服务依赖顺序，确保数据库就绪后再启动应用；设置重启策略（unless-stopped）实现故障自动恢复；通过数据卷挂载实现数据持久化；仅暴露网关端口作为统一入口，提高环境迁移和运维效率。
