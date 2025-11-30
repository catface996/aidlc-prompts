# Anthropic 官方提示词工程教程总结

> 基于 [anthropics/prompt-eng-interactive-tutorial](https://github.com/anthropics/prompt-eng-interactive-tutorial) 仓库内容的系统化总结

---

## 教程结构概览

### 课程目标

- 掌握优秀提示词的基本结构
- 识别常见的失败模式
- 理解 Claude 的优势和局限
- 从零开始构建强大的提示词

### 章节体系

| 级别 | 章节 | 主题 | 核心技术 |
|------|------|------|---------|
| **入门** | Ch1 | 基本提示词结构 | Messages API, System Prompt |
| **入门** | Ch2 | 清晰直接 | Golden Rule, 直接表达 |
| **入门** | Ch3 | 角色分配 | Role Prompting |
| **中级** | Ch4 | 数据与指令分离 | XML Tags, 结构化输入 |
| **中级** | Ch5 | 格式化输出 | Output Formatting, Prefilling |
| **中级** | Ch6 | 预认知/逐步思考 | Chain of Thought |
| **中级** | Ch7 | 使用示例 | Few-Shot Prompting |
| **高级** | Ch8 | 避免幻觉 | Hallucination Prevention |
| **高级** | Ch9 | 复杂提示词构建 | 多元素组合 |
| **附录** | 10.1 | 提示词链 | Prompt Chaining |
| **附录** | 10.2 | 工具使用 | Tool Use |
| **附录** | 10.3 | 性能评估 | Empirical Evaluation |
| **附录** | 10.4 | 搜索与检索 | RAG |

---

## 核心技术要点

### 1. 清晰直接 (Being Clear and Direct)

**黄金法则**：
> 将你的提示词展示给同事或朋友，让他们按照指令操作，看看能否产生你想要的结果。如果他们感到困惑，Claude 也会困惑。

**关键原则**：
- Claude 没有任何上下文，除了你字面告诉它的
- 越直接明确地解释你想要什么，Claude 的响应就越好、越准确
- 像对待新员工一样对待 Claude

**实践技巧**：

| 技巧 | 说明 | 示例 |
|------|------|------|
| **直接要求** | 明确说出你想要什么 | "Skip the preamble" |
| **强制决策** | 让 Claude 必须做出选择 | "if you absolutely had to pick one" |
| **指定长度** | 明确输出长度要求 | "Write at least 800 words" |
| **指定语言** | 明确输出语言 | "Respond in Spanish" |

### 2. 角色分配 (Role Prompting)

**核心理念**：
- 给 Claude 分配特定角色可以改善其在各领域的表现
- 角色可以改变 Claude 响应的风格、语气和方式
- 角色越详细，效果越好

**应用场景**：

| 角色类型 | 效果 | 示例 |
|---------|------|------|
| **专业角色** | 提升专业领域准确性 | "You are a logic bot" |
| **性格角色** | 改变语气风格 | "You are a cat" |
| **受众设定** | 调整内容复杂度 | "Explain to a 5-year-old" |

**最佳实践**：
- 角色定义可以放在 System Prompt 或 User Message 中
- 可以同时指定 Claude 的身份和目标受众
- 专业角色可以帮助 Claude 更好地完成数学和逻辑任务

### 3. 数据与指令分离 (Separating Data from Instructions)

**核心技术**：使用 XML 标签将数据与指令分开

```xml
Here is the document:
<document>
{DOCUMENT_CONTENT}
</document>

Based on the document above, answer this question:
<question>
{USER_QUESTION}
</question>
```

**好处**：
- 清晰区分什么是指令，什么是数据
- 防止数据内容被误解为指令
- 便于程序化处理输出

### 4. 格式化输出 (Formatting Output)

**三大技术**：

| 技术 | 说明 | 用途 |
|------|------|------|
| **XML 标签包装** | 用 XML 标签包裹输出 | 便于程序解析 |
| **预填充 (Prefilling)** | 在 assistant 轮次预先填入开头 | 引导输出格式 |
| **格式指令** | 明确说明输出格式 | 确保一致性 |

**预填充示例**：
```python
messages=[
  {"role": "user", "content": prompt},
  {"role": "assistant", "content": "<response>"}  # 预填充
]
```

### 5. 逐步思考 (Thinking Step by Step)

**核心原则**：
> 让 Claude 有时间思考，可以提高准确性，尤其是对于复杂任务。

**关键点**：
- 思考必须"大声说出来"——Claude 需要显式展示推理过程
- 在给出答案前要求 Claude 思考
- 可以使用 `<thinking>` 标签分离思考过程和最终答案

**实践示例**：
```
Before you answer, pull out the most relevant quotes in <relevant_quotes> tags.
Then, provide your answer in <answer> tags.
```

### 6. 使用示例 (Few-Shot Prompting)

**核心理念**：
> 示例可能是知识工作中让 Claude 按预期行为的最有效工具。

**应用场景**：
- 获得正确答案
- 获得正确格式的答案
- 控制语气和风格

**最佳实践**：
- 一般来说，示例越多越好
- 示例应覆盖常见边缘情况
- 如果使用 scratchpad，示例应展示 scratchpad 的样子

### 7. 避免幻觉 (Avoiding Hallucinations)

**三大策略**：

| 策略 | 说明 | 实现方式 |
|------|------|---------|
| **给予退出选项** | 允许 Claude 说"不知道" | "If you're not sure, say 'I don't know'" |
| **要求先收集证据** | 先引用再回答 | "First extract relevant quotes" |
| **低温度设置** | 减少创造性 | `temperature=0` |

**示例**：
```
If there is not sufficient information to answer, you may write
"Sorry, I do not have sufficient information to answer this question."
```

### 8. 复杂提示词的十大要素

Anthropic 官方推荐的复杂提示词结构：

| # | 元素 | 英文 | 说明 | 必要性 |
|---|------|------|------|--------|
| 1 | 用户角色 | User Role | 确保以 user 角色开始 | MUST |
| 2 | 任务背景 | Task Context | Claude 的角色和目标 | SHOULD |
| 3 | 语气背景 | Tone Context | 期望的语气风格 | COULD |
| 4 | 详细任务描述和规则 | Task Description & Rules | 具体任务和约束 | SHOULD |
| 5 | 示例 | Examples | 理想响应的示例 | SHOULD |
| 6 | 输入数据 | Input Data | 需要处理的数据 | DEPENDS |
| 7 | 即时任务描述 | Immediate Task | 重申当前任务 | SHOULD |
| 8 | 预认知/逐步思考 | Precognition | 要求先思考再回答 | COULD |
| 9 | 输出格式 | Output Formatting | 指定输出格式 | SHOULD |
| 10 | 预填充 | Prefilling | 预先填入响应开头 | COULD |

**元素顺序建议**：
```
1. Task Context (任务背景) - 放在开头
2. Tone Context (语气背景)
3. Task Description & Rules (详细规则)
4. Examples (示例)
5. Input Data (输入数据) - 灵活位置
6. Immediate Task (即时任务) - 放在靠近末尾
7. Precognition (逐步思考) - 放在末尾
8. Output Formatting (输出格式) - 放在末尾
9. Prefilling (预填充) - assistant 轮次
```

---

## 评估方法论

### 三种评估方式

基于附录 10.3 中的评估方法：

| 评估方式 | 适用场景 | 优点 | 缺点 |
|---------|---------|------|------|
| **代码评估** | 有明确正确答案的任务 | 自动化、快速、一致 | 只适用于可量化结果 |
| **人工评估** | 创意、主观任务 | 可评估复杂输出 | 耗时、成本高 |
| **模型评估** | 需要判断的任务 | 可扩展、自动化 | 需要好的评分标准 |

### 代码评估示例

```python
def grade_exercise(text):
    # 检查是否包含预期内容
    return "expected_keyword" in text.lower()
```

### 模型评估示例

```python
def build_grader_prompt(output, rubric):
    return f"""Assess the quality based on this rubric:
    <rubric>{rubric}</rubric>
    <output>{output}</output>
    Provide a score from 1-5."""
```

---

## 与 PQEF 评分体系的对应关系

将 Anthropic 教程中的技术点映射到 PQEF 评分维度：

| PQEF 维度 | Anthropic 对应技术 | 章节来源 |
|-----------|-------------------|---------|
| **Ⅰ. 清晰性** | Golden Rule, 直接表达, XML 分离 | Ch2, Ch4 |
| **Ⅱ. 可控性** | Output Formatting, Prefilling, Examples | Ch5, Ch7 |
| **Ⅲ. 任务性** | Immediate Task, Step-by-Step | Ch6, Ch9 |
| **Ⅳ. 鲁棒性** | Avoiding Hallucinations, "I don't know" | Ch8 |
| **Ⅴ. 可扩展性** | Role Prompting, Task Context, 模块化结构 | Ch3, Ch9 |

---

## 提示词质量检查清单

基于 Anthropic 教程总结的检查清单：

### 清晰性检查
- [ ] 指令是否直接明确？
- [ ] 是否使用 XML 标签分离数据和指令？
- [ ] 同事/朋友能否理解并执行？

### 可控性检查
- [ ] 是否指定了输出格式？
- [ ] 是否使用预填充引导格式？
- [ ] 是否提供了示例？

### 任务性检查
- [ ] 是否有明确的任务描述？
- [ ] 是否在末尾重申了即时任务？
- [ ] 复杂任务是否要求逐步思考？

### 鲁棒性检查
- [ ] 是否给 Claude 退出选项？
- [ ] 是否要求先收集证据再回答？
- [ ] 是否考虑了边缘情况？

### 可扩展性检查
- [ ] 是否定义了角色和背景？
- [ ] 提示词结构是否模块化？
- [ ] 是否可以复用到其他场景？

---

## 关键语录

> "Claude has no context aside from what you literally tell it."
> — Chapter 2

> "Examples are probably the single most effective tool in knowledge work for getting Claude to behave as desired."
> — Chapter 7

> "Prompt engineering is about scientific trial and error."
> — Chapter 9

> "Not all prompts need every element. Use many elements to get your prompt working first, then refine and slim down afterward."
> — Chapter 9

---

*总结版本：v1.0*
*基于仓库：anthropics/prompt-eng-interactive-tutorial*
