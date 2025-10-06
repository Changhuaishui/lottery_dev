# CentOS7 红包抽奖系统中间件安装完整教程

## 📋 目录
- [项目概述](#项目概述)
- [环境信息](#环境信息)
- [安装前准备](#安装前准备)
- [MySQL 5.7.44 安装](#mysql-5744-安装)
- [Redis 6.0.20 安装](#redis-6020-安装)
- [RabbitMQ 3.6.10 安装](#rabbitmq-3610-安装)
- [MinIO 安装](#minio-安装)
- [最终验证](#最终验证)
- [常见问题与解决方案](#常见问题与解决方案)

## 项目概述

本文档记录了红包抽奖系统在CentOS7环境下安装配置所需中间件的完整过程。该系统是一个基于Spring Boot的分布式抽奖平台，需要以下中间件支撑：

- **MySQL 5.7.44**: 关系型数据库，存储业务数据
- **Redis 6.0.x**: 内存数据库，实现缓存和令牌管理
- **RabbitMQ 3.6.10**: 消息队列，处理异步消息
- **MinIO**: 对象存储，管理文件资源

## 环境信息

### 服务器配置
- **操作系统**: CentOS 7.x
- **服务器IP**: 192.168.64.100
- **用户**: root
- **密码**: root

### 目标端口配置
- **MySQL**: 3306 (用户名: root, 密码: root)
- **Redis**: 6379 (无密码)
- **RabbitMQ**: 5672 (用户名: guest, 密码: guest)
  - 管理界面: http://192.168.64.100:15672
- **MinIO**: 9005 (用户名: minioadmin, 密码: minioadmin)
  - 管理界面: http://192.168.64.100:9006

## 安装前准备

### 1. 系统基础配置

```bash
# 更新系统包（可选）
sudo yum update -y

# 安装基础工具
sudo yum install -y wget curl vim net-tools lsof

# 配置防火墙规则
sudo firewall-cmd --permanent --add-port=3306/tcp   # MySQL
sudo firewall-cmd --permanent --add-port=6379/tcp   # Redis  
sudo firewall-cmd --permanent --add-port=5672/tcp   # RabbitMQ AMQP
sudo firewall-cmd --permanent --add-port=15672/tcp  # RabbitMQ Management
sudo firewall-cmd --permanent --add-port=9005/tcp   # MinIO API
sudo firewall-cmd --permanent --add-port=9006/tcp   # MinIO Console
sudo firewall-cmd --reload

# 验证端口开放
sudo firewall-cmd --list-ports
```

### 2. 项目版本需求分析

通过分析项目的Docker配置文件(`deploy/docker-compose.yml`)，确定了精确的版本要求：

```yaml
# 项目实际使用的中间件版本
lottery-mysql: mysql:5.7.43
lottery-rabbitmq: rabbitmq:3.6.10-management  
lottery-redis: redis:6.0.6
lottery-minio: minio/minio:RELEASE.2023-12-02T10-51-33Z
```

## MySQL 5.7.44 安装

### 1. 添加MySQL官方源

```bash
# 下载MySQL 5.7的yum源配置
wget https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm

# 安装yum源
sudo rpm -Uvh mysql57-community-release-el7-11.noarch.rpm

# 导入MySQL GPG密钥
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql

# 验证yum源配置
yum repolist enabled | grep mysql
```

### 2. 安装MySQL服务器

```bash
# 安装MySQL 5.7服务器
sudo yum install -y mysql-community-server

# 启动MySQL服务
sudo systemctl start mysqld
sudo systemctl enable mysqld

# 检查服务状态
sudo systemctl status mysqld
```

### 3. 配置MySQL

```bash
# 获取MySQL初始临时密码
sudo grep 'temporary password' /var/log/mysqld.log
```

**实际获取的临时密码示例**: `3C%o5K(f4)+!`

**遇到的问题1**: 特殊字符密码输入困难
**解决方案**: 使用引号包围密码
```bash
# 方法一：使用引号包围密码
mysql -u root -p'3C%o5K(f4)+!'
```

**遇到的问题2**: 密码策略限制
**错误信息**: `Your password does not satisfy the current policy requirements`
**解决方案**: 
```bash
# 登录MySQL后执行以下命令：
SET GLOBAL validate_password_policy=LOW;
SET GLOBAL validate_password_length=1;
SET PASSWORD = PASSWORD('root');
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'root' WITH GRANT OPTION;
FLUSH PRIVILEGES;
SELECT user, host FROM mysql.user WHERE user = 'root';
EXIT;
```

### 4. 验证MySQL安装

```bash
# 测试连接
mysql -u root -proot -e "SELECT VERSION();"

# 导入项目数据库
mysql -u root -proot < prize_2024-01-03.sql

# 验证数据库导入
mysql -u root -proot -e "USE prize; SHOW TABLES;"
```

## Redis 6.0.20 安装

### 1. 安装Redis依赖源

```bash
# 安装EPEL源
sudo yum install -y epel-release

# 安装Remi源（提供Redis 6.x版本）
sudo yum install -y http://rpms.famillecollet.com/enterprise/remi-release-7.rpm

# 查看可用的Redis版本
yum --enablerepo=remi list redis --showduplicates
```

### 2. 安装Redis

**实际安装过程**: 由于Redis 6.0.6版本不可用，我们安装了兼容的6.0.20版本

```bash
# 查看可用版本的实际输出
yum --enablerepo=remi list redis --showduplicates
# 输出显示可用版本：
# redis.x86_64    6.0.19-1.el7.remi    remi
# redis.x86_64    6.0.20-1.el7.remi    remi
# redis.x86_64    6.2.13-1.el7.remi    remi
# redis.x86_64    6.2.14-1.el7.remi    remi

# 安装Redis 6.0.20（6.0.x系列的最新版本，与项目需求兼容）
sudo yum --enablerepo=remi install -y redis-6.0.20

# 验证Redis版本
redis-server --version
```

### 3. 配置Redis

```bash
# 备份原配置文件
sudo cp /etc/redis.conf /etc/redis.conf.backup

# 修改Redis配置文件
sudo vim /etc/redis.conf
```

需要修改的配置：
```bash
# 允许外部访问
bind 0.0.0.0

# 关闭保护模式
protected-mode no

# 设置为守护进程模式
daemonize yes

# 不设置密码（项目配置中没有密码）
# requirepass 保持注释状态
```

### 4. 启动Redis服务

```bash
# 启动Redis服务
sudo systemctl start redis
sudo systemctl enable redis

# 检查服务状态
sudo systemctl status redis

# 测试Redis连接
redis-cli ping
# 应该返回：PONG

# 测试远程连接
redis-cli -h 192.168.64.100 -p 6379 ping
```

## RabbitMQ 3.6.10 安装

### 1. 安装兼容的Erlang版本

```bash
# RabbitMQ 3.6.10需要Erlang 19.x版本
wget https://github.com/rabbitmq/erlang-rpm/releases/download/v19.3.6.13/erlang-19.3.6.13-1.el7.centos.x86_64.rpm

# 安装Erlang
sudo rpm -ivh erlang-19.3.6.13-1.el7.centos.x86_64.rpm

# 验证Erlang安装
erl -version
```

### 2. 安装RabbitMQ依赖

```bash
# 安装socat依赖
sudo yum install -y socat
```

### 3. 安装RabbitMQ

```bash
# 下载RabbitMQ 3.6.10
wget https://github.com/rabbitmq/rabbitmq-server/releases/download/rabbitmq_v3_6_10/rabbitmq-server-3.6.10-1.el7.noarch.rpm

# 安装RabbitMQ
sudo rpm -ivh rabbitmq-server-3.6.10-1.el7.noarch.rpm

# 启动RabbitMQ服务
sudo systemctl start rabbitmq-server
sudo systemctl enable rabbitmq-server

# 启用管理插件
sudo rabbitmq-plugins enable rabbitmq_management

# 检查服务状态
sudo systemctl status rabbitmq-server
```

### 4. 配置远程访问

**遇到的问题**: guest用户只能从localhost访问
**解决方案**:
```bash
# 创建RabbitMQ配置文件
sudo mkdir -p /etc/rabbitmq
sudo tee /etc/rabbitmq/rabbitmq.config > /dev/null <<'EOF'
[
  {rabbit, [
    {loopback_users, []}
  ]}
].
EOF

# 重启RabbitMQ服务
sudo systemctl restart rabbitmq-server

# 验证配置
curl -u guest:guest http://192.168.64.100:15672/api/overview
```

## MinIO 安装

### 1. 创建MinIO用户和目录

```bash
# 创建MinIO用户
sudo useradd -r minio-user -s /sbin/nologin

# 创建数据目录
sudo mkdir -p /opt/minio/data
sudo chown minio-user:minio-user /opt/minio/data
```

### 2. 下载MinIO二进制文件

**遇到的问题**: 指定版本的下载链接失效
**错误信息**: `404 Not Found` 当尝试下载 `minio.RELEASE.2023-12-02T10-51-33Z`

**解决方案**: 使用最新版本的MinIO
```bash
# 下载最新的MinIO二进制文件
wget https://dl.min.io/server/minio/release/linux-amd64/minio

# 给执行权限
chmod +x minio

# 移动到系统路径
sudo mv minio /usr/local/bin/

# 验证安装（实际安装的版本）
/usr/local/bin/minio --version
# 输出: minio version RELEASE.2025-09-07T16-13-09Z
```

**版本兼容性说明**: 虽然安装的是2025年版本而不是项目指定的2023年版本，但MinIO向后兼容，不会影响项目运行。

### 3. 创建systemd服务文件

```bash
sudo tee /etc/systemd/system/minio.service > /dev/null <<'EOF'
[Unit]
Description=MinIO
Documentation=https://docs.min.io
Wants=network-online.target
After=network-online.target
AssertFileIsExecutable=/usr/local/bin/minio

[Service]
WorkingDirectory=/opt/minio/data
User=minio-user
Group=minio-user
EnvironmentFile=-/etc/default/minio
ExecStart=/usr/local/bin/minio server $MINIO_OPTS $MINIO_VOLUMES
Restart=always
LimitNOFILE=65536
TimeoutStopSec=infinity
SendSIGKILL=no

[Install]
WantedBy=multi-user.target
EOF
```

### 4. 创建配置文件

```bash
sudo tee /etc/default/minio > /dev/null <<'EOF'
# MinIO数据目录
MINIO_VOLUMES="/opt/minio/data"

# MinIO端口配置（程序连9005，管理界面9006）
MINIO_OPTS="--address :9005 --console-address :9006"

# MinIO访问凭据（新版本环境变量）
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=minioadmin
EOF
```

### 5. 启动MinIO服务

```bash
# 重新加载systemd配置
sudo systemctl daemon-reload

# 启动MinIO服务
sudo systemctl start minio

# 设置开机自启
sudo systemctl enable minio

# 检查服务状态
sudo systemctl status minio
```

### 6. 创建项目存储桶

```bash
# 下载MinIO客户端
wget https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc
sudo mv mc /usr/local/bin/

# 配置MinIO客户端
mc alias set local http://192.168.64.100:9005 minioadmin minioadmin

# 创建项目需要的存储桶
mc mb local/prize

# 验证存储桶创建
mc ls local/
```

## 最终验证

### 实际验证结果

根据安装过程的实际反馈，所有中间件都已成功安装并运行：

**MySQL 5.7.44**: ✅ 成功安装，密码已设置为root，支持远程访问
**Redis 6.0.20**: ✅ 成功安装，配置为无密码访问，支持远程连接
**RabbitMQ 3.6.10**: ✅ 成功安装，guest用户可远程访问管理界面
**MinIO 2025-09-07**: ✅ 成功安装，管理界面正常运行

### 服务状态确认

从实际运行日志可以看到：

```bash
# MinIO启动日志显示：
# API: http://192.168.64.100:9005
# WebUI: http://192.168.64.100:9006
# 用户名/密码: minioadmin/minioadmin

# RabbitMQ状态显示：
# Status of node rabbit@localhost
# RabbitMQ Management Console 3.6.10 运行正常
# 监听端口: 5672 (AMQP), 15672 (HTTP管理界面)

# Redis测试结果：
# redis-cli ping 返回 PONG
# 远程连接测试成功

# MySQL连接测试：
# 版本: 5.7.44
# 远程连接配置完成
# 项目数据库导入成功
```

### 完整验证脚本

```bash
cat > /tmp/final_verify.sh << 'EOF'
#!/bin/bash
echo "=========================================="
echo "🎉 红包抽奖系统中间件安装完成验证"
echo "=========================================="

# MySQL验证
echo -n "MySQL (root/root@3306): "
mysql -h 192.168.64.100 -P 3306 -u root -proot -e "SELECT 'OK' as status;" 2>/dev/null | grep -q "OK" && echo "✅ 成功" || echo "❌ 失败"

# Redis验证
echo -n "Redis (无密码@6379): "
redis-cli -h 192.168.64.100 -p 6379 ping 2>/dev/null | grep -q "PONG" && echo "✅ 成功" || echo "❌ 失败"

# RabbitMQ验证
echo -n "RabbitMQ (guest/guest@5672): "
curl -s -u guest:guest http://192.168.64.100:15672/api/overview >/dev/null 2>&1 && echo "✅ 成功" || echo "❌ 失败"

# MinIO验证
echo -n "MinIO (minioadmin/minioadmin@9005/9006): "
curl -s http://192.168.64.100:9006 >/dev/null 2>&1 && echo "✅ 成功" || echo "❌ 失败"

echo "=========================================="
echo "📋 配置信息汇总："
echo "服务器IP: 192.168.64.100"
echo "MySQL: 端口3306, 用户名root, 密码root"
echo "Redis: 端口6379, 无密码"
echo "RabbitMQ: 端口5672, 用户名guest, 密码guest"
echo "  - 管理界面: http://192.168.64.100:15672"
echo "MinIO: 端口9005, 用户名minioadmin, 密码minioadmin"
echo "  - 管理界面: http://192.168.64.100:9006"
echo "=========================================="
echo "🚀 所有中间件安装完成！项目可以启动了！"
echo "=========================================="
EOF

chmod +x /tmp/final_verify.sh
/tmp/final_verify.sh
```

### 端口监听验证

```bash
# 检查所有端口监听状态
sudo netstat -tlnp | grep -E "(3306|6379|5672|15672|9005|9006)"
```

## 常见问题与解决方案

### 1. MySQL相关问题

#### 问题：GPG密钥验证失败
**错误信息**: `GPG 密钥已安装，但是不适用于此软件包`
**解决方案**:
```bash
# 导入所有相关的MySQL GPG密钥
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2022
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql
sudo yum clean all
sudo yum install -y mysql-community-server
```

#### 问题：密码策略限制
**错误信息**: `Your password does not satisfy the current policy requirements`
**解决方案**:
```bash
# 在MySQL中临时禁用密码策略
SET GLOBAL validate_password_policy=LOW;
SET GLOBAL validate_password_length=1;
SET PASSWORD = PASSWORD('root');
```

### 2. Redis相关问题

#### 问题：指定版本不存在
**解决方案**: 使用兼容版本
```bash
# 查看可用版本
yum --enablerepo=remi list redis --showduplicates
# 安装最接近的版本（如6.0.20代替6.0.6）
sudo yum --enablerepo=remi install -y redis-6.0.20
```

### 3. RabbitMQ相关问题

#### 问题：guest用户无法远程访问
**错误信息**: `User can only log in via localhost`
**解决方案**:
```bash
# 修改RabbitMQ配置允许guest远程访问
sudo tee /etc/rabbitmq/rabbitmq.config > /dev/null <<'EOF'
[
  {rabbit, [
    {loopback_users, []}
  ]}
].
EOF
sudo systemctl restart rabbitmq-server
```

### 4. MinIO相关问题

#### 问题：指定版本下载链接失效
**解决方案**: 使用最新版本
```bash
# 下载最新版本的MinIO
wget https://dl.min.io/server/minio/release/linux-amd64/minio
# 使用新版本的环境变量
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=minioadmin
```

#### 问题：服务启动失败
**错误信息**: `Assertion failed for MinIO`
**解决方案**: 确保二进制文件存在且有执行权限
```bash
# 检查文件是否存在
ls -la /usr/local/bin/minio
# 确保有执行权限
chmod +x /usr/local/bin/minio
```

## 总结

本次安装过程中的关键学习点：

1. **版本兼容性的重要性**: 必须严格按照项目需求选择中间件版本
2. **依赖关系管理**: 如RabbitMQ需要特定版本的Erlang支持
3. **配置文件的项目适配**: 所有配置都要符合项目的实际需求
4. **问题排查思路**: 从日志、端口、权限等多个角度分析问题
5. **安全配置**: 在开发环境中适当放宽安全限制，但要了解生产环境的安全要求

通过这次完整的安装过程，不仅搭建了项目运行环境，更重要的是掌握了Linux系统管理、中间件配置和问题排查的实践经验。

---

**文档创建时间**: 2025年9月27日  
**适用环境**: CentOS 7.x  
**项目**: 红包抽奖系统  
**状态**: 安装完成，验证通过 ✅