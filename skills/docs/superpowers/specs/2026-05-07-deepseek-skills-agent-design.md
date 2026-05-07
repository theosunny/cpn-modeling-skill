# DeepSeek Skills Agent — Design Spec

**Date:** 2026-05-07  
**Status:** Approved  
**Location:** `/Users/habitat/ai_project/`

---

## Overview

A standalone agent powered by DeepSeek (official API) that loads OpenClaw-compatible skills, orchestrates their execution via LangGraph, and exposes both a CLI and HTTP API interface. Supports Hermes-style three-layer memory with summarization and upgradeable semantic memory.

---

## Architecture

```
ai_agent/
├── agent/
│   ├── core.py          # LangGraph agent 主循环
│   ├── graph.py         # 状态图定义
│   └── nodes.py         # 各节点逻辑（think/act/observe）
├── skills/
│   ├── loader.py        # 扫描并加载 SKILL.md，注册为 tools
│   └── executor.py      # skill 工作流执行引擎（子 agent）
├── memory/
│   ├── working.py       # 第一层：会话内上下文窗口
│   ├── episodic.py      # 第二层：对话汇总摘要（持久化）
│   └── semantic.py      # 第三层：长期语义记忆（merge 更新）
├── api/
│   ├── server.py        # FastAPI HTTP + WebSocket
│   └── routes.py        # REST 路由定义
├── cli/
│   └── main.py          # Rich 终端界面
├── config.py            # 配置管理（DeepSeek key、路径）
└── main.py              # 统一入口（chat / serve 命令）
```

---

## Components

### 1. Agent Core (LangGraph)

- **State graph** with nodes: `think` → `act` → `observe` → `think` (ReAct loop)
- DeepSeek接入：使用 `langchain-openai` 的 `ChatOpenAI`，base_url 指向 `https://api.deepseek.com/v1`
- Tool calling 驱动 skill 选择与执行
- 支持流式输出（streaming tokens）

### 2. Skills Loader

- 扫描目录（默认 `~/.openclaw/workspace/skills/`，可配置）
- 解析每个 skill 目录下的 `SKILL.md` frontmatter（name、description、tags）
- 将每个 skill 注册为 `StructuredTool`，description 用于 LLM tool selection
- 支持 `/skills/reload` 热重载，无需重启服务

### 3. Skills Executor

- 调用 skill 时，读取完整 `SKILL.md` 内容
- 构建子 agent（LangGraph sub-graph），按工作流步骤逐步执行
- 从 `~/.openclaw/openclaw.json` 自动注入对应 skill 的 config
- 步骤间状态传递，支持外部 API 调用（HTTP/Bash）
- 进度通过 WebSocket 实时推送给用户
- 步骤失败时记录上下文，返回用户并支持重试

### 4. Three-Layer Memory (Hermes Style)

| 层级 | 名称 | 存储 | 生命周期 | 更新方式 |
|------|------|------|----------|----------|
| L1 | Working Memory | 内存（消息列表） | 单次会话 | 自动滚动（超 token 阈值截断） |
| L2 | Episodic Memory | `memory/episodes/YYYY-MM-DD.md` | 持久化 | 会话结束或 token 阈值触发，DeepSeek 自动汇总 |
| L3 | Semantic Memory | `memory/semantic.json` | 持久化 | 检测到新事实/偏好时 merge 更新（不覆盖，合并） |

**记忆更新触发点：**
1. 会话结束 → L2 汇总写入 Episodic
2. LLM 检测到用户偏好/事实变化 → L3 merge 写入 Semantic
3. 手动命令 `memory update` / `POST /memory/update`

**Semantic Memory merge 规则：**
- 新信息与旧记忆由 DeepSeek 合并，保留历史，标注更新时间
- 冲突时新信息优先，旧值保留在 `_history` 字段

### 5. HTTP API (FastAPI)

```
POST /chat                    # 发送消息，SSE 流式响应
GET  /chat/{session_id}       # 获取会话历史
POST /memory/update           # 手动触发记忆汇总升级
GET  /memory                  # 查看三层记忆状态
GET  /skills                  # 列出已加载 skills
POST /skills/reload           # 热重载 skills 目录
WS   /ws/{session_id}         # WebSocket 实时进度推送
```

### 6. CLI (Rich)

```bash
python main.py chat                        # 多轮对话
python main.py chat --session-id <id>      # 恢复历史会话
python main.py skills list                 # 列出可用 skills
python main.py memory show                 # 查看记忆
python main.py serve [--host] [--port]     # 启动 API server
```

终端界面：Rich `Live` + `Panel` 渲染对话，skill 执行进度用 `Progress` 条展示。

---

## Data Flow

```
用户输入
  → Working Memory 注入历史上下文
  → Episodic + Semantic Memory 检索相关记忆
  → DeepSeek LLM (think node)
  → Tool call 选择 skill
  → Skills Executor (sub-agent, 按 SKILL.md 步骤执行)
  → 进度推送 (WebSocket / CLI progress bar)
  → 结果返回用户
  → 记忆更新检测 → L2/L3 异步写入
```

---

## Error Handling

- **DeepSeek API 失败**：指数退避重试 3 次（1s, 2s, 4s）
- **Skill 步骤失败**：记录错误上下文，告知用户，询问是否重试
- **记忆写入失败**：降级到内存，会话结束时再次尝试持久化

---

## Configuration

`config.py` 读取以下环境变量或 `.env` 文件：

```
DEEPSEEK_API_KEY=sk-xxx
DEEPSEEK_MODEL=deepseek-chat
SKILLS_DIR=~/.openclaw/workspace/skills
OPENCLAW_CONFIG=~/.openclaw/openclaw.json
MEMORY_DIR=./memory
API_HOST=0.0.0.0
API_PORT=8000
```

---

## Testing Strategy

- **Unit tests**：memory 层（working/episodic/semantic 各自独立测试）、skills loader 解析逻辑
- **Integration tests**：agent 完整 ReAct 循环（mock DeepSeek API）、skill executor 步骤执行
- **E2E tests**：CLI 多轮对话场景、HTTP API `/chat` 端点
- TDD 顺序：memory → skills loader → agent core → API → CLI

---

## Dependencies

```
langchain-openai
langgraph
fastapi
uvicorn
websockets
rich
python-dotenv
pydantic
pytest
pytest-asyncio
httpx
```
