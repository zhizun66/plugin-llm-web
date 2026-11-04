# AI助手（`llm_desktop` / `llm_web`）

**AI助手** 是 King Box 平台的双端独立进程插件（`stdio-jsonl` 协议，包含桌面端 `llm_desktop` 与云端多租户 Web 端 `llm_web`），用于统一管理兼容 OpenAI 协议的大语言模型配置、AI 员工提示词模板与跨渠道调用明细，并通过宿主 Broker 向其他业务插件（如 Discord 等）提供标准的 `llm.v1` 大模型与翻译服务。

---

## 核心功能（Desktop 与 Web 双端对齐）

### 1. 大模型设置（`/`）
- **供应商预设**：内置 `deepseek`（`https://api.deepseek.com/v1`）、`openai`（`https://api.openai.com/v1`）、`kimi`（`https://api.moonshot.cn/v1`）与 `自定义` 四种选项；预设供应商地址只读防误改，选择 `自定义` 时可填写任意兼容 OpenAI 协议的网关地址。
- **拉取模型与手动输入**：支持一键请求远端 `GET /models` 拉取可用模型下拉列表，也支持手动输入模型 ID。
- **连通性测试**：一键发送测试短句验证接口与 APIKEY 可用性，实时返回模型回复内容与 Token 消耗明细（请求直连，不走业务代理池）。
- **多租户隔离（Web 端）**：各租户的大模型地址、APIKEY 与默认模型在 PostgreSQL `llm.llm_settings` 表中按 `tenant_id` 严格物理隔离。

### 2. AI 员工管理（`/employees`）
- **网格视图 / 列表视图切换**：支持卡片网格视图与紧凑列表视图自由切换，支持按员工名称、调用标识（`key`）或提示词内容实时搜索。
- **双内置翻译员工**：系统按租户自动初始化两个受保护内置员工——`翻译发送消息`（`key = "translate_send"`，便于后续结合业务定制外发话术提示词）与 `翻译接收消息`（`key = "translate_receive"`），固定排在列表前两位且不可删除，支持按需修改名称与提示词模板。
- **自定义提示词模板**：支持创建多个自定义 AI 员工（如客服话术润色、群聊自动回复、多语种摘要等），提示词中使用 `{{变量名}}` 定义动态占位符（两个内置翻译员工均支持 `{{source_lang}}`、`{{target_lang}}`、`{{text}}`）。
- **套餐配额联动**：实时展示并校验当前租户套餐下的 `AI 员工数量`（`employees`）配额使用进度。

### 3. 使用记录（`/usage`）
- **统计汇总卡片**：汇总展示总调用次数、成功调用、失败调用与累计消耗 Tokens（含输入 Prompt Tokens 与输出 Completion Tokens）。
- **多维筛选与排序**：支持按用途/渠道/模块/员工/素材关键词搜索，按渠道（Discord / 系统）与状态（成功 / 失败）过滤，支持创建时间正序/倒序排列与服务端分页（50 / 100 条）。
- **调用详情穿透**：点击任意调用记录行可查看完整的原始素材、渲染后最终 Prompt、模型响应正文或错误详情。

### 4. 接口调试（`/api-debug`，Web 端仅限超级管理员查看）
- **超管专属权限隔离**：在 `apps/web/manifest.json` 中通过 `roles: ["superadmin"]` 声明并配合前后端双重角色校验，仅平台超级管理员可见与可访问。
- **在线 HTTP 接口调试器（类似 Postman）**：内置外部第三方 Open API（`llm.run`、`llm.ask`、`employee.list`、`call.list`、安全边界拦截验证）与内部网关预设模板，支持动态编辑请求头/JSON Body、一键发送、实时展示 HTTP 状态码/业务码/耗时/响应体积/`X-Trace-Id` 并支持复制脱敏 cURL 命令。

### 5. 角色读写权限与操作审计日志（`/logs`）
- **角色读写分层**：`admin`（租户管理员）与 `superadmin`（平台超级管理员，进入租户代管后）拥有全部页面的读写与连通性测试权限；`agent`（普通员工）在管理页面为**只读视图**（APIKEY 自动脱敏，不可修改配置、不可拉取/测试模型、不可增删改 AI 员工），但可正常调用 `llm.run` / `llm.ask` 执行 AI 员工翻译与业务补全。
- **操作审计日志**：记录大模型配置保存、连通性测试、AI 员工增删改等审计事件（APIKEY 全程脱敏，Desktop 端写入私有 `plugin.db` 的 `operation_logs`，Web 端同步写入 `llm.operation_logs` 与宿主 `audit.log` 审计中心）。

---

## 跨插件服务（`llm.v1`）

`llm_desktop` 与 `llm_web` 启动后均在各自宿主 Broker 注册 `llm`（`v1`）服务，同端其他业务插件可通过 `packages/backend/public-libs/llm.NewBrokerClient` 或 `service.invoke` 直接调用：

- `llm.run`：按指定 `employee_key`（如 `translate_send` / `translate_receive`）与 `vars` 变量字典渲染提示词并执行大模型补全，自动写入「使用记录」。
- `llm.ask`：直接发送原始 `prompt` 执行大模型补全（如语种识别、临时问答），自动写入「使用记录」。
- 完整调用文档参见源码仓库 `docs/llm-service-usage.md`。

---

## 安装与更新

1. **插件商店在线安装**：在 King Box Desktop 或 King Box Web 管理端打开「插件 -> 插件商店」，找到 **AI助手** 点击「安装」或「更新」。
2. **Release 离线包安装**：
   - Desktop 端：从 [`plugin-llm-desktop` Releases](https://github.com/zhizun66/plugin-llm-desktop/releases) 下载 `llm_desktop-v*-windows-amd64.zip` 导入安装。
   - Web 端：从 [`plugin-llm-web` Releases](https://github.com/zhizun66/plugin-llm-web/releases) 下载对应服务器架构的 `llm_web-v*-linux-amd64.zip` 或 `llm_web-v*-windows-amd64.zip` 导入安装。
