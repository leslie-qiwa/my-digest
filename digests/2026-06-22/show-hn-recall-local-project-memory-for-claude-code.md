# Show HN: Recall – Local project memory for Claude Code

# Recall

Claude Code 每次启动都是"冷启动"。Recall 会在本地保存会话日志，并将其压缩为一份随时可恢复的摘要——完全在你的机器上完成。无需 API 密钥，无需外部模型，不会向任何地方发送数据。它专为在订阅模式下本地运行 Claude Code 的用户打造：唯一参与的 AI 就是 Claude Code 本身；摘要由经典 Python 提取式算法生成。

- **订阅即可免费使用。** 它解决了冷启动问题——不再需要每次重新向 Claude 解释项目——而且无需额外调用计量模型。摘要由本地算法生成而非 LLM 调用，因此持久记忆不会产生订阅费之外的任何成本。
- **节省使用额度。** 两方面：(1) 摘要在本地构建，捕获和更新记忆不消耗任何模型 token；(2) 从紧凑的 `context.md`（约 1–2K token）恢复，而非每次从头解释项目，意味着每次会话消耗的 token 大幅减少——从而延长订阅的使用额度（或在 API 模式下降低计费额度）。
- **数据不离开你的机器。** 你的会话记录（代码、路径，有时包含密钥）绝不会发送到任何 API。大多数"记忆"工具会将你的上下文发送到模型端点；Recall 提供了它们无法做出的隐私保证。
- **零摩擦。** 无需 `pip install`，无需本地模型，无需配置密钥，离线可用。插件加载后即刻生效。

## 工作原理

两个文件，写入项目的 `.recall/` 目录下：

- `history.md` —— 日志。仅追加模式。每次会话实时记录于此（你的提示词、Claude 的回复、涉及的文件和执行的命令）。
- `context.md` —— 摘要。由本地摘要器覆写——浓缩的"当前进度"，加载到下一次会话中：目标、摘要、后续步骤/未完成事项、涉及的文件以及上次停留的位置。

## 与 Claude Code 内置记忆的关系

Claude Code 自带记忆功能——而 Recall 是补充，不是替代。内置选项解决的是不同问题：

- `CLAUDE.md`（和 `#` 快捷方式）是手动编写的记忆：你维护的规则和备注，作为 Claude 遵循的指令加载。适合表达"我希望你怎样工作"，但需要手动维护，且不记录会话中实际发生了什么。
- `--continue` / `--resume` 重放之前的对话——完全保真，但会重新加载整个对话记录（token 开销大），且绑定在单台机器的本地会话历史上，不是便携可读的摘要。
- 上下文压缩在会话内压缩对话；它不是一个可在数天后重新打开的持久记录。

Recall 填补了这些之间的空白：自动、确定性地记录每次会话的内容，压缩为紧凑的恢复点。

| | `CLAUDE.md` / `#` | `--continue` / `--resume` | Recall |
|---|---|---|---|
| 是什么 | 手动编写的备注和规则 | 重新加载之前的对话 | 自动捕获的会话日志 + 本地摘要 |
| 维护成本 | 手动 | 无（你选择会话） | 无——工作时自动写入 |
| 包含内容 | 要遵循的指令 | 完整的之前对话 | 目标、文件、命令、进度和后续步骤 |
| 恢复成本 | 小 | 大（重放完整对话） | 约 1–2K token（紧凑摘要） |
| 形式 | 你编辑的 Markdown | 本地会话状态 | `.recall/` 中的纯文本——可 diff、可共享 |
| Claude 如何处理 | 视为指令 | 视为对话 | 标记为不可信参考数据 |

简而言之：`CLAUDE.md` 是"我希望你怎样工作"；Recall 是"上次我们做了什么、停在哪里"——离线生成，不消耗任何模型 token。

## 生命周期

| 时刻 | 发生了什么 |
|---|---|
| 会话期间 | Stop / SessionEnd 钩子将新活动追加到 `.recall/history.md`。捕获是增量的（仅新轮次）且完全本地化。 |
| 会话开始时 | SessionStart 钩子展示 `context.md`，并让 Claude 向你确认两件事：是否从保存的上下文恢复？以及是否继续记录本次会话？ |
| 结束前 | 你运行 `/recall:save`。本地摘要器读取 `history.md` 并覆写 `context.md`。 |
| …或自动 | 设置 `auto_save_context: "on_end"`，每次会话结束时 `context.md` 自动重新生成——无需手动 `/recall:save`。 |

## 摘要器

整个过程没有任何 LLM 调用——摘要由本地运行的 TF-IDF + TextRank（提取式摘要）生成。

`scripts/summarizer.py` 对会话中最核心的句子进行排序：

- TF-IDF 句向量
- 句子之间的余弦相似度图
- TextRank——在该图上进行 PageRank 幂迭代——为句子评分
- 保留得分最高的前 N 个句子，按原始顺序排列

`context.md` 将该摘要与从对话记录和 git 中直接提取的确定性信息组合：目标（你的第一个请求）、涉及的文件、执行的命令、上次停留的位置，以及 `git diff --stat`。

无需安装依赖。整个 TF-IDF + TextRank 实现已内嵌在 `summarizer.py` 中。如果 `numpy` 恰好可导入，则用于向量化运算（大型会话更快）；否则，使用完全相同的纯 Python TextRank。相同的算法，相同的结果——`numpy` 只是可选的加速器，绝非必须。保存时的输出会告诉你走了哪条路径。

## 命令

- `/recall:save` —— 运行本地摘要器 → 覆写 `context.md`。
- `/recall:show` —— 打印 `context.md`。
- `/recall:log` —— 查看 `history.md` 尾部。

## 配置

在项目根目录放置配置文件以覆盖默认值：

| 键 | 默认值 | 用途 |
|---|---|---|
| `output_dir` | `".recall"` | `history.md` / `context.md` 的存放位置。 |
| `capture_history` | `true` | 将会话活动追加到 `history.md`。 |
| `auto_save_context` | `"off"` | 会话结束时重新生成 `context.md`：`"off"` 或 `"on_end"`。 |
| `summary_sentences` | `8` | 摘要保留的句子数。 |
| `redact` | `true` | 写入 md 文件前去除明显的密钥。 |
| `include_git` | `true` | 在 `context.md` 中添加 `git diff --stat` + 最近提交。 |
| `max_input_chars` | `200000` | 输入摘要器的文本上限（最旧的内容被丢弃）。 |

暂停某个项目的记录而无需编辑配置：创建 `.recall/.capture-paused`。删除它即恢复。

## 安全性

Recall 不发起网络调用，不使用 API 密钥，不加载第三方模型。摘要器是本地 Python；钩子仅使用标准库（`numpy` 为可选加速器）。它读取你的会话记录，仅写入 `output_dir`。具体而言：

- **永远不需要凭证。** 插件中没有任何 API 密钥、认证、`ANTHROPIC_*` 或 HTTP 的引用。如果 `claude` 本身显示"Invalid API key"，那是 CLI 自身的认证问题——通常是过期的 `ANTHROPIC_API_KEY` 环境变量覆盖了你的订阅登录。执行 `unset ANTHROPIC_API_KEY`（或运行 `env -u ANTHROPIC_API_KEY claude …`）。这与 Recall 无关。
- **脱敏处理。** 写入前会进行尽力而为的脱敏，去除常见密钥模式（API 密钥、token、`.env` 赋值、PEM 密钥），因为 `context.md` / `history.md` 可能会被提交。这是尽力而为，不是保证——提交前请自行检查。
- **加固的 git 调用。** `git diff` / `log` 运行时禁用了 `core.fsmonitor`、`diff.external`、钩子和分页器，因此不受信任的克隆仓库无法利用其自身的 git 配置在 Recall 读取数据时执行代码。设置 `include_git: false` 可完全跳过 git。
- **受限写入。** `output_dir` 被强制保持在项目内部；项目自带的配置无法将写入重定向到绝对路径或 `../..`。
- **限定范围的会话记录。** Recall 仅读取当前项目（按工作目录匹配）的会话记录；它永远不会回退到其他项目的会话。
- **共享记忆的信任边界。** `context.md` 在会话开始时注入模型。如果你将 `.recall/` 作为共享团队记忆提交，请像对待任何其他共享输入一样对待它：团队成员（或拥有仓库写权限的恶意者）可能会构造 `context.md` 尝试提示注入。SessionStart 会对内容进行隔离并标记为不可信数据，Claude 在依赖前会先询问——但如果你不完全信任谁能写入仓库，请保持 `.recall/` 被 gitignore（这是默认行为）。

## 提交还是忽略？

两种都可以。提交用于共享团队记忆，或 gitignore 用于个人记忆（`.gitignore` 默认忽略它——取消注释即可提交）。

## 安装

从市场安装（此仓库本身即市场）：

```
/plugin marketplace add raiyanyahya/recall
/plugin install recall@recall
```
