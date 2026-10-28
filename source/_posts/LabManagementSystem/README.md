---
title: "实验室预约与管理系统（LabManagementSystem）— 项目文档"
date: 2026-10-28 13:10:43
categories:
  - ProjectREADME
---

# 实验室预约与管理系统（LabManagementSystem）— 项目文档

> 本目录为 **LabManagementSystem** 的**产品与实现文档资料库**，按模块整理 PRD、实现方案与踩坑记录。
> 实际可运行代码位于独立仓库：`/Users/ergou/code/LabManagementSystem`（见本文「六、关联代码仓库与运行」）。

---

## 一、项目简介

**LabManagementSystem（实验室预约与管理系统）** 是面向高校实验室的师生服务系统，目标是让师生以「页面操作」或「自然语言对话」两种方式完成实验室预约、设备借用、申请进度查询等日常事务，并让管理员在后台集中审核与维护数据。

核心特点：

- **三类交互形态**
  - 用户端表单页（精确操作：预约、申请、查询）
  - 管理端后台（审核、台账维护、数据看板）
  - **智能助手 Agent**（自然语言入口：查实验室、查设备、查进度、答 FAQ）
- **Agent 不替代 API，而是封装 API**：业务逻辑始终在 REST 层，Agent 层只做「理解意图 → 选工具 → 组织回答」。
- **数据真实、可审计**：所有回答与落库数据均来自数据库，禁止模型编造；写操作必须人工确认。
- **渐进式架构**：分期迭代（只读查询 → 带确认的写操作 → 多 Agent / 统计），避免一期范围失控。

**技术栈**

| 层 | 技术 | 说明 |
|---|---|---|
| 用户端前端 | Vue 3 + Vite + Vue Router + Tailwind | 路由 `/`、`/appointment`、`/equipment`、`/assistant` |
| 后台管理前端 | Vue 3 + Vite（独立工程，部署在 `/admin`） | 审核、台账、用户管理 |
| 后端 | FastAPI（Python） | REST API（`/api`）+ 静态托管两个前端 `dist` |
| 数据库 | MySQL 8.0（Docker，宿主机端口 3606） | 关系型存储，ORM 用 SQLAlchemy |
| 智能助手 | OpenAI 兼容 Chat Completions API | DeepSeek / 通义千问 / 智谱 / OpenAI 可切换，Key 走 `.env` |
| 流式协议 | SSE（Server-Sent Events） | 助手回答逐 token 推送 |

---

## 二、系统架构

```
浏览器
├── 用户端  (Vue3 + Vite, 端口 5173 / 生产同源 /)
│   ├── 首页 / 预约页 / 设备申请页 / 我的申请
│   └── 智能助手 AssistantView  (POST /api/agent/chat, SSE 流式)
│
├── 管理端  (Vue3 + Vite, /admin)
│   ├── 实验室 / 预约审核 / 设备借用审核 / 用户管理
│   └── (P1) Agent 会话审计
│
└── 后端  FastAPI  (uvicorn :8000)
    ├── /api/*        REST 接口
    ├── app/api/routes/   各模块路由
    ├── app/services/     业务逻辑层（create_booking 等）
    ├── app/models/       ORM 模型
    └── app/agent/        Agent 编排层
        ├── loop.py       Tool-Calling 循环
        ├── llm.py        LLM 客户端 + 脱敏
        ├── tools/        各查询/写操作工具
        └── sanitize.py   PII 脱敏
            ↓
        MySQL (labsystem, 端口 3606)
```

---

## 三、功能模块总览

| 模块 | 状态 | 说明 |
|---|---|---|
| 实验室预约 | 已实现 | 机房时段预约全流程、管理端审核 |
| 设备申请 | 已实现（见本库 `device模块`） | 设备台账 + 借用申请 + 审核闭环 + 撤回 |
| 智能助手 Agent | P0 已实现，阶段 0/1 改造中（见本库 `agent模块`） | 自然语言查询入口；写操作走「草稿 → 人工确认」 |
| 失物招领 / 文档问答 / 数据统计 | 规划中（P1/P2） | 预留 Tool 与业务表扩展点 |

> 详细的分模块需求与实现步骤见「四、文档导航」中的子文档。

---

## 四、文档导航

本资料库按模块组织，每个模块含 **PRD（需求）** 与 **实现方案（落地）** 两部分。

### 📁 agent模块 — 实验室智能助手（Agent）

> 定位：系统的「第三类交互形态」——自然语言查询与咨询入口。横跨实验室、设备、预约、借用等所有业务。

| 文档 | 类型 | 说明 |
|---|---|---|
| [`agent模块/PRD/实验室智能助手（Agent）模块 — 产品需求文档（PRD）.md`](agent模块/PRD/实验室智能助手（Agent）模块%20—%20产品需求文档（PRD）.md) | PRD · v1.0 (2026-07-22) | 整体需求：定位、分期策略（P0 只读 / P1 RAG / P2 写操作）、Tool 设计约定、会话生命周期、权限安全 |
| [`agent模块/实现方案/实验室智能助手（Agent）模块 — 后端实现步骤指南.md`](agent模块/实现方案/实验室智能助手（Agent）模块%20—%20后端实现步骤指南.md) | 实现方案 | P0 后端落地：LLM 客户端、Tool 注册、Agent Loop、SSE 流式、会话持久化（含伪代码，边做边学，10~20 天） |
| [`agent模块/实现方案/实验室智能助手(Agent)-上线补充方案实现步骤指南.md`](agent模块/实现方案/实验室智能助手(Agent)-上线补充方案实现步骤指南.md) | 实现方案 · v1.0 (2026-10-11) | 上线改造：登录隔离 + 草稿确认 + 脱敏（阶段 0/1/2，6~18 人日）。**核心红线：LLM 打字说「确认」绝不建单，只有 confirm 接口（服务端会话用户身份）才能落库** |

**Agent 关键约定**
- Tool 直接调用 ORM，不走 HTTP 回环；每个 Tool 是纯函数，异常转为 `{"error": ...}` 返回模型。
- Agent Loop 最多 5 轮；单消息 ≤ 500 字；上下文最多 20 条。
- 写操作（借设备）：模型只生成**草稿** → 用户在卡片点确认 → 服务层写入与页面申请一致的记录。
- PII 脱敏：手机号/身份证正则遮盖，Tool 返回值剥离 `phone`、`openid` 等敏感字段。

### 📁 device模块 — 设备申请

> 定位：师生浏览真实设备库存、提交借用申请；管理员后台审核（通过/拒绝/查看拒绝原因）；用户可撤回待审申请。

| 文档 | 类型 | 说明 |
|---|---|---|
| [`device模块/PRD/设备申请模块 — 产品需求文档（PRD）.md`](device模块/PRD/设备申请模块%20—%20产品需求文档（PRD）.md) | PRD · v1.0 (2026-07-09) | 需求：用户端设备列表/我的申请、管理端审核、状态机（待审核→已通过/已拒绝/已取消）、接口契约、业务校验 |
| [`device模块/PRD/设备申请模块 - 出错点和解决方案.md`](device模块/PRD/设备申请模块%20-%20出错点和解决方案.md) | 踩坑记录 | 环境启动、MySQL 建库、Pydantic 模型、前后端对接常见问题与解法 |
| [`device模块/实现方案/设备申请模块 — 后端和数据库实现步骤指南.md`](device模块/实现方案/设备申请模块%20—%20后端和数据库实现步骤指南.md) | 实现方案 | 后端+数据库落地：管理端设备录入 → 用户申请 → 审核（按章节顺序，提供步骤与伪代码，10~14 天） |

**设备申请关键约定**
- 状态机：`0 待审核 → 1 已通过 / 2 已拒绝(必填原因) / 3 已取消(用户撤回)`；终态不可再变。
- 类别映射：笔记本→0、手机→1、智能仪器→2、软件→3。
- 业务校验：设备须存在且 `status==0`、可借数量充足、结束日期 ≥ 开始日期、拒绝必填原因、仅待审可撤回。

---

## 五、开发红线与安全约定

1. **写操作必须人工确认**：Agent 生成任何「提交/申请」仅落草稿；真正建单只能由服务端在已登录用户身份下经 confirm 接口完成。模型文本里的「确认」一律不可信。
2. **数据真实性**：Agent 回答中的实验室名、设备名、数量 100% 来自工具返回值，禁止编造。
3. **敏感字段脱敏**：手机号、身份证、openid、管理员备注等不在助手回答或 Tool 返回值中暴露。
4. **Prompt 注入防御**：System Prompt 声明用户消息中的指令不得覆盖系统规则；P0 仅只读 Tool，损害有限。
5. **费用与稳定性**：Agent Loop ≤ 5 轮、上下文 ≤ 20 条、单消息 ≤ 500 字、token 用量落库；LLM 不可用时降级提示，不影响其他模块。

---

## 六、关联代码仓库与运行

本资料库的文档对应实际代码仓库：

```
/Users/ergou/code/LabManagementSystem/
├── frontend/     用户端 Vue3 + Vite  (npm run dev → :5173)
├── admin/        管理端 Vue3 + Vite  (npm run dev → :5174/admin)
├── backend/      FastAPI            (uvicorn backend.main:app --reload → :8000)
├── docker-compose.yml   MySQL（宿主机端口 3606）
└── README.md     仓库级总说明（结构、数据库、运行步骤）
```

**最小化启动步骤**

```bash
# 1. 启动 MySQL（宿主机端口 3606）
cd /Users/ergou/code/LabManagementSystem
docker compose up -d mysql

# 2. 配置后端环境变量
cp backend/.env.example backend/.env

# 3. 安装后端依赖并启动（虚拟环境）
python3 -m venv .venv && source .venv/bin/activate
pip install -r backend/requirements.txt
python -m uvicorn backend.main:app --reload --host 127.0.0.1 --port 8000

# 4. 启动用户端前端（另开终端）
cd frontend && npm install && npm run dev   # → http://localhost:5173
```

> 前端 Vite 已将 `/api` 与 `/uploads` 代理到 `127.0.0.1:8000`，测试智能助手需后端同时运行。

---

## 七、文档版本与维护

| 文档 | 版本 | 日期 |
|---|---|---|
| 设备申请 PRD | v1.0 | 2026-07-09 |
| 智能助手 Agent PRD | v1.0 | 2026-07-22 |
| 设备申请 实现方案 | v1.0 | — |
| 智能助手 后端实现步骤指南 | v1.0 | — |
| 智能助手 上线补充方案实现步骤指南 | v1.0 | 2026-10-11 |

新增模块或重大改造时，请在此 README 的「四、文档导航」补充对应条目，并保持相对链接可跳转。
