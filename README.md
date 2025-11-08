# v2 OpenAI 兼容服务（FastAPI + 前端）

本目录提供一个独立于 v1 的 Python 版本，实现 FastAPI 后端与纯静态前端，功能包括：
- 账号管理（SQLite 存储，支持登录/删除/刷新/自定义 other 字段，支持启用/禁用 enabled 开关）
- OpenAI Chat Completions 兼容接口（流式与非流式）
- 自动刷新令牌（401/403 时重试一次）
- URL 登录（设备授权，前端触发，最长等待5分钟自动创建账号并可选启用）
- 将客户端 messages 整理为 “{role}:\n{content}” 文本，替换模板中的占位内容后调用上游
- OpenAI Key 白名单授权：仅用于防止未授权访问；账号选择与 key 无关，始终从“启用”的账号中随机选择

主要文件：
- [v2/app.py](v2/app.py)
- [v2/replicate.py](v2/replicate.py)
- [v2/templates/streaming_request.json](v2/templates/streaming_request.json)
- [v2/frontend/index.html](v2/frontend/index.html)
- [v2/requirements.txt](v2/requirements.txt)
- [v2/.env.example](v2/.env.example)

数据库：运行时会在 v2 目录下创建 data.sqlite3（accounts 表内置 enabled 列，只从 enabled=1 的账号中选取）。

## 1. 安装依赖

建议使用虚拟环境：

```bash
python -m venv .venv
.venv\Scripts\pip install -r v2/requirements.txt
```

若在 Unix:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r v2/requirements.txt
```

## 2. 配置环境变量

复制示例文件生成 .env：

```bash
copy v2\.env.example v2\.env   # Windows
# 或
cp v2/.env.example v2/.env     # Unix
```

配置 OPENAI_KEYS（OpenAI 风格 API Key 白名单，仅用于授权，与账号无关）。使用逗号分隔：

示例：
```env
OPENAI_KEYS="key1,key2,key3"
```

提示：
- 若 OPENAI_KEYS 为空或未设置，则处于开发模式，不校验 Authorization。
- 该 Key 仅用于访问控制，不能也不会映射到任意 AWS 账号。

重要：
- 所有请求在通过授权后，会在“启用”的账号集合中随机选择一个账号执行业务逻辑。
- OPENAI_KEYS 校验失败返回 401；当白名单为空时不校验。
- 若没有任何启用账号，将返回 401。
前端与服务端通过 Authorization: Bearer {key} 进行授权校验（仅验证是否在白名单）；账号选择与 key 无关。

## 3. 启动服务

使用 uvicorn 指定 app 目录启动（无需将 v2 作为包安装）：

```bash
python -m uvicorn app:app --app-dir v2 --reload --port 8000
```

访问：
- 健康检查：http://localhost:8000/healthz
- 前端控制台：http://localhost:8000/

## 4. 账号管理

- 前端在 “账号管理” 面板支持：列表、创建、删除、刷新、快速编辑 label/accessToken、启用/禁用（enabled）
- 也可通过 REST API 操作（返回 JSON）

创建账号：

```bash
curl -X POST http://localhost:8000/v2/accounts ^
  -H "content-type: application/json" ^
  -d "{\"label\":\"main\",\"clientId\":\"...\",\"clientSecret\":\"...\",\"refreshToken\":\"...\",\"accessToken\":null,\"enabled\":true,\"other\":{\"note\":\"可选\"}}"
```

列表：

```bash
curl http://localhost:8000/v2/accounts
```

更新（切换启用状态）：

```bash
curl -X PATCH http://localhost:8000/v2/accounts/{account_id} ^
  -H "content-type: application/json" ^
  -d "{\"enabled\":false}"
```

刷新令牌：

```bash
curl -X POST http://localhost:8000/v2/accounts/{account_id}/refresh
```

删除：

```bash
curl -X DELETE http://localhost:8000/v2/accounts/{account_id}
```

无需在 .env 为账号做映射；只需在数据库创建并启用账号即可参与随机选择。

### URL 登录（设备授权，5分钟超时）

- 前端已在“账号管理”面板提供“开始登录”和“等待授权并创建账号”入口，打开验证链接完成登录后将自动创建账号（可选启用）。
- 也可直接调用以下 API：
  - POST /v2/auth/start
    - 请求体（可选）：
      - label: string（账号标签）
      - enabled: boolean（创建后是否启用，默认 true）
    - 返回：
      - authId: string
      - verificationUriComplete: string（浏览器打开该链接完成登录）
      - userCode: string
      - expiresIn: number（秒）
      - interval: number（建议轮询间隔，秒）
  - POST /v2/auth/claim/{authId}
    - 阻塞等待设备授权完成，最长 5 分钟
    - 成功返回：
      - { "status": "completed", "account": { 新建账号对象 } }
    - 超时返回 408，错误返回 502
  - GET /v2/auth/status/{authId}
    - 返回当前状态 { status, remaining, error, accountId }，remaining 为预计剩余秒数
- 流程建议：
  1. 调用 /v2/auth/start 获取 verificationUriComplete，并在新窗口打开该链接
  2. 用户在浏览器完成登录
  3. 调用 /v2/auth/claim/{authId} 等待创建账号（最多 5 分钟）；或轮询 /v2/auth/status/{authId} 查看状态

## 5. OpenAI 兼容接口

接口：POST /v1/chat/completions

请求体（示例，非流式）：

```json
{
  "model": "claude-sonnet-4",
  "stream": false,
  "messages": [
    {"role":"system","content":"你是一个乐于助人的助手"},
    {"role":"user","content":"你好，请讲一个简短的故事"}
  ]
}
```

授权与账号选择：
- 若配置了 OPENAI_KEYS，则 Authorization: Bearer {key} 必须在白名单中，否则 401。
- 若 OPENAI_KEYS 为空或未设置，开发模式下不校验 Authorization。
- 账号选择策略：在所有 enabled=1 的账号中随机选择；若无可用账号，返回 401。
- 被选账号缺少 accessToken 时，自动尝试刷新一次（成功后重试上游请求）。

非流式调用（以 curl 为例）：

```bash
curl -X POST http://localhost:8000/v1/chat/completions ^
  -H "content-type: application/json" ^
  -H "authorization: Bearer key1" ^
  -d "{\"model\":\"claude-sonnet-4\",\"stream\":false,\"messages\":[{\"role\":\"user\",\"content\":\"你好\"}]}"
```

流式（SSE）调用：

```bash
curl -N -X POST http://localhost:8000/v1/chat/completions ^
  -H "content-type: application/json" ^
  -H "authorization: Bearer key2" ^
  -d "{\"model\":\"claude-sonnet-4\",\"stream\":true,\"messages\":[{\"role\":\"user\",\"content\":\"讲一个笑话\"}]}"
```

响应格式严格遵循 OpenAI Chat Completions 标准：
- 非流式：返回一个 chat.completion 对象
- 流式：返回 chat.completion.chunk 的 SSE 片段，最后以 data: [DONE] 结束

## 6. 历史构造与请求复刻

- 服务将 messages 整理为 “{role}:\n{content}” 文本
- 替换模板 [v2/templates/streaming_request.json](v2/templates/streaming_request.json) 中的占位 “你好，你必须讲个故事”
- 然后按 v1 思路重放请求逻辑，但不依赖 v1 代码，具体实现见 [v2/replicate.py](v2/replicate.py)

## 7. 自动刷新令牌

- 请求上游出现 401/403 时，会尝试刷新一次后重试
- 也可在前端手动点击某账号的 “刷新Token” 按钮

## 8. 前端说明

- 页面路径：[v2/frontend/index.html](v2/frontend/index.html)，由后端根路由 “/” 提供
- 功能：管理账号（含启用开关） + 触发 Chat 请求（支持流式与非流式显示）
- 在页面顶部设置 API Base 与 Authorization（OpenAI Key）

## 9. 运行排错

- 导入失败：使用 --app-dir v2 方式启动 uvicorn
- 401/403：检查账号的 clientId/clientSecret/refreshToken 是否正确，或手动刷新，或确认账号 enabled=1
- 未选到账号：检查 OPENAI_KEYS 映射与账号启用状态；对于通配池 key:* 需保证至少有一个启用账号
- 无响应/超时：检查网络或上游服务可达性

## 10. 设计与来源

- 核心重放与事件流解析来自 v1 的思路，已抽取为 [v2/replicate.py](v2/replicate.py)
- 后端入口：[v2/app.py](v2/app.py)
- 模板请求：[v2/templates/streaming_request.json](v2/templates/streaming_request.json)

## 11. 许可证

仅供内部集成与测试使用。