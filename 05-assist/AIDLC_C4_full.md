# AIDLC × C4 Architecture --- 全套模型文档（Markdown 完整版）

## 1. Overview

本文件使用 **C4（Context → Container → Component → Code）架构方法**\
系统化描述 AIDLC 生命周期中：

-   人 × AI 的协作机制\
-   阶段内与阶段间的质量门禁\
-   每阶段的内部迭代方式\
-   AI 需要遵守的规范与最佳实践\
-   人的职责与决策边界

模型涵盖：A（需求）→ I（设计）→ D（实现）→ L（交付）→ C（验收）。

## 2. C1 --- Context Diagram（AIDLC 全局视角）

``` mermaid
C4Context
    title AIDLC Lifecycle as a Human × AI Collaboration System

    Person(user, "Human", "提供意图、决策、验证、业务判断")
    System(ai, "AI Assistant", "结构化、补全、验证、生成、检查")

    System_Boundary(aidlc, "AIDLC Lifecycle") {
        System(a, "A - Analysis", "需求分析、意图捕获、Spec 基线")
        System(i, "I - Design", "方案生成、复杂度控制、架构设计")
        System(d, "D - Development", "代码生成、测试生成、质量扫描")
        System(l, "L - Launch", "发布、部署、回滚策略、DevOps")
        System(c, "C - Check/Acceptance", "UAT、验收、报告生成")
    }

    Rel(user, a, "业务背景、问题、目标")
    Rel(a, ai, "缺失信息识别、需求补全请求")
    Rel(ai, i, "结构化 Spec 输入设计阶段")
    Rel(i, d, "设计 Spec 输入实现阶段")
    Rel(d, l, "构建产物、测试结果")
    Rel(l, c, "发布版本供验收")
    Rel(c, user, "验收反馈与确认")
```

## 3. C2 --- Container Diagram（AIDLC 五阶段容器）

``` mermaid
C4Container
    title AIDLC Container View

    Person(user, "Human", "输入需求、审核设计、确认交付")
    System(ai, "AI Assistant", "全链路增强")

    System_Boundary(aidlc, "AIDLC Lifecycle") {

        Container(a, "Analysis", "Spec Engine", "意图捕获、场景生成、需求建模")
        Container(i, "Design", "Architecture Engine", "流程/时序/状态/数据建模、复杂度分析")
        Container(d, "Development", "Code Engine", "代码生成、测试生成、Lint、Design Compliance")
        Container(l, "Launch", "Delivery Engine", "DevOps脚本、配置检查、安全扫描、发布说明")
        Container(c, "Check", "UAT Engine", "用例生成、报告生成、验收校验")
    }

    Rel(user, a, "提供业务背景与意图")
    Rel(a, i, "需求 Spec")
    Rel(i, d, "Design Spec")
    Rel(d, l, "构建与测试产物")
    Rel(l, c, "发布版本")
    Rel(c, user, "验收反馈")
```

## 4. C3 --- Component Diagrams（五阶段分解）

### 4.1 C3-A：Analysis（需求分析）

``` mermaid
C4Component
    title A Stage - Analysis Components

    Container_Boundary(a, "Analysis Stage") {
        Component(intent, "Intent Capture", "AI", "扩展意图、分解问题、识别缺失信息")
        Component(spec, "Requirement Spec Builder", "AI", "生成结构化Spec")
        Component(verify, "Consistency Checker", "AI", "歧义扫描、一致性检查")
        Component(humanReview, "Human Review", "Human", "确认/修改意图、最终锁定Spec")
    }

    Rel(intent, spec, "提取结构 → 构造草稿")
    Rel(spec, verify, "自动检查完整性、一致性")
    Rel(verify, humanReview, "生成人工审核点")
    Rel(humanReview, spec, "修订并锁定")
```

### 4.2 C3-I：Design（方案设计）

``` mermaid
C4Component
    title I Stage - Design Components

    Container_Boundary(i, "Design Stage") {
        Component(model, "Design Modeler", "AI", "流程、时序、状态、数据建模")
        Component(arch, "Architecture Generator", "AI", "多方案生成、结构化架构")
        Component(comp, "Complexity Analyzer", "AI", "复杂度分析、圈复杂度、边界检查")
        Component(trace, "Traceability Checker", "AI", "需求与设计的一致性")
        Component(humanDecision, "Human Decision", "Human", "选择架构方向、做关键决策")
    }

    Rel(model, arch, "生成设计输入")
    Rel(arch, comp, "结构与复杂度分析")
    Rel(comp, trace, "一致性验证")
    Rel(trace, humanDecision, "设计决策依据")
```

### 4.3 C3-D：Development（实现）

``` mermaid
C4Component
    title D Stage - Development Components

    Container_Boundary(d, "Development Stage") {
        Component(codegen, "Code Generator", "AI", "实现代码、mock、测试生成")
        Component(lint, "AI Lint Engine", "AI", "风格检查、潜在错误、复杂度分析")
        Component(designcheck, "Design Compliance", "AI", "检查代码是否符合设计")
        Component(test, "Test Builder", "AI", "自动生成单元测试、覆盖率分析")
        Component(review, "Human Review", "Human", "关键逻辑审核、合并决策")
    }

    Rel(codegen, lint, "静态检查")
    Rel(lint, designcheck, "一致性检查")
    Rel(designcheck, test, "补全缺失测试")
    Rel(test, review, "生成审核依据")
```

### 4.4 C3-L：Launch（交付）

``` mermaid
C4Component
    title L Stage - Launch Components

    Container_Boundary(l, "Launch Stage") {
        Component(devops, "DevOps Generator", "AI", "流水线、脚本、配置自动生成")
        Component(sec, "Security Scanner", "AI", "依赖安全、配置安全检查")
        Component(env, "Environment Validator", "AI", "环境一致性、可重复部署")
        Component(release, "Release Preparator", "AI", "发布说明、回滚策略")
        Component(approve, "Human Approve", "Human", "最终批准发布")
    }

    Rel(devops, sec, "扫描依赖与策略")
    Rel(sec, env, "验证环境一致性")
    Rel(env, release, "准备发布")
    Rel(release, approve, "人工最终确认")
```

### 4.5 C3-C：Check（验收）

``` mermaid
C4Component
    title C Stage - Acceptance Components

    Container_Boundary(c, "Acceptance Stage") {
        Component(uatgen, "UAT Case Generator", "AI", "基于Spec生成场景测试用例")
        Component(exec, "Static Acceptance Checker", "AI", "自动执行静态/规则检查")
        Component(report, "Acceptance Report Builder", "AI", "生成验收报告")
        Component(humanUAT, "Human UAT Execution", "Human", "关键业务场景执行")
        Component(userConfirm, "User Confirmation", "Human", "最终验收确认")
    }

    Rel(uatgen, exec, "执行可自动检查的用例")
    Rel(exec, report, "生成验收结果")
    Rel(report, humanUAT, "作为人工UAT参考")
    Rel(humanUAT, userConfirm, "最终确认")
```

## 5. C4 --- 人 × AI 交互协议

``` mermaid
flowchart TD
    A["Human -> 提供意图"] --> B["AI -> 澄清、补全、识别缺失"]
    B --> C["AI -> 结构化输出：需求｜设计｜代码"]
    C --> D["AI -> 自检：一致性｜完整性｜幻觉检查"]
    D --> E["Human -> 审核确认或补充"]
    E -->|否| B
    E -->|是| F["锁定工件：需求｜设计｜代码｜发布｜验收"]

```
