# Docker中间件操作指南

## 环境信息
- 服务器IP：192.168.64.100
- 项目目录：/opt/lottery/deploy

## 基础命令

### 1. 查看所有容器状态
```bash
docker ps
```

### 2. 启动所有服务
```bash
cd /opt/lottery/deploy
docker compose up -d
```

### 3. 停止所有服务
```bash
cd /opt/lottery/deploy
docker compose down
```

### 4. 重启单个服务
```bash
# 重启MySQL
docker restart lottery-mysql

# 重启Redis
docker restart lottery-redis

# 重启RabbitMQ
docker restart lottery-rabbitmq

# 重启MinIO
docker restart lottery-minio
```

### 5. 查看服务日志
```bash
# MySQL日志
docker logs lottery-mysql

# Redis日志
docker logs lottery-redis

# RabbitMQ日志
docker logs lottery-rabbitmq

# MinIO日志
docker logs lottery-minio
```

## 服务访问信息

### 1. MySQL
- 端口：3306
- 用户名：root
- 密码：root
- 连接测试：
```bash
mysql -h 127.0.0.1 -P 3306 -u root -proot -e "SELECT VERSION();"
```

### 2. Redis
- 端口：6379
- 无密码
- 连接测试：
```bash
redis-cli -h 127.0.0.1 -p 6379 ping
```

### 3. RabbitMQ
- 服务端口：5672
- 管理界面：http://192.168.64.100:15672
- 用户名：guest
- 密码：guest
- 连接测试：
```bash
curl -u guest:guest http://192.168.64.100:15672/api/overview
```

### 4. MinIO
- API端口：9005
- 管理界面：http://192.168.64.100:9006
- 用户名：minioadmin
- 密码：minioadmin
- 注意：建议使用Chrome浏览器访问

### 5. Portainer（容器管理）
- 管理界面：http://192.168.64.100:9000
- 用户名：admin
- 密码：admin

## 常见操作

### 1. 服务器重启后恢复服务
```bash
cd /opt/lottery/deploy
docker compose up -d
```

### 2. 查看容器详细信息
```bash
docker inspect lottery-mysql
docker inspect lottery-redis
docker inspect lottery-rabbitmq
docker inspect lottery-minio
```

### 3. 进入容器内部
```bash
# 进入MySQL容器
docker exec -it lottery-mysql bash

# 进入Redis容器
docker exec -it lottery-redis sh

# 进入RabbitMQ容器
docker exec -it lottery-rabbitmq bash

# 进入MinIO容器
docker exec -it lottery-minio sh
```

## 注意事项
1. 修改配置后需要重启对应的容器
2. 数据持久化目录在 /opt/lottery/deploy 下
3. 建议定期备份数据目录
4. MinIO管理界面推荐使用Chrome浏览器访问