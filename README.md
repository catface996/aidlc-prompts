# AIDLC Prompts - AI 开发生命周期提示词库

> AI Development Lifecycle Prompts - 一套完整的 AI 辅助软件开发最佳实践提示词集合

---

## 项目简介

AIDLC Prompts 是一个结构化的提示词库，旨在通过 AI 协助提升软件开发全生命周期的效率和质量。本项目包含：

- **角色定义**：8 种专业角色的行为规范和最佳实践
- **开发阶段**：需求、设计、任务、测试四个核心阶段的指导文档
- **技术栈**：47+ 种前后端技术的最佳实践规范
- **测试体系**：9 种测试类型的完整指南

---

## 目录结构

```
aidlc-prompts/
├── .kiro/steering/           # AIDLC 智能引导系统配置
│   └── AIDLC-GUIDE.md        # AI 协作引导主配置
├── 01-business/              # 业务域模板（按需扩展）
├── 02-role/                  # 角色最佳实践 (8)
├── 03-dlc/                   # 开发生命周期阶段
│   ├── 01-requirements/      # 需求阶段
│   ├── 02-design/            # 设计阶段
│   ├── 03-task/              # 任务阶段
│   └── 04-testing/           # 测试阶段 (9 种测试类型)
├── 04-tech/                  # 技术栈最佳实践
│   ├── 01-frontend/          # 前端技术 (27)
│   └── 02-backend/           # 后端技术 (20)
├── 05-assist/                # 辅助文档
├── 99-how-to-prompt/         # 提示词工程指南
└── 效果/                      # 效果展示
```

---

## 角色定义 (02-role)

| 编号 | 角色 | 文档链接 |
|------|------|----------|
| 01 | 产品经理 | [README.md](./02-role/01-product-manager/README.md) |
| 02 | 需求分析师 | [README.md](./02-role/02-requirement-analyst/README.md) |
| 03 | 架构师 | [README.md](./02-role/03-architect/README.md) |
| 04 | 项目经理 | [README.md](./02-role/04-project-manager/README.md) |
| 05 | 前端工程师 | [README.md](./02-role/05-frontend-engineer/README.md) |
| 06 | 后端工程师 | [README.md](./02-role/06-backend-engineer/README.md) |
| 07 | 测试工程师 | [README.md](./02-role/07-test-engineer/README.md) |
| 08 | DevOps 工程师 | [README.md](./02-role/08-devops-engineer/README.md) |

---

## 开发生命周期 (03-dlc)

### 需求阶段

| 文档 | 链接 |
|------|------|
| 需求最佳实践 (中文) | [01-requirements-best-practices.zh.md](./03-dlc/01-requirements/01-requirements-best-practices.zh.md) |
| 需求最佳实践 (英文) | [01-requirements-best-practices.en.md](./03-dlc/01-requirements/01-requirements-best-practices.en.md) |

### 设计阶段

| 文档 | 链接 |
|------|------|
| 设计最佳实践 (中文) | [02-design-best-practices.zh.md](./03-dlc/02-design/02-design-best-practices.zh.md) |
| 设计最佳实践 (英文) | [02-design-best-practices.en.md](./03-dlc/02-design/02-design-best-practices.en.md) |

### 任务阶段

| 文档 | 链接 |
|------|------|
| 任务规划最佳实践 (中文) | [03-tasks-planning-best-practices.zh.md](./03-dlc/03-task/03-tasks-planning-best-practices.zh.md) |
| 任务规划最佳实践 (英文) | [03-tasks-planning-best-practices.en.md](./03-dlc/03-task/03-tasks-planning-best-practices.en.md) |
| 任务执行最佳实践 (中文) | [04-tasks-execution-best-practices.zh.md](./03-dlc/03-task/04-tasks-execution-best-practices.zh.md) |
| 任务执行最佳实践 (英文) | [04-tasks-execution-best-practices.en.md](./03-dlc/03-task/04-tasks-execution-best-practices.en.md) |

### 测试阶段

| 编号 | 测试类型 | 链接 |
|------|----------|------|
| - | 测试总览 | [README.md](./03-dlc/04-testing/README.md) |
| 00 | 需求静态测试 | [README.md](./03-dlc/04-testing/00-static-review/README.md) |
| 01 | 单元测试 | [README.md](./03-dlc/04-testing/01-unit/README.md) |
| 02 | 集成测试 | [README.md](./03-dlc/04-testing/02-integration/README.md) |
| 03 | E2E 测试 | [README.md](./03-dlc/04-testing/03-e2e/README.md) |
| 04 | API 测试 | [README.md](./03-dlc/04-testing/04-api/README.md) |
| 05 | 性能测试 | [README.md](./03-dlc/04-testing/05-performance/README.md) |
| 06 | 安全测试 | [README.md](./03-dlc/04-testing/06-security/README.md) |
| 07 | 兼容性测试 | [README.md](./03-dlc/04-testing/07-compatibility/README.md) |
| 08 | 回归测试 | [README.md](./03-dlc/04-testing/08-regression/README.md) |

---

## 技术栈最佳实践 (04-tech)

### 前端技术 (27 项)

| 分类 | 技术 | 链接 |
|------|------|------|
| 基础 | HTML | [README.md](./04-tech/01-frontend/01-html/README.md) |
| 基础 | CSS | [README.md](./04-tech/01-frontend/02-css/README.md) |
| 基础 | JavaScript | [README.md](./04-tech/01-frontend/03-javascript/README.md) |
| 基础 | TypeScript | [README.md](./04-tech/01-frontend/04-typescript/README.md) |
| 框架 | React | [README.md](./04-tech/01-frontend/05-react/README.md) |
| 框架 | Vue | [README.md](./04-tech/01-frontend/06-vue/README.md) |
| 框架 | Angular | [README.md](./04-tech/01-frontend/07-angular/README.md) |
| 框架 | Svelte | [README.md](./04-tech/01-frontend/08-svelte/README.md) |
| 元框架 | Next.js | [README.md](./04-tech/01-frontend/09-nextjs/README.md) |
| 元框架 | Nuxt.js | [README.md](./04-tech/01-frontend/10-nuxtjs/README.md) |
| 构建工具 | Webpack | [README.md](./04-tech/01-frontend/11-webpack/README.md) |
| 构建工具 | Vite | [README.md](./04-tech/01-frontend/12-vite/README.md) |
| 工程化 | ESLint | [README.md](./04-tech/01-frontend/13-eslint/README.md) |
| 工程化 | Husky | [README.md](./04-tech/01-frontend/14-husky/README.md) |
| 网络通信 | Axios | [README.md](./04-tech/01-frontend/15-axios/README.md) |
| 网络通信 | WebSocket | [README.md](./04-tech/01-frontend/16-websocket/README.md) |
| 状态管理 | Zustand | [README.md](./04-tech/01-frontend/17-zustand/README.md) |
| 路由 | React Router | [README.md](./04-tech/01-frontend/18-react-router/README.md) |
| 认证 | JWT | [README.md](./04-tech/01-frontend/19-jwt/README.md) |
| 性能 | Caching | [README.md](./04-tech/01-frontend/20-caching/README.md) |
| 性能 | Performance | [README.md](./04-tech/01-frontend/21-performance/README.md) |
| 测试 | Testing | [README.md](./04-tech/01-frontend/22-testing/README.md) |
| 测试 | Cypress | [README.md](./04-tech/01-frontend/23-cypress/README.md) |
| 样式 | Tailwind CSS | [README.md](./04-tech/01-frontend/24-tailwindcss/README.md) |
| 动画 | Animation | [README.md](./04-tech/01-frontend/25-animation/README.md) |
| 图形 | Canvas | [README.md](./04-tech/01-frontend/26-canvas/README.md) |
| 国际化 | i18n | [README.md](./04-tech/01-frontend/27-i18n/README.md) |

### 后端技术 (20 项)

| 分类 | 技术 | 链接 |
|------|------|------|
| 语言 | Java | [README.md](./04-tech/02-backend/01-java/README.md) |
| 框架 | Spring Boot | [README.md](./04-tech/02-backend/02-spring-boot/README.md) |
| 框架 | Spring Cloud | [README.md](./04-tech/02-backend/03-spring-cloud/README.md) |
| ORM | MyBatis | [README.md](./04-tech/02-backend/04-mybatis/README.md) |
| 数据库 | MySQL | [README.md](./04-tech/02-backend/05-mysql/README.md) |
| 缓存 | Redis | [README.md](./04-tech/02-backend/06-redis/README.md) |
| 消息队列 | RocketMQ | [README.md](./04-tech/02-backend/07-rocketmq/README.md) |
| 消息队列 | Kafka | [README.md](./04-tech/02-backend/08-kafka/README.md) |
| 搜索 | Elasticsearch | [README.md](./04-tech/02-backend/09-elasticsearch/README.md) |
| 微服务 | Nacos | [README.md](./04-tech/02-backend/10-nacos/README.md) |
| 微服务 | Sentinel | [README.md](./04-tech/02-backend/11-sentinel/README.md) |
| 微服务 | Seata | [README.md](./04-tech/02-backend/12-seata/README.md) |
| 可观测性 | SkyWalking | [README.md](./04-tech/02-backend/13-skywalking/README.md) |
| 容器化 | Docker | [README.md](./04-tech/02-backend/14-docker/README.md) |
| 容器化 | Kubernetes | [README.md](./04-tech/02-backend/15-kubernetes/README.md) |
| 网关/代理 | Nginx | [README.md](./04-tech/02-backend/16-nginx/README.md) |
| 网关 | Gateway | [README.md](./04-tech/02-backend/17-gateway/README.md) |
| 安全 | Security | [README.md](./04-tech/02-backend/18-security/README.md) |
| 可观测性 | Logging | [README.md](./04-tech/02-backend/19-logging/README.md) |
| 可观测性 | Monitoring | [README.md](./04-tech/02-backend/20-monitoring/README.md) |

---

## 辅助文档 (05-assist)

| 文档 | 说明 | 链接 |
|------|------|------|
| AI辅助需求分析流程与实施指南 | AI 辅助需求分析完整流程 | [查看](./05-assist/AI辅助需求分析流程与实施指南.md) |
| AIDLC-C4-Architecture | AIDLC C4 架构文档 | [查看](./05-assist/AIDLC-C4-Architecture.md) |
| AIDLC_C4_full | AIDLC C4 完整版本 | [查看](./05-assist/AIDLC_C4_full.md) |
| codebase-analysis-prompt | 代码库分析提示词 | [查看](./05-assist/codebase-analysis-prompt.md) |
| how-to-test | 测试方法论指南 | [查看](./05-assist/how-to-test.md) |

---

## 提示词工程指南 (99-how-to-prompt)

| 文档 | 说明 | 链接 |
|------|------|------|
| OpenSpec提示词总结 | OpenSpec 提示词最佳实践 | [查看](./99-how-to-prompt/OpenSpec提示词总结.md) |
| anthropic-prompt-engineering-summary | Anthropic 提示词工程总结 | [查看](./99-how-to-prompt/anthropic-prompt-engineering-summary.md) |
| Spec-Kit-优秀提示词案例全集 | 优秀提示词案例集合 | [查看](./99-how-to-prompt/Spec-Kit-优秀提示词案例全集.md) |
| prompt-scoring-framework | 提示词评分框架 (PQEF) | [查看](./99-how-to-prompt/prompt-scoring-framework.md) |

---

## 智能引导系统

本项目配置了 AIDLC 智能引导系统 ([AIDLC-GUIDE.md](./.kiro/steering/AIDLC-GUIDE.md))，支持：

1. **角色识别**：自动推荐合适的 AI 角色
2. **阶段判断**：确定当前开发阶段
3. **技术栈匹配**：识别并加载相关技术文档
4. **文档自动加载**：根据上下文加载最佳实践文档

### 快速启动

```
/start [角色] [阶段] [技术栈]

示例：
/start 后端工程师 设计 "Java,Spring Boot,MySQL"
/start 需求分析师 需求 -
/start 架构师 设计 "微服务,K8s"
```

---

## 提示词设计规范

本项目的最佳实践文档遵循统一的结构规范：

| 章节 | 说明 |
|------|------|
| 角色设定 | AI 扮演的专家角色 |
| 触发词映射 | 用户表达 → 动作 → 输出物 |
| NON-NEGOTIABLE 规则 | MUST/NEVER/STRICTLY 强制规则 |
| 核心原则 | 该领域的核心概念和定位 |
| 具体规则 | 详细的执行规则和检查清单 |
| 执行步骤 | 分步骤的执行流程 |
| Gate Check 验证清单 | 完成前必须确认的检查点 |
| 输出格式模板 | 标准化的输出格式 |
| 提示词模板 | 可直接使用的提示词 |
| 最佳实践清单 | 快速参考的检查列表 |

---

## 使用方式

### 方式一：直接引用

在与 AI 对话时，引用相关的最佳实践文档：

```
请参考 03-dlc/04-testing/01-unit/README.md 帮我编写单元测试
```

### 方式二：智能引导

启动 AIDLC 智能引导系统，通过交互确定角色和阶段：

```
我要开发一个电商订单系统
```

系统将自动识别并推荐合适的角色、阶段和技术栈文档。

### 方式三：快速命令

使用快速启动命令直接进入协作模式：

```
/start 后端工程师 任务 "Spring Boot,MySQL,Redis"
```

---

## 贡献指南

1. Fork 本仓库
2. 创建特性分支 (`git checkout -b feature/new-tech-stack`)
3. 提交更改 (`git commit -m 'Add new tech stack best practices'`)
4. 推送分支 (`git push origin feature/new-tech-stack`)
5. 创建 Pull Request

---

## 许可证

MIT License

---

*AIDLC Prompts v1.0 - 让 AI 成为你的最佳开发伙伴*
