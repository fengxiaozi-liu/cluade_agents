# Skill 机制设计说明

## 文档目的

本文档面向不了解当前代码仓库实现细节的读者，独立说明一套 Agent 与 Skill 的协作机制。

目标是回答四个问题：

1. Agent 是如何使用 Skill 的
2. Skill 是如何传递给模型的
3. 模型是如何知道“该怎么用 Skill”的
4. 整个系统为什么要采用这种设计

本文不依赖任何具体代码文件路径，可以单独阅读和转发。

## 一句话定义

Skill 可以理解为一种“按需加载的专业能力包”。

它本质上是一段结构化说明，里面定义了：

- 这个能力叫什么
- 适合什么时候使用
- 允许使用哪些工具
- 接收什么参数
- 真正执行时应该遵循什么步骤

Agent 则是负责决策和调用这些 Skill 的执行者。

模型不是在启动时就读到所有 Skill 的完整正文，而是：

1. 先知道有哪些 Skill 可用
2. 再知道应该通过哪个工具调用 Skill
3. 最后在真正需要时，才读取某个 Skill 的完整内容

这就是整套机制的核心。

---

## 核心角色

在这套设计里，可以把系统拆成四个角色：

### 1. Skill

Skill 是能力定义本身。

它通常包含两层内容：

- 元信息
  - 名称
  - 描述
  - 使用时机
  - 参数说明
  - 权限要求
- 正文说明
  - 具体步骤
  - 约束
  - 注意事项
  - 可执行策略

Skill 更像“操作手册”或“可复用工作流模板”，而不是程序代码本身。

### 2. Agent

Agent 是负责解决用户问题的智能执行体。

它的职责包括：

- 理解用户意图
- 判断当前任务是否适合某个 Skill
- 按规则调用 Skill
- 在 Skill 展开后继续执行任务

Agent 不需要一开始就把所有 Skill 背下来，它只需要知道“可以用哪些 Skill”以及“如何调用它们”。

### 3. Model

Model 是真正执行推理和生成的模型。

模型本身不会天然知道某个项目里有哪些 Skill，也不会天然知道该如何调用它们。

这些知识必须由系统显式告诉它，包括：

- 有哪些 Skill
- 它们大概做什么
- 什么时候应该调用
- 调用时要使用哪个工具
- 调用后会发生什么

### 4. Skill Tool

Skill Tool 是 Skill 机制中的桥梁。

它承担三个关键职责：

- 让模型能够正式发起 Skill 调用
- 在调用前做合法性校验
- 在调用后把 Skill 的完整正文注入到对话中

可以把它理解成：

“模型访问 Skill 的唯一正规入口”。

---

## 为什么不把所有 Skill 全文直接塞给模型

这是整个设计最关键的出发点。

如果系统一开始就把所有 Skill 的完整正文都放进模型上下文，会有几个问题：

### 1. 上下文成本高

Skill 数量多时，prompt 会迅速膨胀。

这会带来：

- token 消耗增加
- 成本上升
- 推理速度下降
- 模型注意力被分散

### 2. 噪音变大

模型并不需要每一轮都看到所有 Skill 的详细说明。

大多数任务只会命中极少数几个 Skill。

如果把所有正文都提前注入，会让大量无关说明进入当前上下文，反而降低判断质量。

### 3. 权限控制更难

有些 Skill 会附带额外权限、工具使用规则或特殊执行方式。

如果一开始就全部展开，权限边界会变模糊，不利于控制和审计。

### 4. 审计能力差

按需加载时，系统可以明确知道：

- 模型什么时候决定调用了某个 Skill
- 调用了哪个 Skill
- Skill 具体展开了什么内容

这对于调试、分析和安全控制都更友好。

所以，更合理的设计是：

- 先暴露 Skill 索引
- 后按需加载 Skill 正文

---

## 整体流程

整个 Skill 机制可以分成四个阶段：

1. Skill 注册阶段
2. Skill 暴露阶段
3. Skill 调用阶段
4. Skill 执行阶段

下面按顺序展开。

## 第一阶段：Skill 注册

这一阶段的目标是把磁盘上的 Skill 定义，变成系统运行时可识别的对象。

典型步骤如下：

1. 扫描技能目录
2. 找到每个 Skill 的定义文件
3. 解析元信息和正文
4. 转成内存中的 Skill 对象
5. 注册进全局可用技能列表

这一阶段做完之后，系统知道：

- 当前有哪些 Skill
- 每个 Skill 的名称是什么
- 描述是什么
- 是否允许模型调用
- 是否有参数
- 是否有特殊权限或特殊上下文要求

注意：

这时候系统只是“知道这个 Skill 存在”，并没有把 Skill 正文都交给模型。

---

## 第二阶段：Skill 暴露给 Agent 和 Model

这一阶段的重点不是传全文，而是传“索引”和“调用规则”。

系统通常会把下面两类信息提供给模型：

### 1. Skill 列表

也就是一份精简的能力清单，通常包含：

- Skill 名称
- Skill 描述
- 使用时机

它的作用是让模型先做能力发现。

模型通过这份清单知道：

- 当前能用哪些 Skill
- 哪个 Skill 可能和用户任务匹配

### 2. Skill 使用规则

系统还会通过 system prompt 或 tool prompt 告诉模型：

- 当用户提到某个 slash command 时，应该把它理解成 Skill
- 不能凭空猜测 Skill
- 必须通过 Skill Tool 来调用
- 一旦判断某个 Skill 与任务匹配，应该优先调用 Skill Tool

这一步非常重要。

因为模型真正知道“如何使用 Skill”，靠的不是记忆，而是系统明确给出的调用契约。

---

## 第三阶段：模型调用 Skill

当模型发现某个 Skill 与当前任务匹配时，会发起一次 Skill Tool 调用。

例如它可能表达为：

- 要调用哪个 Skill
- 需要传什么参数

这时候系统不会立刻盲目信任，而是进入校验阶段。

校验通常包括：

- 这个 Skill 是否存在
- 这个 Skill 是否允许模型调用
- 参数是否合法
- 这个 Skill 是否属于 prompt 型技能
- 当前权限是否允许执行这个 Skill

如果不合法，则拒绝调用。
如果合法，才进入下一步。

---

## 第四阶段：Skill 展开并注入模型上下文

当 Skill 校验通过后，系统才会把该 Skill 的完整正文展开成模型可见内容。

这一阶段通常会做以下处理：

1. 把 Skill 正文读出来
2. 替换参数占位符
3. 注入技能目录、会话信息等运行时变量
4. 处理与权限、shell、工具相关的扩展逻辑
5. 把最终文本作为消息插入当前对话

到了这一步，模型才真正“读到”这个 Skill 的完整说明。

也就是说，Skill 正文不是一开始就在模型脑子里，而是在调用时才被送进去。

然后模型会把这段 Skill 正文当作当前任务的一部分继续执行。

---

## Agent 与 Skill 的关系

Agent 和 Skill 的关系，不是“Agent 包含 Skill”，而是“Agent 在合适的时候调度 Skill”。

更准确地说：

- Agent 负责判断
- Skill 负责提供专业流程
- Skill Tool 负责桥接
- Model 负责最终推理执行

所以 Skill 更像是 Agent 的外挂能力模块，而不是 Agent 自己的固定知识。

---

## Agent 与 Skill 流程图

下面是一个独立于具体代码实现的流程图。

```text
+------------------+
| 用户提出任务      |
+------------------+
          |
          v
+------------------+
| Agent 理解任务    |
+------------------+
          |
          v
+--------------------------+
| Agent 查看可用 Skill 列表 |
+--------------------------+
          |
          v
+------------------------------+
| Agent 判断是否有匹配的 Skill |
+------------------------------+
      | 是                      | 否
      v                         v
+----------------------+   +----------------------+
| 通过 Skill Tool 调用  |   | 直接按常规流程执行    |
| Skill                |   | 任务                  |
+----------------------+   +----------------------+
          |
          v
+----------------------+
| 系统校验 Skill 调用   |
+----------------------+
          |
          v
+----------------------+
| 展开 Skill 正文       |
+----------------------+
          |
          v
+----------------------+
| 注入当前对话上下文    |
+----------------------+
          |
          v
+----------------------+
| Agent 继续执行任务    |
+----------------------+
          |
          v
+----------------------+
| 返回结果给用户        |
+----------------------+
```

---

## Skill 与 Model 的关系

Skill 与 Model 的关系也很重要。

可以这样理解：

- Model 本身不拥有 Skill
- Model 只是被系统告知“有哪些 Skill 可用”
- Model 再通过规则决定是否调用某个 Skill
- 真正调用后，Skill 的正文才进入 Model 的工作上下文

因此，Skill 对 Model 来说不是“常驻知识”，而是“按需注入的外部能力说明”。

这种设计很像：

- 检索增强中的按需检索
- 工具调用中的按需执行
- 工作流引擎中的按需装配

只是这里装配的不是数据库内容，而是一段能力说明。

---

## Skill 与 Model 流程图

```text
+--------------------------+
| 系统先给模型两类信息      |
| 1. Skill 列表            |
| 2. Skill 使用规则        |
+--------------------------+
             |
             v
+--------------------------+
| 模型理解用户任务          |
+--------------------------+
             |
             v
+------------------------------+
| 模型判断是否需要某个 Skill   |
+------------------------------+
      | 需要                     | 不需要
      v                          v
+--------------------------+   +----------------------+
| 模型调用 Skill Tool       |   | 模型继续普通推理      |
+--------------------------+   +----------------------+
             |
             v
+--------------------------+
| 系统校验 Skill 调用是否合法 |
+--------------------------+
             |
             v
+--------------------------+
| 系统展开 Skill 完整正文    |
+--------------------------+
             |
             v
+--------------------------+
| Skill 正文进入模型上下文   |
+--------------------------+
             |
             v
+--------------------------+
| 模型按 Skill 说明继续执行  |
+--------------------------+
```

---

## 一个更贴近真实运行的过程

为了让这个机制更容易理解，可以看一个典型例子。

假设用户说：

“帮我提交一下这次改动。”

系统并不会一开始就把所有 Skill 的完整正文都交给模型。

相反，模型先看到的是：

- 有一个 `commit` Skill
- 它适合在代码准备提交时使用
- 如果需要使用它，要通过 Skill Tool

于是模型判断：

当前任务很匹配 `commit` Skill。

然后模型发起 Skill Tool 调用：

- skill = `commit`
- args = 可选参数

系统通过校验后，才把 `commit` Skill 的正文注入当前对话。

Skill 正文里可能写着：

- 检查改动文件
- 总结修改内容
- 生成提交信息

模型这时才真正读取到这些步骤，并据此继续执行。

这个例子很能说明整套机制的本质：

模型先知道“有这个能力”，再在需要时读取“这个能力的详细说明”。

---

## 预加载模式

除了按需加载，还存在一种预加载模式。

在某些场景里，系统会在 Agent 创建时就指定：

- 这个 Agent 天生应该带哪些 Skill

这种模式常用于：

- 某类专用 sub-agent
- 某个固定工作流的代理
- 明确需要绑定特定能力的执行单元

在这种情况下，Skill 不需要等模型自己决定是否调用，而是在 agent 初始化阶段就被注入上下文。

这和普通模式的差别在于：

- 普通模式是先暴露索引，后按需展开
- 预加载模式是直接把选定 Skill 提前注入

预加载的优点是：

- 响应更直接
- 减少一次调用判断

缺点是：

- 上下文成本更高
- 灵活性更弱

所以一般只适合少量、明确、稳定的技能集合。

---

## 我对这套设计的理解

如果用一句更工程化的话来总结，我的理解是：

这是一套“Skill 目录索引 + Tool 调用 + 按需正文注入”的能力编排机制。

它的核心思想不是“把能力写死在模型里”，而是：

- 让模型先看到能力目录
- 用统一入口管理能力调用
- 在真正命中时再把详细说明加载进来

我认为这套设计有四个非常明显的优点。

### 1. 成本控制合理

绝大多数任务不会使用全部 Skill。

先给索引、再按需展开，能明显降低上下文开销。

### 2. 对模型更友好

模型不需要在一大堆无关 Skill 正文里找重点。

先看清单，再命中，再展开，决策链路更自然。

### 3. 易于治理

Skill 是通过统一工具入口调用的，因此更容易：

- 做权限限制
- 做调用日志
- 做审计
- 做调试

### 4. 可扩展性强

新增 Skill 时，系统通常不需要大改架构。

只要能注册进技能索引，并遵守统一调用协议，模型就能使用。

---

## 设计本质

我认为这套机制的本质，不是“给模型加一个 prompt 模板库”，而是：

把 Skill 作为一种可检索、可调度、可审计的外部能力单元。

它介于下面几种东西之间：

- prompt 模板
- 工具调用
- 轻量工作流
- 运行时知识注入

因此它特别适合解决这类问题：

- 某些任务有固定流程，但不应该每轮都占上下文
- 某些能力需要复用，但不适合写死在主系统 prompt 中
- 某些工作流需要明确边界、权限和审计能力

---

## 推荐的对外表达方式

如果你要把这套机制讲给外部读者，我建议可以用下面这句话：

Skill 是一种按需加载的能力模块。  
系统先把 Skill 的“目录和使用规则”告诉模型，模型在任务命中时通过统一工具调用某个 Skill，系统再把该 Skill 的完整说明注入当前上下文，帮助模型按既定流程完成任务。

这句话通常足够准确，也足够容易被理解。

---

## Go 示例

下面给一个独立的 Go 示例，演示如何实现一套类似的机制。

这个示例分成三部分：

1. 注册 Skill
2. 给模型构造 Skill 列表和调用规则
3. 在模型命中 Skill 时再展开正文

```go
package main

import (
	"bufio"
	"errors"
	"fmt"
	"os"
	"path/filepath"
	"strings"
)

type Skill struct {
	Name               string
	Description        string
	WhenToUse          string
	UserInvocable      bool
	DisableModelInvoke bool
	BaseDir            string
	Body               string
}

type SkillRegistry struct {
	skills map[string]Skill
}

func NewSkillRegistry() *SkillRegistry {
	return &SkillRegistry{
		skills: map[string]Skill{},
	}
}

func (r *SkillRegistry) LoadDir(root string) error {
	entries, err := os.ReadDir(root)
	if err != nil {
		return err
	}

	for _, entry := range entries {
		if !entry.IsDir() {
			continue
		}

		name := entry.Name()
		skillDir := filepath.Join(root, name)
		skillFile := filepath.Join(skillDir, "SKILL.md")

		raw, err := os.ReadFile(skillFile)
		if err != nil {
			if os.IsNotExist(err) {
				continue
			}
			return err
		}

		skill, err := parseSkillFile(name, skillDir, string(raw))
		if err != nil {
			return fmt.Errorf("parse skill %s failed: %w", skillFile, err)
		}

		r.skills[skill.Name] = skill
	}

	return nil
}

func (r *SkillRegistry) ListForModel() []string {
	var out []string
	for _, skill := range r.skills {
		if skill.DisableModelInvoke {
			continue
		}

		desc := skill.Description
		if skill.WhenToUse != "" {
			desc += " - " + skill.WhenToUse
		}

		out = append(out, fmt.Sprintf("- %s: %s", skill.Name, desc))
	}
	return out
}

func (r *SkillRegistry) ExpandSkill(name, args, sessionID string) (string, error) {
	skill, ok := r.skills[name]
	if !ok {
		return "", errors.New("unknown skill: " + name)
	}

	if skill.DisableModelInvoke {
		return "", errors.New("skill is not allowed for model invocation: " + name)
	}

	body := skill.Body
	body = strings.ReplaceAll(body, "$ARGUMENTS", args)
	body = strings.ReplaceAll(body, "${CLAUDE_SKILL_DIR}", filepath.ToSlash(skill.BaseDir))
	body = strings.ReplaceAll(body, "${CLAUDE_SESSION_ID}", sessionID)

	return fmt.Sprintf("Base directory for this skill: %s\n\n%s", skill.BaseDir, body), nil
}

type Agent struct {
	registry *SkillRegistry
}

func NewAgent(registry *SkillRegistry) *Agent {
	return &Agent{registry: registry}
}

func (a *Agent) BuildSystemPrompt() string {
	lines := []string{
		"你可以使用 Skills 来增强任务执行能力。",
		"当任务匹配某个 Skill 时，应通过 SkillTool 发起调用。",
		"不要猜测不存在的 Skill，只能使用下方列出的 Skill。",
		"",
		"可用 Skill 列表：",
	}

	lines = append(lines, a.registry.ListForModel()...)
	return strings.Join(lines, "\n")
}

func (a *Agent) HandleTask(userTask string) string {
	return "这里通常会接入模型推理。模型先看到 system prompt 和 skill 列表，再决定是否调用 SkillTool。"
}

func (a *Agent) InvokeSkill(name, args string) (string, error) {
	return a.registry.ExpandSkill(name, args, "session-123")
}

func parseSkillFile(name, baseDir, raw string) (Skill, error) {
	fm, body := parseFrontmatter(raw)

	skill := Skill{
		Name:               name,
		Description:        fm["description"],
		WhenToUse:          fm["when_to_use"],
		UserInvocable:      parseBoolDefaultTrue(fm["user-invocable"]),
		DisableModelInvoke: parseBoolDefaultFalse(fm["disable-model-invocation"]),
		BaseDir:            baseDir,
		Body:               strings.TrimSpace(body),
	}

	if skill.Description == "" {
		skill.Description = firstNonEmptyLine(skill.Body)
	}

	return skill, nil
}

func parseFrontmatter(raw string) (map[string]string, string) {
	result := map[string]string{}

	if !strings.HasPrefix(raw, "---\n") {
		return result, raw
	}

	parts := strings.SplitN(raw, "\n---\n", 2)
	if len(parts) != 2 {
		return result, raw
	}

	header := strings.TrimPrefix(parts[0], "---\n")
	body := parts[1]

	scanner := bufio.NewScanner(strings.NewReader(header))
	for scanner.Scan() {
		line := strings.TrimSpace(scanner.Text())
		if line == "" || strings.HasPrefix(line, "#") {
			continue
		}

		key, value, ok := strings.Cut(line, ":")
		if !ok {
			continue
		}

		result[strings.TrimSpace(key)] = strings.TrimSpace(value)
	}

	return result, body
}

func firstNonEmptyLine(s string) string {
	scanner := bufio.NewScanner(strings.NewReader(s))
	for scanner.Scan() {
		line := strings.TrimSpace(scanner.Text())
		if line != "" {
			return line
		}
	}
	return ""
}

func parseBoolDefaultTrue(v string) bool {
	if v == "" {
		return true
	}
	return strings.EqualFold(v, "true")
}

func parseBoolDefaultFalse(v string) bool {
	if v == "" {
		return false
	}
	return strings.EqualFold(v, "true")
}

func main() {
	registry := NewSkillRegistry()
	if err := registry.LoadDir("./skills"); err != nil {
		panic(err)
	}

	agent := NewAgent(registry)

	fmt.Println("=== 传给模型的 Skill 索引和规则 ===")
	fmt.Println(agent.BuildSystemPrompt())

	fmt.Println()
	fmt.Println("=== 模型命中某个 Skill 后，系统展开其正文 ===")
	expanded, err := agent.InvokeSkill("commit", "fix login bug")
	if err != nil {
		panic(err)
	}

	fmt.Println(expanded)
}
```

### 示例 Skill 文件

```md
---
description: 生成一个清晰的 git commit 提交说明
when_to_use: 当代码变更已经完成并准备提交时使用
user-invocable: true
disable-model-invocation: false
---

你正在帮助用户完成一次代码提交。

参数：$ARGUMENTS
技能目录：${CLAUDE_SKILL_DIR}
会话 ID：${CLAUDE_SESSION_ID}

请遵循以下步骤：
1. 检查当前改动
2. 总结核心变化
3. 输出清晰且简洁的提交说明
```

---

## 最终总结

如果只保留一句最重要的话，我会这样总结这套机制：

Agent 并不是一开始就拥有所有 Skill 的完整内容，而是先知道“有哪些 Skill 可以用”，再通过统一的 Skill Tool 在需要时请求某个 Skill，系统随后把该 Skill 的完整说明注入模型上下文，帮助模型按专业流程执行任务。

这是一种兼顾成本、清晰度、权限控制和扩展性的设计。
