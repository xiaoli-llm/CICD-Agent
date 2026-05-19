# CICD-Agent

> 基于 LangGraph + FastAPI 的多智能体 CI/CD 流水线系统

---

## 系统架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                        外部触发 (Webhook / Manual)                    │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     🎯 Planner Agent (:8000)                         │
│                     全局规划智能体                                    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌────────────┐ │
│  │ EventParser  │→│ EventValid  │→│ StageGen    │→│ PlanValid  │ │
│  └─────────────┘  └─────────────┘  └─────────────┘  └────────────┘ │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ HTTP POST
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    🔄 Orchestrator Agent (:8001)                     │
│                    全局编排智能体                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Stage Dispatcher                          │   │
│  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐   │   │
│  │  │Pull  │→│Comp  │→│Scan  │→│UT Gen│→│UT Exe│→│Deploy│   │   │
│  │  └──────┘ └──────┘ └──────┘ └──────┘ └──────┘ └──────┘   │   │
│  └─────────────────────────────────────────────────────────────┘   │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ HTTP POST
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    🧪 UT Generate Agent (:8004)                     │
│                    UT 生成智能体                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │CodeParser│→│TestPlan  │→│MockExpert│→│TestGen   │           │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘           │
│                                                     │               │
│                                                     ▼               │
│                                              ┌──────────┐           │
│                                              │Validator │←─retry─┐ │
│                                              └──────────┘        │ │
│                                                     │             │ │
│                                                     └─────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 智能体概览

| 智能体 | 端口 | 功能 | 状态 |
|--------|------|------|------|
| **Planner Agent** | 8000 | 解析触发事件，生成 7 阶段执行计划 | ✅ 已实现 |
| **Orchestrator Agent** | 8001 | 加载计划，循环调度各阶段执行 | ✅ 已实现 |
| **Code Pull Agent** | 8002 | 拉取代码仓库 | ✅ 已实现 |
| **Code Pull Check Agent** | 8003 | 校验代码拉取结果 | ✅ 已实现 |
| **UT Generate Agent** | 8004 | 自动生成 JUnit 5 单元测试 | ✅ 已实现 |

---

## 智能体详解

### 🎯 Planner Agent — 全局规划智能体

**端口**: 8000

**职责**: 接收外部触发事件（GitLab Webhook 或手动触发），解析并校验事件合法性，生成包含 7 个阶段的执行计划。

**核心流程**:
1. **Event Parser** — 解析 Webhook Push / Manual 触发事件
2. **Event Validator** — 校验事件合法性（分支、提交 ID 等）
3. **Stage Generator** — 根据配置生成阶段任务列表
4. **Plan Assembler** — 组装完整执行计划（含依赖图）
5. **Plan Validator** — 校验计划完整性

**API 接口**:
```bash
# 仅生成计划
POST /api/v1/plan/generate

# 触发完整流程（规划 → 编排）
POST /api/v1/cicd/trigger
```

**请求示例**:
```json
{
  "trigger_type": "webhook_push",
  "raw_event": {
    "ref": "refs/heads/main",
    "after": "abc123",
    "commits": [{"id": "abc123", "message": "feat: new feature"}]
  }
}
```

---

### 🔄 Orchestrator Agent — 全局编排智能体

**端口**: 8001

**职责**: 接收执行计划，按阶段顺序调度对应的智能体执行，管理流水线生命周期。

**7 个阶段**:
| 阶段 | 名称 | 说明 |
|------|------|------|
| 1 | `code_pull` | 拉取代码仓库 |
| 2 | `compile` | 编译项目 |
| 3 | `quality_scan` | 代码质量扫描 |
| 4 | `ut_generate` | 生成单元测试 |
| 5 | `ut_execute` | 执行单元测试 |
| 6 | `deploy` | 部署应用 |
| 7 | `release` | 发布版本 |

**错误处理策略**:
- `abort` — 终止流水线
- `skip` — 跳过当前阶段
- `retry` — 重试当前阶段

**API 接口**:
```bash
# 启动流水线
POST /api/v1/pipeline/start

# 查询流水线状态
GET /api/v1/pipeline/{pipeline_id}/status
```

---

### 🧪 UT Generate Agent — UT 生成智能体

**端口**: 8004

**职责**: 基于 Spoon 静态解析 + LLM，自动生成可编译的 JUnit 5 单元测试代码。

**6 节点子图**:
1. **Init Subgraph** — 初始化子图状态
2. **Code Parser** — 使用 Spoon 解析 Java 源码 AST
3. **Test Planner** — LLM 规划测试用例（边界值、异常路径）
4. **Mock Expert** — LLM 设计 Mock 策略（@Mockito / @SpringBootTest）
5. **Test Generator** — LLM 生成 JUnit 5 测试代码
6. **UT Validator** — 编译校验，失败时自动重试（最多 3 次）

**核心特性**:
- 纯本地静态解析，无幻觉
- 自动关联 SonarQube 风险方法
- 编译失败自动重试，逐次降低 LLM 温度
- 支持 Spoon + Python 双解析模式

**API 接口**:
```bash
POST /api/v1/ut/generate
```

**请求示例**:
```json
{
  "pipeline_id": "pipeline_001",
  "source_code_path": "/path/to/java/project",
  "compile_target_path": "/path/to/classes",
  "risk_method_list": [
    {"class_name": "UserService", "method_name": "createUser", "severity": "HIGH"}
  ]
}
```

---

## 共享工具框架

`shared/tool_registry/` 提供统一的工具调用机制：

```python
from tool_registry.base_tool import BaseTool

class MyTool(BaseTool):
    tool_key = "my_tool"
    required_params = ["project_path"]

    def _execute(self, params: dict) -> dict:
        return {"result": "ok"}
```

**特性**:
- 自动参数校验
- 命令沙箱（拦截高危命令）
- 超时控制
- 自动重试
- 标准化 ToolResult 返回

---

## 快速开始

### 环境要求

- Python 3.11+
- Java 11+（UT 生成需要 Spoon）

### 安装依赖

```bash
# 各智能体独立依赖
cd planner-agent && pip install -r requirements.txt
cd orchestrator-agent && pip install -r requirements.txt
cd ut-generate-agent && pip install -r requirements.txt
```

### 配置

在项目根目录创建 `.env` 文件：

```env
# LLM 配置
LLM_API_KEY=your_api_key
LLM_MODEL=gpt-4
LLM_BASE_URL=https://api.openai.com/v1

# 各智能体配置（可选）
PLANNER_ORCHESTRATOR_URL=http://localhost:8001
```

### 启动服务

```bash
# 启动单个智能体
cd planner-agent && python main.py
cd orchestrator-agent && python main.py
cd ut-generate-agent && python main.py
```

### 运行测试

```bash
# 单元测试
cd <agent-dir> && python -m pytest tests/ -v

# E2E 集成测试
python -m pytest test_e2e_flow.py -v
```

---

## 项目结构

```
CICD-Agent/
├── planner-agent/              # 全局规划智能体
│   ├── api/server.py           # FastAPI 路由
│   ├── config/settings.py      # 配置
│   ├── graph/                  # LangGraph 图定义
│   ├── models/                 # 数据模型
│   ├── nodes/                  # 图节点
│   └── tests/                  # 测试
│
├── orchestrator-agent/         # 全局编排智能体
│   ├── api/server.py           # FastAPI 路由
│   ├── config/settings.py      # 配置
│   ├── graph/                  # LangGraph 图定义
│   ├── models/                 # 数据模型
│   ├── nodes/                  # 图节点
│   ├── stage_agents/           # 阶段智能体实现
│   └── tests/                  # 测试
│
├── ut-generate-agent/          # UT 生成智能体
│   ├── api/server.py           # FastAPI 路由
│   ├── config/settings.py      # 配置
│   ├── graph/                  # LangGraph 图定义
│   ├── llm/                    # LLM 客户端
│   ├── models/                 # 数据模型
│   ├── nodes/                  # 图节点
│   ├── prompts/                # LLM 提示词
│   ├── rules/                  # 过滤规则
│   ├── tools/                  # 工具实现
│   └── tests/                  # 测试
│
├── code-pull-agent/            # 代码拉取智能体
├── code-pull-check-agent/      # 代码拉取校验智能体
├── shared/                     # 共享工具框架
│   ├── tool_registry/          # 工具注册中心
│   └── llm_config/             # LLM 共享配置
│
├── test_e2e_flow.py            # E2E 集成测试
└── CLAUDE.md                   # Claude Code 指南
```

---

## 技术栈

| 组件 | 技术 |
|------|------|
| 框架 | LangGraph + FastAPI |
| LLM | OpenAI API（兼容接口） |
| 代码解析 | Spoon (Java AST) |
| 数据校验 | Pydantic |
| 配置管理 | pydantic-settings |
| 测试 | pytest |

---

## License

MIT
