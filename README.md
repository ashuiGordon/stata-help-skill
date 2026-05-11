# stata-help Skill

`stata-help` 是一个面向 Stata 用户和 AI Agent 的文档查询 Skill。

它通过 Stata MCP 调用本机 Stata 帮助系统，自动查找 `.sthlp`、`.hlp`、`.ihlp` 帮助文件，并使用 Stata 自带的 `translate, translator(smcl2txt)` 将 SMCL 文档转换为纯文本，帮助 Agent 基于本机官方文档理解 Stata 命令的语法、选项、示例和 stored results。

## 为什么需要它

AI Agent 在生成 Stata 代码时，容易对用户自编命令或 SSC 命令产生语法幻觉，例如 option 名称写错、参数格式不准确、示例代码与本机版本不一致。

Stata 的 help 文件通常和本机安装的命令版本一致，因此让 Agent 先读取本机 help，再回答问题，可以显著提高 Stata 代码生成和解释的可靠性。

## 适用场景

- 查询 Stata 命令的官方帮助文档
- 查看 SSC 或用户自编命令的语法和选项
- 根据 help 文件生成更可靠的示例代码
- 理解 Stata 命令的 stored results
- 辅助解释论文复现代码或 `.do` 文件

## 工作原理

核心 Stata 流程如下：

```stata
findfile command.sthlp
translate command.sthlp output.txt, translator(smcl2txt) replace
type output.txt
```

Skill 会指导 Agent：

1. 从用户问题中提取 Stata help topic；
2. 使用 Stata MCP 执行 `findfile` 查找帮助文件；
3. 尝试 `.sthlp`、`.hlp`、`.ihlp` 等扩展名；
4. 使用 `translate, translator(smcl2txt)` 转换为纯文本；
5. 根据输出总结 Syntax、Options、Examples、Stored results 等内容。

## 文件说明

- `skill.md`：Skill 主文件，包含调用条件、Stata MCP 查询流程、错误处理和示例。

## 使用要求

- 本机已安装 Stata；
- 当前 Agent 环境可调用 Stata MCP；
- 要查询的 Stata 命令或包已安装在本机 Stata 环境中。

## 示例

用户提问：

> 帮我查一下 `oneclick` 的文档。

Agent 会通过 Stata MCP 查找：

```stata
findfile oneclick.sthlp
```

然后执行：

```stata
translate "oneclick.sthlp" "output.txt", translator(smcl2txt) replace
type "output.txt"
```

最后基于本机 help 文件总结 `oneclick` 的功能、语法、选项和示例。

## 注意事项

- 本 Skill 只读取帮助文档，不运行目标 Stata 命令；
- 本 Skill 不会自动安装 Stata 包；
- 如果本机未安装目标命令，可能需要先在 Stata 中使用 `search command_name` 查找来源；
- 对于子命令，如 `ddml init`，Skill 会尝试 `ddml init`、`ddml_init` 和 `ddml` 等候选 topic。

## License

MIT License
