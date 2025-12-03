# AIDLC Prompts - AI 开发生命周期提示词库

> AI Development Lifecycle Prompts - 一套完整的 AI 辅助软件开发最佳实践提示词集合

![AIDLC Prompt 架构图](100-result/aidlc-prompt.png)

---

## 项目简介

AIDLC Prompts 是一个结构化的提示词库，旨在通过 AI 协助提升软件开发全生命周期的效率和质量。本项目包含：

- **角色定义**：8 种专业角色的行为规范和最佳实践
- **开发阶段**：需求、设计、任务、测试四个核心阶段的指导文档
- **技术栈**：47+ 种前后端技术的最佳实践规范
- **测试体系**：9 种测试类型的完整指南

所有文档均提供中英文双语版本（.zh.md / .en.md）。

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

| 编号 | 角色 | 中文 | English |
|------|------|------|---------|
| 01 | 产品经理 | [README.zh.md](./02-role/01-product-manager/README.zh.md) | [README.en.md](./02-role/01-product-manager/README.en.md) |
| 02 | 需求分析师 | [README.zh.md](./02-role/02-requirement-analyst/README.zh.md) | [README.en.md](./02-role/02-requirement-analyst/README.en.md) |
| 03 | 架构师 | [README.zh.md](./02-role/03-architect/README.zh.md) | [README.en.md](./02-role/03-architect/README.en.md) |
| 04 | 项目经理 | [README.zh.md](./02-role/04-project-manager/README.zh.md) | [README.en.md](./02-role/04-project-manager/README.en.md) |
| 05 | 前端工程师 | [README.zh.md](./02-role/05-frontend-engineer/README.zh.md) | [README.en.md](./02-role/05-frontend-engineer/README.en.md) |
| 06 | 后端工程师 | [README.zh.md](./02-role/06-backend-engineer/README.zh.md) | [README.en.md](./02-role/06-backend-engineer/README.en.md) |
| 07 | 测试工程师 | [README.zh.md](./02-role/07-test-engineer/README.zh.md) | [README.en.md](./02-role/07-test-engineer/README.en.md) |
| 08 | DevOps 工程师 | [README.zh.md](./02-role/08-devops-engineer/README.zh.md) | [README.en.md](./02-role/08-devops-engineer/README.en.md) |

---

## 开发生命周期 (03-dlc)

### 需求阶段

| 文档 | 中文 | English |
|------|------|---------|
| 需求最佳实践 | [.zh.md](./03-dlc/01-requirements/01-requirements-best-practices.zh.md) | [.en.md](./03-dlc/01-requirements/01-requirements-best-practices.en.md) |

### 设计阶段

| 文档 | 中文 | English |
|------|------|---------|
| 设计最佳实践 | [.zh.md](./03-dlc/02-design/02-design-best-practices.zh.md) | [.en.md](./03-dlc/02-design/02-design-best-practices.en.md) |

### 任务阶段

| 文档 | 中文 | English |
|------|------|---------|
| 任务规划最佳实践 | [.zh.md](./03-dlc/03-task/03-tasks-planning-best-practices.zh.md) | [.en.md](./03-dlc/03-task/03-tasks-planning-best-practices.en.md) |
| 任务执行最佳实践 | [.zh.md](./03-dlc/03-task/04-tasks-execution-best-practices.zh.md) | [.en.md](./03-dlc/03-task/04-tasks-execution-best-practices.en.md) |

### 测试阶段

| 编号 | 测试类型 | 中文 | English |
|------|----------|------|---------|
| - | 测试总览 | [README.zh.md](./03-dlc/04-testing/README.zh.md) | [README.en.md](./03-dlc/04-testing/README.en.md) |
| 00 | 需求静态测试 | [README.zh.md](./03-dlc/04-testing/00-static-review/README.zh.md) | [README.en.md](./03-dlc/04-testing/00-static-review/README.en.md) |
| 01 | 单元测试 | [README.zh.md](./03-dlc/04-testing/01-unit/README.zh.md) | [README.en.md](./03-dlc/04-testing/01-unit/README.en.md) |
| 02 | 集成测试 | [README.zh.md](./03-dlc/04-testing/02-integration/README.zh.md) | [README.en.md](./03-dlc/04-testing/02-integration/README.en.md) |
| 03 | E2E 测试 | [README.zh.md](./03-dlc/04-testing/03-e2e/README.zh.md) | [README.en.md](./03-dlc/04-testing/03-e2e/README.en.md) |
| 04 | API 测试 | [README.zh.md](./03-dlc/04-testing/04-api/README.zh.md) | [README.en.md](./03-dlc/04-testing/04-api/README.en.md) |
| 05 | 性能测试 | [README.zh.md](./03-dlc/04-testing/05-performance/README.zh.md) | [README.en.md](./03-dlc/04-testing/05-performance/README.en.md) |
| 06 | 安全测试 | [README.zh.md](./03-dlc/04-testing/06-security/README.zh.md) | [README.en.md](./03-dlc/04-testing/06-security/README.en.md) |
| 07 | 兼容性测试 | [README.zh.md](./03-dlc/04-testing/07-compatibility/README.zh.md) | [README.en.md](./03-dlc/04-testing/07-compatibility/README.en.md) |
| 08 | 回归测试 | [README.zh.md](./03-dlc/04-testing/08-regression/README.zh.md) | [README.en.md](./03-dlc/04-testing/08-regression/README.en.md) |

---

## 技术栈最佳实践 (04-tech)

### 前端技术 (27 项)

| 分类 | 技术 | 中文 | English |
|------|------|------|---------|
| 基础 | HTML | [README.zh.md](./04-tech/01-frontend/01-html/README.zh.md) | [README.en.md](./04-tech/01-frontend/01-html/README.en.md) |
| 基础 | CSS | [README.zh.md](./04-tech/01-frontend/02-css/README.zh.md) | [README.en.md](./04-tech/01-frontend/02-css/README.en.md) |
| 基础 | JavaScript | [README.zh.md](./04-tech/01-frontend/03-javascript/README.zh.md) | [README.en.md](./04-tech/01-frontend/03-javascript/README.en.md) |
| 基础 | TypeScript | [README.zh.md](./04-tech/01-frontend/04-typescript/README.zh.md) | [README.en.md](./04-tech/01-frontend/04-typescript/README.en.md) |
| 框架 | React | [README.zh.md](./04-tech/01-frontend/05-react/README.zh.md) | [README.en.md](./04-tech/01-frontend/05-react/README.en.md) |
| 框架 | Vue | [README.zh.md](./04-tech/01-frontend/06-vue/README.zh.md) | [README.en.md](./04-tech/01-frontend/06-vue/README.en.md) |
| 框架 | Angular | [README.zh.md](./04-tech/01-frontend/07-angular/README.zh.md) | [README.en.md](./04-tech/01-frontend/07-angular/README.en.md) |
| 框架 | Svelte | [README.zh.md](./04-tech/01-frontend/08-svelte/README.zh.md) | [README.en.md](./04-tech/01-frontend/08-svelte/README.en.md) |
| 元框架 | Next.js | [README.zh.md](./04-tech/01-frontend/09-nextjs/README.zh.md) | [README.en.md](./04-tech/01-frontend/09-nextjs/README.en.md) |
| 元框架 | Nuxt.js | [README.zh.md](./04-tech/01-frontend/10-nuxtjs/README.zh.md) | [README.en.md](./04-tech/01-frontend/10-nuxtjs/README.en.md) |
| 构建工具 | Webpack | [README.zh.md](./04-tech/01-frontend/11-webpack/README.zh.md) | [README.en.md](./04-tech/01-frontend/11-webpack/README.en.md) |
| 构建工具 | Vite | [README.zh.md](./04-tech/01-frontend/12-vite/README.zh.md) | [README.en.md](./04-tech/01-frontend/12-vite/README.en.md) |
| 工程化 | ESLint | [README.zh.md](./04-tech/01-frontend/13-eslint/README.zh.md) | [README.en.md](./04-tech/01-frontend/13-eslint/README.en.md) |
| 工程化 | Husky | [README.zh.md](./04-tech/01-frontend/14-husky/README.zh.md) | [README.en.md](./04-tech/01-frontend/14-husky/README.en.md) |
| 网络通信 | Axios | [README.zh.md](./04-tech/01-frontend/15-axios/README.zh.md) | [README.en.md](./04-tech/01-frontend/15-axios/README.en.md) |
| 网络通信 | WebSocket | [README.zh.md](./04-tech/01-frontend/16-websocket/README.zh.md) | [README.en.md](./04-tech/01-frontend/16-websocket/README.en.md) |
| 状态管理 | Zustand | [README.zh.md](./04-tech/01-frontend/17-zustand/README.zh.md) | [README.en.md](./04-tech/01-frontend/17-zustand/README.en.md) |
| 路由 | React Router | [README.zh.md](./04-tech/01-frontend/18-react-router/README.zh.md) | [README.en.md](./04-tech/01-frontend/18-react-router/README.en.md) |
| 认证 | JWT | [README.zh.md](./04-tech/01-frontend/19-jwt/README.zh.md) | [README.en.md](./04-tech/01-frontend/19-jwt/README.en.md) |
| 性能 | Caching | [README.zh.md](./04-tech/01-frontend/20-caching/README.zh.md) | [README.en.md](./04-tech/01-frontend/20-caching/README.en.md) |
| 性能 | Performance | [README.zh.md](./04-tech/01-frontend/21-performance/README.zh.md) | [README.en.md](./04-tech/01-frontend/21-performance/README.en.md) |
| 测试 | Testing | [README.zh.md](./04-tech/01-frontend/22-testing/README.zh.md) | [README.en.md](./04-tech/01-frontend/22-testing/README.en.md) |
| 测试 | Cypress | [README.zh.md](./04-tech/01-frontend/23-cypress/README.zh.md) | [README.en.md](./04-tech/01-frontend/23-cypress/README.en.md) |
| 样式 | Tailwind CSS | [README.zh.md](./04-tech/01-frontend/24-tailwindcss/README.zh.md) | [README.en.md](./04-tech/01-frontend/24-tailwindcss/README.en.md) |
| 动画 | Animation | [README.zh.md](./04-tech/01-frontend/25-animation/README.zh.md) | [README.en.md](./04-tech/01-frontend/25-animation/README.en.md) |
| 图形 | Canvas | [README.zh.md](./04-tech/01-frontend/26-canvas/README.zh.md) | [README.en.md](./04-tech/01-frontend/26-canvas/README.en.md) |
| 国际化 | i18n | [README.zh.md](./04-tech/01-frontend/27-i18n/README.zh.md) | [README.en.md](./04-tech/01-frontend/27-i18n/README.en.md) |

### 后端技术 (20 项)

| 分类 | 技术 | 中文 | English |
|------|------|------|---------|
| 语言 | Java | [README.zh.md](./04-tech/02-backend/01-java/README.zh.md) | [README.en.md](./04-tech/02-backend/01-java/README.en.md) |
| 框架 | Spring Boot | [README.zh.md](./04-tech/02-backend/02-spring-boot/README.zh.md) | [README.en.md](./04-tech/02-backend/02-spring-boot/README.en.md) |
| 框架 | Spring Cloud | [README.zh.md](./04-tech/02-backend/03-spring-cloud/README.zh.md) | [README.en.md](./04-tech/02-backend/03-spring-cloud/README.en.md) |
| ORM | MyBatis | [README.zh.md](./04-tech/02-backend/04-mybatis/README.zh.md) | [README.en.md](./04-tech/02-backend/04-mybatis/README.en.md) |
| 数据库 | MySQL | [README.zh.md](./04-tech/02-backend/05-mysql/README.zh.md) | [README.en.md](./04-tech/02-backend/05-mysql/README.en.md) |
| 缓存 | Redis | [README.zh.md](./04-tech/02-backend/06-redis/README.zh.md) | [README.en.md](./04-tech/02-backend/06-redis/README.en.md) |
| 消息队列 | RocketMQ | [README.zh.md](./04-tech/02-backend/07-rocketmq/README.zh.md) | [README.en.md](./04-tech/02-backend/07-rocketmq/README.en.md) |
| 消息队列 | Kafka | [README.zh.md](./04-tech/02-backend/08-kafka/README.zh.md) | [README.en.md](./04-tech/02-backend/08-kafka/README.en.md) |
| 搜索 | Elasticsearch | [README.zh.md](./04-tech/02-backend/09-elasticsearch/README.zh.md) | [README.en.md](./04-tech/02-backend/09-elasticsearch/README.en.md) |
| 微服务 | Nacos | [README.zh.md](./04-tech/02-backend/10-nacos/README.zh.md) | [README.en.md](./04-tech/02-backend/10-nacos/README.en.md) |
| 微服务 | Sentinel | [README.zh.md](./04-tech/02-backend/11-sentinel/README.zh.md) | [README.en.md](./04-tech/02-backend/11-sentinel/README.en.md) |
| 微服务 | Seata | [README.zh.md](./04-tech/02-backend/12-seata/README.zh.md) | [README.en.md](./04-tech/02-backend/12-seata/README.en.md) |
| 可观测性 | SkyWalking | [README.zh.md](./04-tech/02-backend/13-skywalking/README.zh.md) | [README.en.md](./04-tech/02-backend/13-skywalking/README.en.md) |
| 容器化 | Docker | [README.zh.md](./04-tech/02-backend/14-docker/README.zh.md) | [README.en.md](./04-tech/02-backend/14-docker/README.en.md) |
| 容器化 | Kubernetes | [README.zh.md](./04-tech/02-backend/15-kubernetes/README.zh.md) | [README.en.md](./04-tech/02-backend/15-kubernetes/README.en.md) |
| 网关/代理 | Nginx | [README.zh.md](./04-tech/02-backend/16-nginx/README.zh.md) | [README.en.md](./04-tech/02-backend/16-nginx/README.en.md) |
| 网关 | Gateway | [README.zh.md](./04-tech/02-backend/17-gateway/README.zh.md) | [README.en.md](./04-tech/02-backend/17-gateway/README.en.md) |
| 安全 | Security | [README.zh.md](./04-tech/02-backend/18-security/README.zh.md) | [README.en.md](./04-tech/02-backend/18-security/README.en.md) |
| 可观测性 | Logging | [README.zh.md](./04-tech/02-backend/19-logging/README.zh.md) | [README.en.md](./04-tech/02-backend/19-logging/README.en.md) |
| 可观测性 | Monitoring | [README.zh.md](./04-tech/02-backend/20-monitoring/README.zh.md) | [README.en.md](./04-tech/02-backend/20-monitoring/README.en.md) |

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
请参考 03-dlc/04-testing/01-unit/README.zh.md 帮我编写单元测试
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
