# 知序 · 学习规划助手

知序是一个基于 Vue 3 与 FastAPI 的个人学习规划应用。用户填写目标、当前水平、每周投入和目标日期后，后端通过四个协作 Agent 整理学习目标、搜索资料、生成学习策略，并编排近期任务与远期阶段。计划和任务状态保存在 SQLite 中，可在后续规划时接续未完成任务。

## 功能概览

- **目标解析**：将目标整理为明确描述、规划假设和阶段里程碑，并估算计划周数。
- **资料书架**：可选调用 MCP 搜索工具获取资料；未配置或搜索失败时，提供 Google 和哔哩哔哩搜索入口。
- **经验整合**：结合学习背景、历史未完成任务、资料和每周可用时间给出策略、风险提示与已采用的上下文。
- **近期任务**：生成未来约两周的学习任务，包含日期、时长、优先级和可执行说明；前端支持标记完成或重新打开。
- **远期路线**：将两周之后到目标日期的安排整理为按周划分的阶段目标和预期成果。
- **接续与独立规划**：接续模式沿用当前学习空间；独立模式使用新的学习空间，不读取其他空间的历史任务。
- **任务迁移**：生成新计划时读取该学习空间所有未完成任务，沿用任务 ID 和状态、调整日期，并将优先级数值向 P1 调整。已完成任务不会迁移。
- **本地降级运行**：模型服务未配置或调用失败时，目标拆解、学习策略和任务内容会使用本地规则生成；MCP 不可用时会退回搜索链接。

## 工作流程

生成计划时，后端按以下顺序运行并在响应中返回 Agent 记录：

1. **目标解析 Agent**：生成澄清后的目标、假设、里程碑和预计周数。
2. **资料搜索 Agent**：使用“主题/目标 + 当前水平 + 学习课程/文档/练习”构造查询，并尝试调用 MCP。
3. **经验整合 Agent**：结合目标、资料、补充背景和历史未完成任务形成策略与风险提示。
4. **规划协调 Agent**：将未完成任务安排到近期日期，再补充新任务，并为第 3 周起的周期生成远期阶段。
5. **保存计划**：计划与近期任务写入 SQLite；更新任务状态时同时更新任务记录和对应计划内容。

配置了模型 Key 后，后端会请求 OpenAI Chat Completions 兼容接口，并要求模型返回 JSON。当前使用模型的部分包括目标解析、经验整合和近期任务建议；MCP 搜索由单独配置控制。

## 技术结构

```text
backend/
	app/
		agents/       目标解析、资料搜索、经验整合、规划协调与流程编排
		api/          FastAPI 路由
		models/       Pydantic 请求与响应模型
		services/     SQLite 上下文记忆、模型客户端、MCP 搜索
	data/           SQLite 数据目录（运行后创建）
	tests/          后端单元测试
	.env.example    环境变量示例
	run.py          本地 API 启动入口
frontend/
	src/App.vue     Vue 单页界面
	src/services/   后端 API 请求
	src/types.ts    TypeScript API 数据类型
```

后端通过 Pydantic 模型校验 API 输入；前端 `src/types.ts` 对应相同 JSON 字段。Vite 开发服务器监听 5173 端口，并将 `/api` 请求代理至 `http://localhost:8000`。

## 环境要求

- Python 3.10 或更高版本
- Node.js 与 npm
- （可选）OpenAI 兼容模型 API
- （可选）可通过 stdio 启动并提供搜索工具的 MCP 服务

## 本地运行

在仓库根目录打开两个终端。先启动后端。

### Windows PowerShell

```powershell
cd backend
py -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
Copy-Item .env.example .env
python run.py
```

### macOS / Linux

```bash
cd backend
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
python run.py
```

然后在第二个终端启动前端：

```bash
cd frontend
npm install
npm run dev
```

访问地址：

- 前端：<http://localhost:5173>
- API 文档：<http://localhost:8000/docs>
- 健康检查：<http://localhost:8000/api/health>

默认不需要模型 API Key 或 MCP 服务即可使用。若不需要配置外部服务，也可以跳过复制 `.env.example` 这一步；应用会使用默认配置。

## 配置

后端从 `backend/.env` 读取设置。变量名大小写不敏感；未知变量会被忽略。

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `APP_NAME` | `学习规划助手` | FastAPI 应用名称。 |
| `HOST` | `0.0.0.0` | 后端监听地址。 |
| `PORT` | `8000` | 后端监听端口。 |
| `LLM_API_KEY` | 空 | OpenAI 兼容 API 的密钥；留空时不调用模型。 |
| `LLM_BASE_URL` | `https://api.openai.com/v1` | API 基础地址，客户端会在其后追加 `/chat/completions`。 |
| `LLM_MODEL` | `gpt-4o-mini` | 请求使用的模型名称。 |
| `MCP_COMMAND` | 空 | MCP stdio 服务的启动命令；留空时显示搜索入口。 |
| `MCP_ARGS` | `[]` | 传给启动命令的参数，必须是 JSON 字符串数组。 |
| `MCP_TOOL_NAME` | `search` | MCP 服务中要调用的工具名称。 |
| `DATABASE_PATH` | `data/study_planner.db` | SQLite 文件路径；相对路径以 `backend/` 为基准。 |

启用模型时，至少设置 `LLM_API_KEY`；按服务商要求调整 `LLM_BASE_URL` 和 `LLM_MODEL`。模型请求超时或响应不符合预期时，相关 Agent 会使用本地规则作为后备结果。

## MCP 搜索配置

MCP 服务通过 Python MCP SDK 的 stdio transport 启动。工具需要接受名为 `query` 的字符串参数，调用参数形如：

```json
{"query":"Python 初学者 学习课程 文档 练习"}
```

例如，假设 MCP 服务可以由 `npx` 启动，`.env` 中的配置形式如下；命令、参数和工具名称需要替换成服务实际支持的值：

```dotenv
MCP_COMMAND=npx
MCP_ARGS=["-y","your-mcp-search-package"]
MCP_TOOL_NAME=search
```

程序会检查返回内容中的文本和 URL，并最多整理 8 条资料。未配置服务、调用异常或返回内容无法识别时，应用会返回搜索入口，而不会阻断计划生成。若现有 MCP 工具使用其他参数结构，可在 MCP 服务端提供接受 `query` 的适配工具。

## API

### `POST /api/plans/generate`

创建并保存学习计划。常用请求字段如下：

| 字段 | 类型 | 默认值/限制 |
| --- | --- | --- |
| `learner_id` | 字符串 | `default-learner`；长度 1–80。用于隔离学习空间。 |
| `goal` | 字符串 | 必填；长度 4–500。 |
| `subject` | 字符串 | 空字符串；最多 100 字符。 |
| `current_level` | 字符串 | `初学者`；最多 100 字符。 |
| `weekly_hours` | 数字 | `6`；大于 0 且不超过 60。 |
| `study_days_per_week` | 整数 | `5`；范围 1–7。 |
| `session_minutes` | 整数 | `60`；范围 15–240。 |
| `deadline` | 日期 | 默认今天起 60 天；格式为 `YYYY-MM-DD`，不能早于今天。 |
| `context_notes` | 字符串 | 空字符串；最多 2000 字符。 |

请求示例：

```json
{
	"learner_id": "my-learning-space",
	"goal": "三个月内掌握 Python 数据分析并完成一个作品集项目",
	"subject": "Python 数据分析",
	"current_level": "初学者",
	"weekly_hours": 6,
	"study_days_per_week": 5,
	"session_minutes": 60,
	"deadline": "2026-12-01",
	"context_notes": "有基础编程经验，希望周末安排更多练习"
}
```

### `GET /api/plans/current?learner_id=...`

返回指定学习空间最近保存的计划；未找到时返回 `null`。省略 `learner_id` 时后端使用 `default-learner`。

### `PATCH /api/tasks/{task_id}`

更新近期任务状态。请求体中的 `status` 可取 `todo`、`in_progress` 或 `completed`。任务不存在时返回 404。当前前端的任务按钮在“完成”和“待办”之间切换。

### `GET /api/health`

服务可用时返回 `{"status":"healthy","service":"study-planner"}`。

## 数据与学习空间

默认数据库位于 `backend/data/study_planner.db`，首次使用时自动创建目录和表。设置绝对路径可将数据库放在其他位置；相对路径按 `backend/` 解析。计划查询按 `learner_id` 隔离，前端会在浏览器 `localStorage` 保存当前学习空间 ID。

接续规划会从该学习空间读取状态不是 `completed` 的任务，其中包括 `todo` 和 `in_progress`。迁移后的任务保留原 ID、状态和已有来源日期，并写入新计划；标记完成后的任务不会在后续计划中迁移。独立新建会创建新的学习空间 ID，因此不会读取原空间的任务。

## 测试与构建

运行后端测试：

```bash
cd backend
python -m unittest discover -s tests
```

构建前端并执行 TypeScript 检查：

```bash
cd frontend
npm run build
```
