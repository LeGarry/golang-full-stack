# Golang全栈开发笔记

## 项目介绍

这是一个关于Golang全栈开发的综合学习笔记，涵盖了后端开发、前端开发、数据库、工具使用和项目实践等多个方面。本笔记致力于提供完整的Golang全栈开发知识体系，帮助开发者快速掌握Golang开发技能。

## 技术栈

### 后端技术
- Go 1.16+
- Gin/Echo/Fiber框架
- GORM
- JWT/OAuth2
- WebSocket
- gRPC

### 数据库
- MySQL 5.7+
- PostgreSQL 10+
- Redis
- MongoDB

### 前端技术
- React/Vue
- TypeScript
- Ant Design/Element UI
- Redux/Vuex

### 开发工具
- Docker & Docker Compose
- Kubernetes
- Git
- VSCode/GoLand

## 目录结构

```
golang-full-stack/
├── backend/                # 后端开发
│   ├── 01-golang-basics.md      # Golang基础知识
│   ├── 02-web-frameworks.md     # Web框架使用
│   ├── 03-database.md          # 数据库操作
│   ├── 04-middleware.md        # 中间件开发
│   ├── 05-auth.md             # 认证和授权
│   ├── 06-api-design.md       # API设计
│   ├── 07-testing.md          # 测试
│   ├── 08-deployment.md       # 部署
│   ├── 09-microservices.md    # 微服务架构
│   ├── 10-cloud-native.md     # 云原生开发
│   ├── 11-devops.md          # DevOps实践
│   ├── 12-security.md        # 安全最佳实践
│   └── 13-performance.md     # 高级性能优化
├── frontend/               # 前端开发
│   ├── 01-frontend-frameworks.md # 前端框架
│   ├── 02-state-management.md   # 状态管理
│   ├── 03-routing.md           # 路由管理
│   └── 04-api-integration.md   # API集成
├── database/              # 数据库
│   ├── 01-database-design.md   # 数据库设计
│   ├── 02-sql-basics.md        # SQL基础
│   └── 03-orm-usage.md         # ORM使用
├── tools/                 # 工具
│   ├── 01-development-environment.md # 开发环境
│   ├── 02-debugging.md         # 调试工具
│   └── 03-performance.md       # 性能工具
└── projects/              # 项目
    ├── 01-project-structure.md # 项目结构
    ├── 02-best-practices.md    # 最佳实践
    └── 03-common-issues.md     # 常见问题
```

## 学习路径

### 1. 基础阶段 - [开始学习](./backend/01-golang-basics.md)
- Go语言基础语法
- 并发编程基础
- 标准库使用
- 错误处理
- 包管理

### 2. Web开发 - [开始学习](./backend/02-web-frameworks.md)
- HTTP基础
- Gin框架使用
- 中间件开发
- 数据库操作
- API设计

### 3. 进阶主题 - [开始学习](./backend/09-microservices.md)
- 微服务架构
- 分布式系统
- 性能优化
- 安全实践
- 云原生开发

### 4. 前端开发 - [开始学习](./frontend/01-frontend-frameworks.md)
- React/Vue基础
- 状态管理
- 路由系统
- API集成

### 5. 工程实践 - [开始学习](./projects/01-project-structure.md)
- 项目结构设计
- 开发最佳实践
- 测试策略
- CI/CD流程

## 快速开始

1. 克隆仓库
```bash
git clone https://github.com/LeGarry/golang-full-stack.git
cd golang-full-stack
```

2. 选择学习路径
- 完全新手：从[Golang基础](./backend/01-golang-basics.md)开始
- 有Go基础：从[Web框架](./backend/02-web-frameworks.md)开始
- 后端开发者：从[微服务](./backend/09-microservices.md)开始
- 前端开发者：从[前端框架](./frontend/01-frontend-frameworks.md)开始

## 最佳实践

- [项目结构最佳实践](./projects/01-project-structure.md)
- [开发最佳实践](./projects/02-best-practices.md)
- [测试最佳实践](./backend/07-testing.md)
- [部署最佳实践](./backend/08-deployment.md)
- [安全最佳实践](./backend/12-security.md)
- [性能优化最佳实践](./backend/13-performance.md)

## 常见问题

- [性能优化问题](./projects/03-common-issues.md#performance)
- [部署问题](./projects/03-common-issues.md#deployment)
- [安全问题](./projects/03-common-issues.md#security)
- [开发环境问题](./projects/03-common-issues.md#development)

## 贡献指南

1. Fork 本仓库
2. 创建特性分支 (`git checkout -b feature/AmazingFeature`)
3. 提交更改 (`git commit -m 'Add some AmazingFeature'`)
4. 推送到分支 (`git push origin feature/AmazingFeature`)
5. 提交Pull Request

## 版本历史

- v1.0.0 (2024-01) - 初始版本发布
- v1.1.0 (2024-02) - 添加微服务和云原生内容
- v1.2.0 (2024-03) - 增加性能优化和安全实践

## 参考资源

- [Go官方文档](https://golang.org/doc/)
- [Gin框架文档](https://gin-gonic.com/docs/)
- [GORM文档](https://gorm.io/docs/)
- [Kubernetes文档](https://kubernetes.io/docs/)
- [Docker文档](https://docs.docker.com/)

## 许可证

本项目采用 MIT 许可证 - 查看 [LICENSE](LICENSE) 文件了解详情

## 联系方式

- 项目维护者: [legary]
- Email: [legary@foxmail.com]
- GitHub: [https://github.com/LeGarry]

## 致谢

感谢所有为这个项目做出贡献的开发者！
