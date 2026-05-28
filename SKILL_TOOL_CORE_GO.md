# SkillTool Core Logic in Go

本文把当前 TypeScript 里的 SkillTool 核心调用链翻译成 Go 风格伪实现，用来帮助理解运行时主干。

对应源码：

| 逻辑 | TypeScript 源码 |
| --- | --- |
| SkillTool 主调用入口 | [src/tools/SkillTool/SkillTool.ts](/c/Users/Administrator/Desktop/myproject/cluade_agents/src/tools/SkillTool/SkillTool.ts:577) `call()` |
| inline skill 展开入口 | [src/utils/processUserInput/processSlashCommand.tsx](/c/Users/Administrator/Desktop/myproject/cluade_agents/src/utils/processUserInput/processSlashCommand.tsx:817) `processPromptSlashCommand()` |
| inline skill 生成消息 | [src/utils/processUserInput/processSlashCommand.tsx](/c/Users/Administrator/Desktop/myproject/cluade_agents/src/utils/processUserInput/processSlashCommand.tsx:827) `getMessagesForPromptSlashCommand()` |
| 完整 SKILL.md 内容展开 | [src/skills/loadSkillsDir.ts](/c/Users/Administrator/Desktop/myproject/cluade_agents/src/skills/loadSkillsDir.ts:270) `createSkillCommand().getPromptForCommand()` |
| fork skill 上下文准备 | [src/utils/forkedAgent.ts](/c/Users/Administrator/Desktop/myproject/cluade_agents/src/utils/forkedAgent.ts:191) `prepareForkedCommandContext()` |

## 1. 核心数据结构

```go
package skillruntime

import (
	"context"
	"fmt"
	"strings"
)

type CommandType string

const (
	CommandTypePrompt CommandType = "prompt"
)

type ExecutionContext string

const (
	ExecutionInline ExecutionContext = "inline"
	ExecutionFork   ExecutionContext = "fork"
)

type ContentBlock struct {
	Type string
	Text string
}

type Message struct {
	Type              string
	Content           any
	IsMeta            bool
	SourceToolUseID   string
}

type Attachment struct {
	Type         string
	AllowedTools []string
	Model        string
}

type Command struct {
	Type                   CommandType
	Name                   string
	Source                 string
	LoadedFrom             string
	Description            string
	AllowedTools           []string
	Model                  string
	Effort                 *int
	Context                ExecutionContext
	DisableModelInvocation bool
	SkillRoot              string
	Hooks                  *HooksSettings

	GetPromptForCommand func(ctx context.Context, args string, tuc ToolUseContext) ([]ContentBlock, error)
}

type ToolUseContext struct {
	SessionID       string
	AgentID         string
	Commands        []Command
	Messages        []Message
	AppState         AppState
}

type AppState struct {
	AlwaysAllowCommands []string
	Effort              *int
}

type SlashCommandResult struct {
	Messages     []Message
	ShouldQuery  bool
	AllowedTools []string
	Model        string
	Effort       *int
	Command      Command
}

type ToolResult struct {
	Success      bool
	CommandName  string
	Status       string
	AllowedTools []string
	Model        string
	NewMessages  []Message
	ContextPatch func(ToolUseContext) ToolUseContext
}

type HooksSettings struct{}
```

## 2. SkillTool.call 的 Go 版本

TypeScript 的主干在 `SkillTool.ts:577`。Go 版本可以理解成下面这样：

```go
func CallSkillTool(
	ctx context.Context,
	inputSkill string,
	args string,
	tuc ToolUseContext,
	parentToolUseID string,
) (ToolResult, error) {
	// 1. 标准化技能名：兼容模型传入 "/skill-name"。
	commandName := strings.TrimSpace(inputSkill)
	commandName = strings.TrimPrefix(commandName, "/")
	if commandName == "" {
		return ToolResult{}, fmt.Errorf("empty skill name")
	}

	// 2. 从当前上下文拿到所有可调用命令。
	// 对应 TS: getAllCommands(context)，包含 local commands + MCP skills。
	commands := GetAllCommands(tuc)
	command, ok := FindCommand(commandName, commands)
	if !ok {
		return ToolResult{}, fmt.Errorf("unknown skill: %s", commandName)
	}

	// 3. validateInput 已经做过一次；这里展示核心约束。
	if command.DisableModelInvocation {
		return ToolResult{}, fmt.Errorf("skill %s cannot be model-invoked", commandName)
	}
	if command.Type != CommandTypePrompt {
		return ToolResult{}, fmt.Errorf("skill %s is not prompt-based", commandName)
	}

	RecordSkillUsage(commandName)

	// 4. fork skill：不把内容注入主对话，而是开子 agent 执行。
	if command.Context == ExecutionFork {
		return ExecuteForkedSkill(ctx, command, commandName, args, tuc)
	}

	// 5. inline skill：展开成 meta user message，交回主循环继续问模型。
	processed, err := ProcessPromptSlashCommand(ctx, commandName, args, commands, tuc)
	if err != nil {
		return ToolResult{}, err
	}
	if !processed.ShouldQuery {
		return ToolResult{}, fmt.Errorf("command processing failed")
	}

	// 6. 过滤掉展示用 command-message，只把真正需要进模型上下文的消息挂到 toolUseID 下。
	newMessages := TagMessagesWithToolUseID(
		FilterProgressAndCommandMetadata(processed.Messages),
		parentToolUseID,
	)

	allowedTools := processed.AllowedTools
	model := processed.Model
	effort := command.Effort

	// 7. 返回 ToolResult。contextPatch 对应 TS 里的 contextModifier：
	// 后续主循环会把 allowed tools、model、effort 应用到 ToolUseContext。
	return ToolResult{
		Success:      true,
		CommandName:  commandName,
		Status:       "inline",
		AllowedTools: allowedTools,
		Model:        model,
		NewMessages:  newMessages,
		ContextPatch: func(next ToolUseContext) ToolUseContext {
			if len(allowedTools) > 0 {
				next.AppState.AlwaysAllowCommands = UnionStrings(
					next.AppState.AlwaysAllowCommands,
					allowedTools,
				)
			}
			if effort != nil {
				next.AppState.Effort = effort
			}
			// model override 在真实实现中还要保留 [1m] suffix 等细节。
			return next
		},
	}, nil
}
```

### 2.1 模型调用 SkillTool 后，真正发回模型的是什么

这里容易误解：**模型不是只收到 metadata，然后再自己去找详细数据。**

实际链路是：

```text
模型调用 SkillTool({ skill: "xxx" })
  -> 运行时在 command registry 里找到 xxx 对应的 Command
  -> 运行时调用 command.getPromptForCommand(args, context)
  -> getPromptForCommand 读取/展开完整 SKILL.md 内容
  -> processPromptSlashCommand 生成 messages
  -> SkillTool.call 过滤掉展示用 metadata message
  -> 返回 ToolResult + newMessages
  -> 下一次模型请求里包含完整 skill content
```

因此，SkillTool 返回给模型的主要内容分两层：

| 内容 | 是否包含详细 SKILL.md | 作用 |
| --- | --- | --- |
| tool result | 否 | 告诉模型工具调用成功，例如 `Launching skill: xxx`。 |
| newMessages 中的 meta user message | 是 | 真正把完整展开后的 SKILL.md 注入下一轮模型上下文。 |
| metadata user message | SkillTool 路径中通常不会进入模型 | 用于 UI/命令显示和 slash-command 路径；SkillTool 会过滤 `<command-message>`。 |
| command_permissions attachment | 否 | 把 skill frontmatter 的 `allowed-tools` 和 `model` 传给运行时上下文。 |

对应 TypeScript 位置：

| 逻辑 | 源码 |
| --- | --- |
| `processPromptSlashCommand()` 先生成 metadata + meta skill content | [src/utils/processUserInput/processSlashCommand.tsx](/c/Users/Administrator/Desktop/myproject/cluade_agents/src/utils/processUserInput/processSlashCommand.tsx:817) |
| `getMessagesForPromptSlashCommand()` 调用 `command.getPromptForCommand()` 拿详细内容 | [src/utils/processUserInput/processSlashCommand.tsx](/c/Users/Administrator/Desktop/myproject/cluade_agents/src/utils/processUserInput/processSlashCommand.tsx:866) |
| 生成 metadata message + `isMeta: true` skill content message | [src/utils/processUserInput/processSlashCommand.tsx](/c/Users/Administrator/Desktop/myproject/cluade_agents/src/utils/processUserInput/processSlashCommand.tsx:903) |
| SkillTool 过滤掉 `<command-message>` metadata | [src/tools/SkillTool/SkillTool.ts](/c/Users/Administrator/Desktop/myproject/cluade_agents/src/tools/SkillTool/SkillTool.ts:712) |
| SkillTool 返回 `newMessages` | [src/tools/SkillTool/SkillTool.ts](/c/Users/Administrator/Desktop/myproject/cluade_agents/src/tools/SkillTool/SkillTool.ts:758) |

更贴近真实行为的 Go 版本可以拆成下面这样：

```go
func CallSkillToolAndBuildNextModelInput(
	ctx context.Context,
	inputSkill string,
	args string,
	tuc ToolUseContext,
	parentToolUseID string,
) (ToolResult, []Message, error) {
	result, err := CallSkillTool(ctx, inputSkill, args, tuc, parentToolUseID)
	if err != nil {
		return ToolResult{}, nil, err
	}

	// ToolResult 本身映射成 tool_result block，内容很短：
	toolResultMessage := Message{
		Type:    "tool_result",
		Content: "Launching skill: " + result.CommandName,
	}

	// result.NewMessages 才包含完整 skill content。
	// 后续 query 会把 tool_result + newMessages 一起放进下一次模型请求。
	nextModelMessages := append([]Message{toolResultMessage}, result.NewMessages...)

	return result, nextModelMessages, nil
}
```

## 3. inline skill 展开逻辑

对应 `processPromptSlashCommand()` 和 `getMessagesForPromptSlashCommand()`。

```go
func ProcessPromptSlashCommand(
	ctx context.Context,
	commandName string,
	args string,
	commands []Command,
	tuc ToolUseContext,
) (SlashCommandResult, error) {
	command, ok := FindCommand(commandName, commands)
	if !ok {
		return SlashCommandResult{}, fmt.Errorf("unknown command: %s", commandName)
	}
	if command.Type != CommandTypePrompt {
		return SlashCommandResult{}, fmt.Errorf("expected prompt command: %s", commandName)
	}

	return GetMessagesForPromptSlashCommand(ctx, command, args, tuc)
}

func GetMessagesForPromptSlashCommand(
	ctx context.Context,
	command Command,
	args string,
	tuc ToolUseContext,
) (SlashCommandResult, error) {
	// 1. 调用 command.getPromptForCommand，真正拿到完整技能内容。
	blocks, err := command.GetPromptForCommand(ctx, args, tuc)
	if err != nil {
		return SlashCommandResult{}, err
	}

	// 2. 注册 skill hooks。真实代码还会检查 plugin-only policy/source trust。
	if command.Hooks != nil {
		RegisterSkillHooks(tuc.SessionID, command.Name, command.SkillRoot, command.Hooks)
	}

	// 3. 保存已展开内容，供 compaction 后用 invoked_skills 恢复。
	skillContent := JoinTextBlocks(blocks, "\n\n")
	skillPath := command.Name
	if command.Source != "" {
		skillPath = command.Source + ":" + command.Name
	}
	AddInvokedSkill(command.Name, skillPath, skillContent, tuc.AgentID)

	// 4. allowed-tools 从 frontmatter 进入权限附件和后续 contextPatch。
	allowedTools := ParseToolList(command.AllowedTools)

	// 5. 生成两条关键消息：
	// metadata message 用于 UI/防重复调用；
	// meta content message 才是完整 SKILL.md 展开后输入模型的内容。
	metadata := FormatCommandLoadingMetadata(command, args)
	mainContent := blocks

	messages := []Message{
		{
			Type:    "user",
			Content: metadata,
		},
		{
			Type:    "user",
			Content: mainContent,
			IsMeta:  true,
		},
		{
			Type: "attachment",
			Content: Attachment{
				Type:         "command_permissions",
				AllowedTools: allowedTools,
				Model:        command.Model,
			},
		},
	}

	return SlashCommandResult{
		Messages:     messages,
		ShouldQuery:  true,
		AllowedTools: allowedTools,
		Model:        command.Model,
		Effort:       command.Effort,
		Command:      command,
	}, nil
}
```

在 SkillTool 路径中，上面 `messages` 会继续被 `SkillTool.call` 处理：

```go
func FilterProgressAndCommandMetadata(messages []Message) []Message {
	out := make([]Message, 0, len(messages))
	for _, msg := range messages {
		if msg.Type == "progress" {
			continue
		}

		// 这就是 SkillTool.ts 里的关键过滤：
		// metadata 里有 <command-message>，SkillTool 会把它拿掉。
		// 因为 SkillTool 自己会渲染“正在启动 skill”的 UI，
		// 不需要再把这条 metadata 塞给模型。
		if text, ok := msg.Content.(string); ok && strings.Contains(text, "<command-message>") {
			continue
		}

		out = append(out, msg)
	}
	return out
}
```

过滤后仍会留下：

```go
[]Message{
	{
		Type:    "user",
		Content: []ContentBlock{{Type: "text", Text: "<完整展开后的 SKILL.md>"}},
		IsMeta:  true,
	},
	{
		Type: "attachment",
		Content: Attachment{
			Type:         "command_permissions",
			AllowedTools: []string{"..."},
			Model:        "...",
		},
	},
}
```

所以详细数据不是由模型“找到”的，而是运行时先找到 `Command`，再调用 `GetPromptForCommand` 主动展开并注入给模型。

## 4. getPromptForCommand 的 Go 版本

这段对应 `createSkillCommand()` 里闭包形式的 `getPromptForCommand()`。它是“完整 SKILL.md 如何变成模型输入”的核心。

```go
func CreateSkillCommand(
	skillName string,
	markdownContent string,
	baseDir string,
	allowedTools []string,
	argNames []string,
	loadedFrom string,
	sessionID func() string,
	shell ShellConfig,
) Command {
	return Command{
		Type:         CommandTypePrompt,
		Name:         skillName,
		AllowedTools: allowedTools,
		SkillRoot:    baseDir,

		GetPromptForCommand: func(ctx context.Context, args string, tuc ToolUseContext) ([]ContentBlock, error) {
			finalContent := markdownContent

			// 1. 有 skill root 时，注入 base directory header。
			if baseDir != "" {
				finalContent = "Base directory for this skill: " + baseDir + "\n\n" + markdownContent
			}

			// 2. 替换 $ARGUMENTS 或命名参数。
			finalContent = SubstituteArguments(finalContent, args, argNames)

			// 3. 替换 ${CLAUDE_SKILL_DIR}。
			// Windows 下真实实现会把 "\" 转成 "/"，避免 shell escape 问题。
			if baseDir != "" {
				skillDir := NormalizeSkillDirForShell(baseDir)
				finalContent = strings.ReplaceAll(finalContent, "${CLAUDE_SKILL_DIR}", skillDir)
			}

			// 4. 替换 ${CLAUDE_SESSION_ID}。
			finalContent = strings.ReplaceAll(finalContent, "${CLAUDE_SESSION_ID}", sessionID())

			// 5. 本地 skill 允许执行 SKILL.md 里的 inline shell。
			// MCP skill 是远程不可信内容，不能执行。
			if loadedFrom != "mcp" {
				shellCtx := tuc
				shellCtx.AppState.AlwaysAllowCommands = allowedTools

				expanded, err := ExecuteShellCommandsInPrompt(
					ctx,
					finalContent,
					shellCtx,
					"/"+skillName,
					shell,
				)
				if err != nil {
					return nil, err
				}
				finalContent = expanded
			}

			return []ContentBlock{
				{Type: "text", Text: finalContent},
			}, nil
		},
	}
}
```

## 5. fork skill 的核心差异

fork skill 不走 `ProcessPromptSlashCommand()` 往主对话塞 meta message，而是把展开后的 skill content 当成子 agent 的第一条 user message。

```go
func ExecuteForkedSkill(
	ctx context.Context,
	command Command,
	commandName string,
	args string,
	tuc ToolUseContext,
) (ToolResult, error) {
	prepared, err := PrepareForkedCommandContext(ctx, command, args, tuc)
	if err != nil {
		return ToolResult{}, err
	}

	agentResult, err := RunAgent(ctx, RunAgentInput{
		AgentDefinition: prepared.AgentDefinition,
		PromptMessages:  prepared.PromptMessages,
		ToolUseContext:  prepared.ModifiedContext,
		Model:           command.Model,
	})
	if err != nil {
		return ToolResult{}, err
	}

	return ToolResult{
		Success:     true,
		CommandName: commandName,
		Status:      "forked",
		NewMessages: []Message{
			{
				Type:    "tool_result",
				Content: ExtractResultText(agentResult.Messages, "Skill execution completed"),
			},
		},
	}, nil
}

type PreparedForkContext struct {
	SkillContent    string
	ModifiedContext ToolUseContext
	AgentDefinition AgentDefinition
	PromptMessages  []Message
}

func PrepareForkedCommandContext(
	ctx context.Context,
	command Command,
	args string,
	tuc ToolUseContext,
) (PreparedForkContext, error) {
	blocks, err := command.GetPromptForCommand(ctx, args, tuc)
	if err != nil {
		return PreparedForkContext{}, err
	}

	skillContent := JoinTextBlocks(blocks, "\n")
	allowedTools := ParseToolList(command.AllowedTools)

	modified := tuc
	modified.AppState.AlwaysAllowCommands = UnionStrings(
		modified.AppState.AlwaysAllowCommands,
		allowedTools,
	)

	agent := ResolveAgentDefinition(command)

	return PreparedForkContext{
		SkillContent:    skillContent,
		ModifiedContext: modified,
		AgentDefinition: agent,
		PromptMessages: []Message{
			{Type: "user", Content: skillContent},
		},
	}, nil
}
```

## 6. 最小辅助函数占位

这些函数在真实 TypeScript 里分散在多个模块，这里只保留语义。

```go
type ShellConfig struct{}
type AgentDefinition struct{}
type RunAgentInput struct {
	AgentDefinition AgentDefinition
	PromptMessages  []Message
	ToolUseContext  ToolUseContext
	Model           string
}
type RunAgentResult struct {
	Messages []Message
}

func GetAllCommands(tuc ToolUseContext) []Command { return tuc.Commands }
func FindCommand(name string, commands []Command) (Command, bool) {
	for _, cmd := range commands {
		if cmd.Name == name {
			return cmd, true
		}
	}
	return Command{}, false
}

func RecordSkillUsage(name string) {}
func RegisterSkillHooks(sessionID, skillName, skillRoot string, hooks *HooksSettings) {}
func AddInvokedSkill(name, path, content, agentID string) {}
func ParseToolList(input []string) []string { return input }
func FormatCommandLoadingMetadata(command Command, args string) string {
	return fmt.Sprintf("<command-message>%s</command-message>\n<command-name>/%s</command-name>", command.Name, command.Name)
}
func JoinTextBlocks(blocks []ContentBlock, sep string) string {
	var parts []string
	for _, block := range blocks {
		if block.Type == "text" {
			parts = append(parts, block.Text)
		}
	}
	return strings.Join(parts, sep)
}
func TagMessagesWithToolUseID(messages []Message, toolUseID string) []Message {
	for i := range messages {
		messages[i].SourceToolUseID = toolUseID
	}
	return messages
}
func FilterProgressAndCommandMetadata(messages []Message) []Message {
	out := make([]Message, 0, len(messages))
	for _, msg := range messages {
		if msg.Type == "progress" {
			continue
		}
		if text, ok := msg.Content.(string); ok && strings.Contains(text, "<command-message>") {
			continue
		}
		out = append(out, msg)
	}
	return out
}
func UnionStrings(a, b []string) []string {
	seen := map[string]bool{}
	var out []string
	for _, item := range append(a, b...) {
		if !seen[item] {
			seen[item] = true
			out = append(out, item)
		}
	}
	return out
}
func SubstituteArguments(content, args string, names []string) string {
	return strings.ReplaceAll(content, "$ARGUMENTS", args)
}
func NormalizeSkillDirForShell(path string) string {
	return strings.ReplaceAll(path, "\\", "/")
}
func ExecuteShellCommandsInPrompt(ctx context.Context, content string, tuc ToolUseContext, commandName string, shell ShellConfig) (string, error) {
	return content, nil
}
func RunAgent(ctx context.Context, input RunAgentInput) (RunAgentResult, error) {
	return RunAgentResult{}, nil
}
func ExtractResultText(messages []Message, fallback string) string { return fallback }
func ResolveAgentDefinition(command Command) AgentDefinition { return AgentDefinition{} }
```

## 7. 主干总结

```text
SkillTool.call
  -> 标准化 skill name
  -> getAllCommands + findCommand
  -> fork ? executeForkedSkill : processPromptSlashCommand
  -> command.getPromptForCommand 展开完整 SKILL.md
  -> 注册 hooks + addInvokedSkill
  -> 生成 meta user message + command_permissions attachment
  -> 返回 newMessages + contextModifier
```
