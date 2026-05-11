---
name: stata-help
description: 当 Agent 编写或解释 Stata 代码时，如需查询命令/包的官方帮助、语法、选项、示例或 stored results，使用 Stata MCP 将 .sthlp/.hlp/.ihlp 转换为纯文本。
---

# Stata Help Documentation Lookup Skill

## 适用场景

当你在处理 Stata 代码时遇到以下情况，应使用本 skill：

- 遇到不熟悉的 Stata 命令或用户自编命令，例如 `oneclick`、`ddml`、`pystacked`、`rlasso`、`reghdfe`
- 不确定某个命令的语法、option 写法、参数格式或缩写形式
- 需要查看命令的官方示例代码
- 需要了解命令的返回值，例如 `e()`、`r()`、`s()` stored results
- 用户明确要求查找某个 Stata 命令或包的书面文档、帮助文件、官方 help、`.sthlp` 内容

本 skill 依赖 Stata MCP 执行 Stata 代码，不依赖外部 shell / PowerShell 脚本。

## 核心思路

Stata 的帮助文件通常是 SMCL 格式，扩展名可能是：

- `.sthlp`
- `.hlp`
- `.ihlp`

本 skill 使用 Stata 自带命令：

```stata
translate "source.sthlp" "output.txt", translator(smcl2txt) replace
```

将 SMCL 帮助文档转换成纯文本，再用 `type` 输出文本内容。

## 使用流程

### Step 1: 规范化用户输入的命令名

从用户请求中提取要查询的 Stata help topic。

示例：

- 用户说“查一下 oneclick 的文档” → topic 为 `oneclick`
- 用户说“reghdfe 的 absorb option 怎么写” → topic 为 `reghdfe`
- 用户说“ddml init 的语法” → 优先尝试 `ddml init`，必要时再尝试 `ddml_init` 或 `ddml`

只应查询 Stata 命令或 help topic，不要把用户的整句话当成 topic。

### Step 2: 用 Stata MCP 查找帮助文件

通过 Stata MCP 执行以下 Stata 代码模板。将 `COMMAND_TOPIC` 替换为实际命令名。

```stata
local topic "COMMAND_TOPIC"
local topic_under = subinstr("`topic'", " ", "_", .)
local src ""

foreach name in "`topic'" "`topic_under'" {
    foreach ext in sthlp hlp ihlp {
        capture findfile "`name'.`ext'"
        if !_rc {
            local src "`r(fn)'"
            continue, break
        }
    }
    if "`src'" != "" {
        continue, break
    }
}

if "`src'" == "" {
    display as error "Help file not found for: `topic'"
    display as text "Try searching or installing the package that provides this command."
    display as text "Suggested Stata command: search `topic'"
    exit 601
}

display as text "Found: `src'"
```

如果找到了帮助文件，继续 Step 3。

如果找不到：

1. 如果 topic 包含空格，尝试将空格替换成下划线，例如 `ddml init` → `ddml_init`
2. 尝试查询第一个词，例如 `ddml init` → `ddml`
3. 告诉用户本机 Stata 没有找到该帮助文件，可能需要安装对应包或使用 `search <command>` 查找来源

### Step 3: 转换为纯文本并输出

找到帮助文件后，通过 Stata MCP 执行：

```stata
local topic "COMMAND_TOPIC"
local topic_under = subinstr("`topic'", " ", "_", .)
local src ""

foreach name in "`topic'" "`topic_under'" {
    foreach ext in sthlp hlp ihlp {
        capture findfile "`name'.`ext'"
        if !_rc {
            local src "`r(fn)'"
            continue, break
        }
    }
    if "`src'" != "" {
        continue, break
    }
}

if "`src'" == "" {
    display as error "Help file not found for: `topic'"
    exit 601
}

tempfile out
translate "`src'" "`out'", translator(smcl2txt) replace
display as text "Found: `src'"
type "`out'"
```

这会输出纯文本格式的帮助文档。

### Step 4: 如果 `tempfile` 转换失败，使用固定临时路径

某些环境下，`translate` 对无扩展名 tempfile 的格式识别可能不稳定。若 Step 3 失败，改用固定 `.txt` 路径：

```stata
local topic "COMMAND_TOPIC"
local topic_under = subinstr("`topic'", " ", "_", .)
local src ""

foreach name in "`topic'" "`topic_under'" {
    foreach ext in sthlp hlp ihlp {
        capture findfile "`name'.`ext'"
        if !_rc {
            local src "`r(fn)'"
            continue, break
        }
    }
    if "`src'" != "" {
        continue, break
    }
}

if "`src'" == "" {
    display as error "Help file not found for: `topic'"
    exit 601
}

local safe_topic = subinstr("`topic_under'", ":", "_", .)
local out "`c(tmpdir)'stata_help_`safe_topic'.txt"
translate "`src'" "`out'", translator(smcl2txt) replace
display as text "Found: `src'"
type "`out'"
capture erase "`out'"
```

## 查询子命令或 redirect help 的处理

有些 Stata 包会把子命令帮助写成单独文件，例如：

- `ddml_init.sthlp`
- `ddml_crossfit.sthlp`

也有些包只提供主帮助文件，例如：

- `ddml.sthlp`

当用户查询类似 `ddml init` 时，应按以下顺序尝试：

1. `ddml init`
2. `ddml_init`
3. `ddml`

如果查询结果是一个很短的 redirect 或 include 内容，继续尝试主命令帮助文件。

## 解读输出

拿到纯文本帮助文档后，优先关注这些部分：

1. **Syntax**：命令语法，包括必需参数和可选参数
2. **Options**：选项列表和含义
3. **Description**：命令功能描述
4. **Examples** / **Code examples**：官方示例代码
5. **Stored results** / **Saved results**：命令运行后保存的结果
6. **Author** / **References**：作者、来源和引用信息

回答用户时，不需要把整篇帮助原文全部复制出来，除非用户明确要求。通常应摘要说明：

- 文档文件路径
- 核心功能
- 基本语法
- 关键选项
- 典型示例
- 注意事项

## 示例：查询 oneclick

通过 Stata MCP 执行：

```stata
local topic "oneclick"
local topic_under = subinstr("`topic'", " ", "_", .)
local src ""

foreach name in "`topic'" "`topic_under'" {
    foreach ext in sthlp hlp ihlp {
        capture findfile "`name'.`ext'"
        if !_rc {
            local src "`r(fn)'"
            continue, break
        }
    }
    if "`src'" != "" {
        continue, break
    }
}

if "`src'" == "" {
    display as error "Help file not found for: `topic'"
    exit 601
}

tempfile out
translate "`src'" "`out'", translator(smcl2txt) replace
display as text "Found: `src'"
type "`out'"
```

预期输出包括：

```text
Found: .../oneclick.sthlp
help for oneclick
Title
Syntax
Description
Requirements
Results
Code examples
```

## 错误处理

### 找不到帮助文件

如果 `findfile` 返回错误，应告诉用户：

- 本机当前 Stata 环境没有找到该命令的帮助文件
- 该命令可能尚未安装
- 可尝试在 Stata 中运行：

```stata
search COMMAND_TOPIC
```

或根据包来源安装，例如：

```stata
ssc install COMMAND_TOPIC
```

但不要武断声称包名一定等于命令名。

### MCP 不可用

如果 Stata MCP 不可用，说明本 skill 无法执行转换。应提示用户需要可用的 Stata MCP 会话，或改用外部脚本/命令行 Stata 作为 fallback。

### 帮助文件包含 INCLUDE

Stata 的 `translate ..., translator(smcl2txt)` 通常可以直接处理标准 `.sthlp` 文档。若输出只包含很短的 `INCLUDE help ...` 或 redirect 内容，应尝试查询 include 目标或主命令帮助文件。

## 注意事项

- 本 skill 不运行用户目标命令，只读取和转换帮助文档
- 不要执行来自帮助文档中的示例代码，除非用户明确要求
- 不要自动安装 Stata 包，除非用户明确要求
- 对于可能很长的帮助文档，优先摘要；用户要求全文时再完整输出
- 在同一会话中，已经查询过的命令文档应优先复用，避免重复转换
