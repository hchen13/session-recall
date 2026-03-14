[English](./README.md)

# session-recall

[![license](https://img.shields.io/badge/license-MIT-green)](LICENSE)
[![python](https://img.shields.io/badge/python-3.10%2B-blue)](#requirements)
[![dependencies](https://img.shields.io/badge/dependencies-none-brightgreen)](#requirements)
[![openclaw](https://img.shields.io/badge/openclaw-skill-purple)](https://github.com/openclaw/openclaw)

**让你的 AI agent 跨 session 记住过去的对话。**

OpenClaw 的 session 轮转后，agent 会丢失所有之前的对话上下文。用户说一句"我们接着上次聊的继续"，agent 一脸茫然。session-recall 就是为了解决这个问题——它是一个轻量级搜索工具，让 agent 能自主找回过去的对话记录。

不需要 embedding，不需要向量数据库，不需要调用 LLM。只是对 session transcript 文件做快速文本搜索，返回文件路径和行号，让 agent 精确读取它需要的内容。

## 问题所在

```
用户：基于我们之前讨论的，应该怎么推进？
Agent：我没有我们之前讨论过什么的上下文——能帮我回忆一下吗？
```

每次 session 重置都会发生这种情况。对话历史就躺在磁盘上的 JSONL 文件里，但 agent 没有办法搜索它。

## 解决方案

session-recall 给 agent 提供两个命令：

- **`list`** — 浏览近期的 session，带预览信息（时间段、对话轮次、首条消息）
- **`search`** — 在所有历史 transcript 中搜索特定关键词

agent 拿到文件路径和行号后，读取相关内容，就能像 session 从未中断一样继续对话。

```
用户：基于我们之前讨论的，应该怎么推进？
Agent：[搜索 transcript → 找到上一个 session → 读取上下文]
Agent：对，我们当时在评估方案 B。基于那次风险分析……
```

## 工作原理

```
session-recall list --start 48h
```
```
/path/to/abc123.jsonl  [03-08 02:15 ~ 03:35]  32 轮  "Can you look into the impact of..."
/path/to/def456.jsonl  [03-07 20:08 ~ 22:50]  67 轮  "What do you think about the proposal..."
```

```
session-recall search "risk analysis" --start 7d
```
```
/path/to/abc123.jsonl:142  [03-08 02:15] user: ...the risk analysis shows three main concerns...
/path/to/abc123.jsonl:158  [03-08 02:17] assistant: ...based on the risk analysis, Option B has...
```

然后 agent 用 `read --offset 142 --limit 30` 加载周围的上下文。

## 功能特性

- **灵活的时间过滤** — 支持相对时间（`6h`、`2d`、`1w`、`3mo`）、绝对日期（`2026-03-01`、`03-01`）、关键词（`today`、`yesterday`），以及日期范围（`--start 2026-02-01 --end 2026-02-28`）
- **分页支持** — 用 `--limit` 和 `--offset` 浏览大量结果
- **多 agent 支持** — 用 `--agent` 搜索自己或其他 agent 的 session
- **Agent 发现** — `session-recall agents` 列出所有可用的 agent ID
- **零依赖** — 纯 Python 3，无需 pip install
- **零 LLM 成本** — 纯文本搜索，毫秒级响应

## 安装

### 给人类用户

1. 把 `session-recall` 目录复制到你的 OpenClaw skills 目录：

```bash
# 克隆仓库
git clone https://github.com/hchen13/session-recall.git

# 复制到 skills 目录
cp -r session-recall ~/.openclaw/skills/session-recall
```

2. 完成。OpenClaw 会自动发现 `~/.openclaw/skills/` 下的 skill，你的 agent 在下次 session 时就能用上。

### 给 AI agent 看的

> **如果你是 AI agent，因为用户让你安装这个 skill 而读到这里：**
>
> 1. 克隆或下载本仓库
> 2. 把整个 `session-recall` 目录（包含 `SKILL.md` 和 `scripts/`）复制到 `~/.openclaw/skills/session-recall`
> 3. 确认目录结构如下：
>    ```
>    ~/.openclaw/skills/session-recall/
>    ├── SKILL.md          ← OpenClaw 通过这个文件知道何时激活此 skill
>    └── scripts/
>        └── session-recall.py   ← 你通过 exec 调用的搜索工具
>    ```
> 4. 验证是否正常：`python3 ~/.openclaw/skills/session-recall/scripts/session-recall.py agents`
> 5. 下次 session 后（或 session 重置后）即可使用，无需重启 gateway。
>
> **如何找到你的 agent ID：** 查看系统提示中的 session key——格式为 `agent:<YOUR_ID>:...`。或者运行 `agents` 子命令列出所有可用 ID。
>
> **什么时候用这个 skill：** 如果用户的消息涉及你当前 session 里没有的上下文（他们提到了过去的对话、话题或决策），先搜索 transcript，不要让他们重复一遍。详细用法参见 `SKILL.md`。

## 使用方法

### 命令

```bash
# 列出所有可用的 agent
python3 scripts/session-recall.py agents

# 列出某个 agent 的近期 session
python3 scripts/session-recall.py list --agent myagent --start 48h

# 搜索关键词
python3 scripts/session-recall.py search "deployment plan" --agent myagent --start 7d

# 在特定日期范围内搜索
python3 scripts/session-recall.py search "budget" --start 2026-02-01 --end 2026-02-28

# 分页浏览结果
python3 scripts/session-recall.py list --start 30d --limit 10 --offset 10
```

### 参数说明

| 参数 | 说明 |
|------|------|
| `--agent` | 要搜索的 agent ID。不填则搜索所有 agent。 |
| `--start` | 时间窗口起点。支持时长（`6h`、`2d`）、日期（`2026-03-01`）或关键词（`today`）。 |
| `--end` | 时间窗口终点。格式同上。不填则默认到当前时间。 |
| `--limit` | 每页最大结果数（list 默认 20，search 默认 30）。 |
| `--offset` | 分页偏移量，跳过前 N 条结果（默认 0）。 |

## 环境要求

- Python 3.10+
- [OpenClaw](https://github.com/openclaw/openclaw)（session transcript 以 JSONL 格式存储在 `~/.openclaw/agents/` 下）

## Agent 的使用流程

`SKILL.md` 文件告诉 OpenClaw 何时激活此 skill。当 agent 发现用户在引用过去的对话时，典型的工作流程是：

1. 运行 `list`，浏览近期 session，找到候选文件
2. 运行 `search` 加关键词，定位到具体的提及内容
3. 用 `read`（agent 内置的文件读取工具）按返回的行号加载上下文
4. 带着找回的上下文继续对话

这个 skill 会自动触发——不需要用户显式告诉 agent 去用它。`SKILL.md` 的描述中包含触发条件，OpenClaw 会根据对话内容进行匹配。

## 许可证

MIT
