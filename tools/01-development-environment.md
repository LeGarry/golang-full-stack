# 开发环境搭建

## 1. Go环境配置

### 1.1 安装Go
```bash
# Windows
# 1. 下载安装包：https://golang.org/dl/
# 2. 运行安装程序
# 3. 配置环境变量
setx GOPATH "%USERPROFILE%\go"
setx PATH "%PATH%;%USERPROFILE%\go\bin"

# Linux/MacOS
# 使用包管理器安装
# Ubuntu
sudo apt-get update
sudo apt-get install golang

# MacOS
brew install go

# 验证安装
go version
```

### 1.2 配置GOPATH
```bash
# GOPATH目录结构
GOPATH/
  ├── bin/    # 可执行文件
  ├── pkg/    # 编译后的包文件
  └── src/    # 源代码

# Go Modules配置
go env -w GO111MODULE=on
go env -w GOPROXY=https://goproxy.cn,direct

# 初始化项目
mkdir myproject
cd myproject
go mod init myproject
```

## 2. IDE配置

### 2.1 VSCode配置
```json
// settings.json
{
    "go.useLanguageServer": true,
    "go.toolsManagement.autoUpdate": true,
    "go.formatTool": "goimports",
    "editor.formatOnSave": true,
    "[go]": {
        "editor.defaultFormatter": "golang.go"
    }
}

// 安装必要插件
code --install-extension golang.go
code --install-extension ms-vscode.go
```

### 2.2 GoLand配置
```
# 推荐设置
1. 启用自动导入
File > Settings > Editor > General > Auto Import > Go
勾选 "Add unambiguous imports on the fly"

2. 配置代码格式化
File > Settings > Editor > Code Style > Go
选择 "Use tab character"
设置 "Tab size" 为 4

3. 配置文件监视
File > Settings > Appearance & Behavior > System Settings
增加 "File Monitoring" 限制
```

## 3. 数据库环境

### 3.1 MySQL安装
```bash
# Docker安装MySQL
docker run --name mysql \
    -e MYSQL_ROOT_PASSWORD=password \
    -p 3306:3306 \
    -d mysql:8.0

# 配置MySQL
docker exec -it mysql mysql -uroot -p

CREATE DATABASE myapp CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'myapp'@'%' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON myapp.* TO 'myapp'@'%';
FLUSH PRIVILEGES;
```

### 3.2 Redis安装
```bash
# Docker安装Redis
docker run --name redis \
    -p 6379:6379 \
    -d redis:6.2

# Redis配置
docker exec -it redis redis-cli

# 设置密码
CONFIG SET requirepass "your-password"
AUTH your-password

# 测试连接
PING
```

## 4. 前端环境

### 4.1 Node.js安装
```bash
# 使用nvm安装Node.js
# Windows
# 下载nvm-windows: https://github.com/coreybutler/nvm-windows/releases

# Linux/MacOS
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash

# 安装Node.js
nvm install 16
nvm use 16

# 验证安装
node --version
npm --version

# 配置npm镜像
npm config set registry https://registry.npmmirror.com
```

### 4.2 前端工具链
```bash
# 安装常用工具
npm install -g yarn
npm install -g @vue/cli
npm install -g create-react-app
npm install -g typescript
npm install -g eslint
npm install -g prettier

# 创建Vue项目
vue create my-vue-app

# 创建React项目
npx create-react-app my-react-app --template typescript
```

## 5. Docker环境

### 5.1 Docker安装
```bash
# Ubuntu安装Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# 配置用户组
sudo usermod -aG docker $USER
newgrp docker

# 启动Docker
sudo systemctl start docker
sudo systemctl enable docker

# 验证安装
docker --version
docker-compose --version
```

### 5.2 Docker Compose配置
```yaml
# docker-compose.yml
version: '3'

services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - DB_HOST=mysql
      - DB_USER=myapp
      - DB_PASSWORD=password
      - DB_NAME=myapp
      - REDIS_HOST=redis
      - REDIS_PASSWORD=password
    depends_on:
      - mysql
      - redis

  mysql:
    image: mysql:8.0
    environment:
      - MYSQL_ROOT_PASSWORD=password
      - MYSQL_DATABASE=myapp
      - MYSQL_USER=myapp
      - MYSQL_PASSWORD=password
    volumes:
      - mysql_data:/var/lib/mysql
    ports:
      - "3306:3306"

  redis:
    image: redis:6.2
    command: redis-server --requirepass password
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"

volumes:
  mysql_data:
  redis_data:
```

## 6. 版本控制

### 6.1 Git配置
```bash
# 安装Git
# Ubuntu
sudo apt-get install git

# MacOS
brew install git

# 基本配置
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# 生成SSH密钥
ssh-keygen -t ed25519 -C "your.email@example.com"

# 配置.gitignore
cat > .gitignore << EOF
# Go
/bin/
/pkg/
*.exe
*.exe~
*.dll
*.so
*.dylib
*.test
*.out

# IDE
.idea/
.vscode/
*.swp
*.swo

# 依赖
/vendor/
/node_modules/
yarn.lock
package-lock.json

# 环境文件
.env
.env.local
.env.*.local

# 日志
*.log
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# 系统文件
.DS_Store
Thumbs.db
EOF
```

### 6.2 Git工作流
```bash
# 创建新分支
git checkout -b feature/new-feature

# 提交代码
git add .
git commit -m "feat: add new feature"

# 合并主分支
git checkout main
git pull origin main
git merge feature/new-feature

# 解决冲突
git mergetool

# 推送更改
git push origin main
```

## 7. 监控工具

### 7.1 Prometheus配置
```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'myapp'
    static_configs:
      - targets: ['localhost:8080']
```

### 7.2 Grafana配置
```bash
# Docker运行Grafana
docker run -d \
  -p 3000:3000 \
  --name=grafana \
  -v grafana-storage:/var/lib/grafana \
  grafana/grafana

# 访问 http://localhost:3000
# 默认用户名/密码: admin/admin
``` 