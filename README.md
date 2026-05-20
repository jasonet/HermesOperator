# Hermes + Operator 架构设计方案

> 公共服务器部署 Hermes Agent + 独立的 Operator 审核控制系统

---

## 一、设计目标

1. **Hermes 核心可自动升级** — `hermes update` 无感知更新，不受 Operator 代码影响
2. **Operator 完全独立** — 独立的登录认证、Web 入口、会话区
3. **资源共享** — Operator 共用 Hermes 的插件、技能、模型 providers
4. **会话隔离可管理** — Operator 会话在 state.db 中有独立标识，可审计
5. **升级后改动可恢复** — Operator 代码存放在 Hermes 更新不会触碰的路径

---

## 二、架构总览

```
┌─────────────────────────────────────────────────────────────────┐
│                        公共服务器                                │
│                                                                  │
│  ┌──────────────────────────────┐  ┌──────────────────────────┐ │
│  │     Hermes Core (上游)       │  │   Operator System (扩展) │ │
│  │  ~/.hermes/hermes-agent/     │  │                           │ │
│  │                              │  │  入口: operator.example.com│ │
│  │  • hermes update (自动)     │  │  端口: 8643               │ │
│  │  • CLI / Gateway / TUI      │  │                           │ │
│  │  • API Server :8642         │  │  ┌─────────────────────┐  │ │
│  │  • state.db (共享)          │  │  │ Operator FastAPI App│  │ │
│  │  • 所有 plugins/skills     │  │  │ • JWT 认证           │  │ │
│  │  • 所有 model providers    │  │  │ • 角色鉴权           │  │ │
│  │                              │  │  │ • Web Dashboard     │  │ │
│  │  更新方式: git pull upstream │  │  │ • 会话管理 UI       │  │ │
│  │  永不直接改动核心代码       │  │  └─────────┬───────────┘  │ │
│  └──────────────┬───────────────┘  │            │              │ │
│                 │                  │  ┌─────────▼───────────┐  │ │
│                 │    Python API    │  │ Operator Plugin     │  │ │
│                 │   (AIAgent 类)   │  │ ~/.hermes/plugins/  │  │ │
│                 └──────────────────┤  │   operator/         │  │ │
│                                    │  │                     │  │ │
│                                    │  │ • 扩展指令审核      │  │ │
│                                    │  │ • 离散化审计日志    │  │ │
│                                    │  │ • 自定义工具注册    │  │ │
│                                    │  └─────────────────────┘  │ │
│                                    │                           │ │
│                                    │  会话标记: source='operator'│
│                                    │  HERMES_HOME 指向独立配置 │ │
│                                    └──────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### 文件布局

```
~/.hermes/                          # 主 Hermes 配置
├── hermes-agent/                   # 核心代码 (git 管理，自动更新)
│   └── ...                         # 永不直接修改这里的文件
├── config.yaml                     # 主配置
├── .env                            # 主密钥
├── state.db                        # 共享会话数据库
│
├── profiles/operator/              # Operator 独立配置
│   ├── config.yaml                 # Operator 专用配置
│   ├── .env                        # Operator 密钥
│   ├── sessions/                   # Operator 本地会话日志
│   ├── skills/                     # 可独立安装额外技能
│   └── logs/
│
├── plugins/operator/               # Operator 插件 (升级不触碰)
│   ├── plugin.yaml                 # 插件元数据
│   ├── __init__.py                 # 插件入口 + 工具注册
│   ├── auth.py                     # JWT 认证模块
│   ├── server.py                   # FastAPI Web 服务
│   ├── operator_agent.py           # Operator 专用的 AIAgent 封装
│   ├── audit.py                    # 审核/离散化模块
│   └── web/                        # 前端静态资源
│       └── dashboard.html
│
└── scripts/operator/               # 运维脚本
    ├── start.sh                    # 启动脚本
    └── auto_update.sh              # 自动升级脚本
```

---

## 三、升级兼容策略

### Hermes 升级流程

`hermes update` 内部执行:

```
1. git stash push --include-untracked  # 暂存本地改动
2. git pull upstream main              # 拉取最新代码
3. uv pip install -e .                 # 更新 Python 依赖
4. git stash apply                     # 恢复本地改动
5. 如有冲突 → git reset --hard HEAD   # 放弃恢复，保留 stash
```

### Operator 如何生存

| 操作 | 影响 Operator? | 原因 |
|------|:---:|------|
| `git pull` 核心代码 | ❌ 不影响 | Operator 代码在 `~/.hermes/plugins/operator/`，不在 repo 内 |
| `pip install` 更新依赖 | ⚠️ 可能影响 | 需确保 Operator 依赖在 `requirements.txt` 中声明 |
| `config.yaml` 更新 | ❌ 不影响 | Operator 有独立 `profiles/operator/config.yaml` |
| 数据库 schema 迁移 | ⚠️ 需关注 | 新字段只增不删，Operator 查询用 `SELECT` 指定列名 |

**关键原则**: Operator 代码 **只放在** `~/.hermes/plugins/operator/` 和 `~/.hermes/scripts/operator/`，
这些路径不在 `~/.hermes/hermes-agent/` git 仓库内，`hermes update` 绝不会触碰。

### 依赖管理

```bash
# Operator 依赖声明在插件目录
# ~/.hermes/plugins/operator/requirements.txt
fastapi>=0.100.0
uvicorn>=0.23.0
python-jose[cryptography]>=3.3.0
passlib>=1.7.4
python-multipart>=0.0.6

# 核心依赖跟随 Hermes 升级
# Hermes 的 requirements.txt 已包含:
#   aiohttp, sqlite3 (stdlib), PyYAML, rich, prompt_toolkit, ...
```

---

## 四、会话管理 API 计划

### 4.1 SessionDB 关键表和字段

```sql
-- state.db (与 Hermes 共享)
CREATE TABLE sessions (
    id TEXT PRIMARY KEY,           -- 会话 UUID，Operator 用 "op_" 前缀
    source TEXT NOT NULL,          -- 'cli' | 'telegram' | 'discord' | 'operator'
    user_id TEXT,                  -- Operator 用户 ID
    model TEXT,                    -- 使用的模型
    title TEXT,                    -- 会话标题
    parent_session_id TEXT,        -- 分支父会话
    started_at REAL,               -- Unix 时间戳
    ended_at REAL,
    end_reason TEXT,
    message_count INTEGER,
    input_tokens INTEGER,
    output_tokens INTEGER,
    estimated_cost_usd REAL,
    -- ...更多字段
);

CREATE TABLE messages (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT REFERENCES sessions(id),
    role TEXT,                     -- 'user' | 'assistant' | 'tool' | 'system'
    content TEXT,
    tool_calls TEXT,               -- JSON
    tool_name TEXT,
    timestamp REAL,
    reasoning TEXT,
    -- ...更多字段
);

-- 关键索引
CREATE INDEX idx_sessions_source ON sessions(source);
CREATE INDEX idx_sessions_started ON sessions(started_at DESC);
```

### 4.2 Operator 会话创建流程

```python
# operator_agent.py — 核心思路

import uuid
from run_agent import AIAgent
from hermes_state import SessionDB

class OperatorSession:
    """Operator 管理的独立会话"""

    def __init__(self, user_id: str, db: SessionDB):
        self.user_id = user_id
        self.db = db
        # 使用 "op_" 前缀标识 Operator 会话
        self.session_id = f"op_{uuid.uuid4().hex[:12]}"

    async def chat(self, message: str, model: str = None) -> dict:
        """发送消息到 AIAgent，获取回复"""
        agent = AIAgent(
            # 共用 Hermes 的 provider 配置
            provider="deepseek",       # 或从 config 读取
            model=model or "deepseek-v4-pro",
            api_mode="chat_completions",

            # Operator 专属会话标识
            session_id=self.session_id,
            platform="operator",       # 自定义平台标识
            source="operator",         # 写入 state.db 的 source 字段

            # 共用 Hermes 资源
            enabled_toolsets=["web", "file", "terminal", "skills", "delegation"],

            # 不跳过上下文文件（让 operator 的 AGENTS.md 生效）
            skip_context_files=False,
        )

        # 执行对话
        result = agent.run_conversation(message)
        return result
```

### 4.3 Operator 会话查询 API

```python
# audit.py — 审核和离散化

class OperatorAudit:
    """Operator 审核层 — 会话查询、审计日志"""

    def __init__(self, db: SessionDB):
        self.db = db

    def list_operator_sessions(self, user_id: str = None, limit: int = 50):
        """列出 Operator 所有会话"""
        query = "SELECT * FROM sessions WHERE source = 'operator'"
        params = []
        if user_id:
            query += " AND user_id = ?"
            params.append(user_id)
        query += " ORDER BY started_at DESC LIMIT ?"
        params.append(limit)
        return self.db._conn.execute(query, params).fetchall()

    def get_session_messages(self, session_id: str, limit: int = 200):
        """获取指定会话的消息历史"""
        return self.db.get_messages(session_id, limit=limit)

    def search_sessions(self, keyword: str):
        """全文搜索会话内容（利用 FTS5）"""
        return self.db.search(keyword)

    def get_session_cost_summary(self, session_id: str):
        """获取会话费用摘要"""
        row = self.db._conn.execute(
            "SELECT input_tokens, output_tokens, estimated_cost_usd "
            "FROM sessions WHERE id = ?", (session_id,)
        ).fetchone()
        return row

    def get_daily_usage_report(self, date: str = None):
        """每日使用报告（离散化审计）"""
        # 按天汇总所有 operator 会话的 token 消耗和费用
        ...
```

### 4.4 会话 ID 命名规范

| 前缀 | 来源 | 示例 |
|------|------|------|
| `op_` | Operator 用户会话 | `op_a3f2c9d1e456` |
| `op_audit_` | Operator 审核记录 | `op_audit_20260520_001` |
| (无前缀) | Hermes 原生会话 | `20260520_143052_a1b2c3` |

通过 `source` 字段 + session_id 前缀双重标识，确保不会误操作。

---

## 五、Operator 独立认证系统

### 5.1 认证架构

```
┌──────────────────────────────────────────┐
│         Operator 认证层 (独立)           │
│                                          │
│  用户 ──→ POST /auth/login ──→ JWT Token│
│          POST /auth/register             │
│          POST /auth/refresh              │
│                                          │
│  所有 /api/* 请求 ──→ Bearer Token 校验  │
│                                          │
│  角色:                                    │
│    • admin   — 全权限，可管理用户         │
│    • auditor — 审核角色，查看所有会话     │
│    • user    — 普通用户，管理自己会话     │
│    • viewer  — 只读                       │
└──────────────────────────────────────────┘
```

### 5.2 认证实现

```python
# ~/.hermes/plugins/operator/auth.py

from datetime import datetime, timedelta
from jose import JWTError, jwt
from passlib.context import CryptContext
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

# 配置（从 plugins/operator/plugin.yaml 或环境变量读取）
SECRET_KEY = os.getenv("OPERATOR_JWT_SECRET", "...")
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 120

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
security = HTTPBearer()

class OperatorAuth:
    def __init__(self, db_path: str = None):
        # Operator 用户数据库（独立于 Hermes）
        self.db_path = db_path or os.path.expanduser(
            "~/.hermes/plugins/operator/users.db"
        )
        self._init_db()

    def _init_db(self):
        conn = sqlite3.connect(self.db_path)
        conn.execute("""
            CREATE TABLE IF NOT EXISTS users (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                username TEXT UNIQUE NOT NULL,
                password_hash TEXT NOT NULL,
                role TEXT NOT NULL DEFAULT 'user',
                created_at REAL NOT NULL,
                last_login REAL
            )
        """)
        conn.commit()
        conn.close()

    def authenticate(self, username: str, password: str) -> str | None:
        """验证用户凭据，返回 JWT token"""
        ...

    def create_user(self, username: str, password: str, role: str = "user"):
        """创建新用户"""
        ...

    def get_current_user(
        self,
        credentials: HTTPAuthorizationCredentials = Depends(security)
    ) -> dict:
        """FastAPI 依赖注入：从 Bearer token 解析用户"""
        token = credentials.credentials
        try:
            payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
            return {
                "user_id": payload["sub"],
                "role": payload["role"],
                "username": payload.get("username")
            }
        except JWTError:
            raise HTTPException(status_code=401, detail="Invalid token")

    def require_role(self, *roles: str):
        """角色鉴权装饰器"""
        async def role_checker(user: dict = Depends(self.get_current_user)):
            if user["role"] not in roles:
                raise HTTPException(status_code=403, detail="Insufficient permissions")
            return user
        return role_checker
```

---

## 六、Operator Web 服务

### 6.1 FastAPI 应用结构

```python
# ~/.hermes/plugins/operator/server.py

from fastapi import FastAPI, Depends, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from fastapi.staticfiles import StaticFiles
from contextlib import asynccontextmanager

from .auth import OperatorAuth
from .operator_agent import OperatorSessionManager
from .audit import OperatorAudit

# 全局实例
auth = OperatorAuth()
session_mgr = None  # 延迟初始化
auditor = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    """应用生命周期 — 初始化 Hermes SessionDB 连接"""
    global session_mgr, auditor
    from hermes_state import SessionDB
    db = SessionDB()  # 共享 state.db
    session_mgr = OperatorSessionManager(db, auth)
    auditor = OperatorAudit(db)
    yield
    db.close()

app = FastAPI(
    title="Hermes Operator API",
    version="1.0.0",
    lifespan=lifespan
)

# CORS — 允许 Operator Dashboard 前端访问
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://operator.example.com"],
    allow_methods=["*"],
    allow_headers=["*"],
)

# ─── 认证路由 ───

@app.post("/auth/login")
async def login(username: str, password: str):
    token = auth.authenticate(username, password)
    if not token:
        raise HTTPException(401, "Invalid credentials")
    return {"access_token": token, "token_type": "bearer"}

@app.post("/auth/register")
async def register(
    username: str,
    password: str,
    user: dict = Depends(auth.require_role("admin"))
):
    """仅 admin 可创建用户"""
    return auth.create_user(username, password)

# ─── 会话路由 ───

@app.post("/api/chat")
async def chat(
    message: str,
    session_id: str = None,  # None = 新会话
    model: str = "deepseek-v4-pro",
    user: dict = Depends(auth.get_current_user)
):
    """发送消息到 AI agent"""
    return await session_mgr.chat(
        user_id=user["user_id"],
        message=message,
        session_id=session_id,
        model=model
    )

@app.get("/api/sessions")
async def list_sessions(
    limit: int = 50,
    user: dict = Depends(auth.get_current_user)
):
    """列出当前用户的会话"""
    if user["role"] in ("admin", "auditor"):
        return auditor.list_operator_sessions(limit=limit)
    return auditor.list_operator_sessions(user_id=user["user_id"], limit=limit)

@app.get("/api/sessions/{session_id}/messages")
async def get_messages(
    session_id: str,
    limit: int = 200,
    user: dict = Depends(auth.get_current_user)
):
    """获取会话消息历史"""
    # 权限检查: 用户只能看自己的会话，admin/auditor 可看所有
    session = auditor.get_session(session_id)
    if not session:
        raise HTTPException(404, "Session not found")
    if user["role"] not in ("admin", "auditor") and session["user_id"] != user["user_id"]:
        raise HTTPException(403, "Access denied")
    return auditor.get_session_messages(session_id, limit=limit)

# ─── 审核路由 ───

@app.get("/api/audit/daily-summary")
async def daily_summary(
    date: str = None,
    user: dict = Depends(auth.require_role("admin", "auditor"))
):
    """每日使用摘要"""
    return auditor.get_daily_usage_report(date)

@app.get("/api/audit/user/{username}")
async def user_audit(
    username: str,
    user: dict = Depends(auth.require_role("admin", "auditor"))
):
    """按用户审计"""
    return auditor.get_user_audit_log(username)

# ─── 模型路由 ───

@app.get("/api/models")
async def list_models(user: dict = Depends(auth.get_current_user)):
    """列出可用的模型（共用 Hermes 配置）"""
    from hermes_cli.config import load_config
    config = load_config()
    return {
        "default": config.get("model", {}).get("default", "deepseek-v4-pro"),
        "available": list_available_models(config)  # 从 provider 目录解析
    }

# 静态文件 — Operator Dashboard
app.mount("/", StaticFiles(directory=str(Path(__file__).parent / "web"), html=True))
```

### 6.2 启动配置

```yaml
# ~/.hermes/profiles/operator/config.yaml
model:
  default: deepseek-v4-pro
  provider: deepseek

# Operator 专属配置
operator:
  host: "0.0.0.0"
  port: 8643
  jwt_secret_env: "OPERATOR_JWT_SECRET"
  cors_origins:
    - "https://operator.example.com"
  session_prefix: "op_"

# 工具集 — 可以比主 Hermes 更严格
agent:
  enabled_toolsets:
    - web
    - file
    - skills
    - delegation
  max_turns: 60  # Operator 默认限制 60 轮

terminal:
  backend: local
  cwd: /home/hermes/operator_workspace
  approval_mode: manual  # 强制手动审批

# 审计
audit:
  enabled: true
  log_retention_days: 90
```

---

## 七、部署清单

### 7.1 初始化步骤

```bash
# 1. 安装 Hermes（如果未安装）
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash

# 2. 创建 Operator Profile
hermes profile create operator

# 3. 创建 Operator 插件目录
mkdir -p ~/.hermes/plugins/operator/web

# 4. 编写插件元数据
cat > ~/.hermes/plugins/operator/plugin.yaml << 'EOF'
name: operator
version: "1.0.0"
description: "Operator — 独立审核控制系统"
author: YourTeam
requires_env:
  - OPERATOR_JWT_SECRET
hooks:
  - on_gateway_start
EOF

# 5. 安装 Operator 依赖
source ~/.hermes/hermes-agent/.venv/bin/activate
pip install -r ~/.hermes/plugins/operator/requirements.txt

# 6. 生成 JWT 密钥
export OPERATOR_JWT_SECRET=$(python -c "import secrets; print(secrets.token_urlsafe(64))")
echo "OPERATOR_JWT_SECRET=$OPERATOR_JWT_SECRET" >> ~/.hermes/profiles/operator/.env

# 7. 启动 Operator 服务
~/.hermes/scripts/operator/start.sh
```

### 7.2 自动升级 Cron Job

```bash
# ~/.hermes/scripts/operator/auto_update.sh
#!/bin/bash
# 每周日凌晨 3:00 自动升级 Hermes

LOG=~/.hermes/logs/auto_update.log
echo "=== Auto update $(date) ===" >> "$LOG"

# 升级前备份 Operator 配置
cp -r ~/.hermes/plugins/operator ~/.hermes/backups/operator_$(date +%Y%m%d)/
cp ~/.hermes/profiles/operator/config.yaml ~/.hermes/backups/

# 执行升级
hermes update --yes >> "$LOG" 2>&1

# 重启 Operator 服务
systemctl --user restart hermes-operator

echo "=== Update complete ===" >> "$LOG"
```

用 `cronjob` 工具创建定时任务:
```bash
hermes cron create "0 3 * * 0" --name "auto-update-hermes"
# 或用执行脚本模式
# no_agent=True, script=~/.hermes/scripts/operator/auto_update.sh
```

### 7.3 systemd 服务

```ini
# ~/.config/systemd/user/hermes-operator.service
[Unit]
Description=Hermes Operator API Server
After=network.target

[Service]
Type=simple
WorkingDirectory=%h/.hermes/plugins/operator
Environment="HERMES_PROFILE=operator"
EnvironmentFile=%h/.hermes/profiles/operator/.env
ExecStart=%h/.hermes/hermes-agent/.venv/bin/python -m uvicorn server:app --host 0.0.0.0 --port 8643
Restart=always
RestartSec=5

[Install]
WantedBy=default.target
```

```bash
systemctl --user enable hermes-operator
systemctl --user start hermes-operator
```

### 7.4 Nginx 反向代理

```nginx
# /etc/nginx/sites-available/hermes-operator
server {
    listen 443 ssl;
    server_name operator.example.com;

    ssl_certificate /etc/letsencrypt/live/operator.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/operator.example.com/privkey.pem;

    # Hermes API Server (主入口)
    location /hermes/ {
        proxy_pass http://127.0.0.1:8642/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 300s;
    }

    # Operator Web (独立入口)
    location / {
        proxy_pass http://127.0.0.1:8643/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 300s;
    }
}
```

---

## 八、扩展指令和离散化审核

### 8.1 扩展指令注入

```python
# ~/.hermes/plugins/operator/operator_agent.py

class OperatorSessionManager:
    def _build_system_prompt(self, user: dict, instructions: list[str] = None):
        """构建 Operator 专用的系统提示词"""

        base_prompt = f"""
You are Hermes Agent operating under Operator supervision.

Operator User: {user['username']} (role: {user['role']})
Session context: Managed and audited by Operator control plane.

Audit requirements:
- All tool calls are logged and reviewable
- Destructive operations require explicit confirmation
- Cost threshold alerts at $0.50 cumulative per session
"""

        if instructions:
            base_prompt += "\n\nAdditional operator instructions:\n"
            for i, inst in enumerate(instructions, 1):
                base_prompt += f"{i}. {inst}\n"

        return base_prompt
```

### 8.2 离散化审核日志

```python
# ~/.hermes/plugins/operator/audit.py

class DiscreteAuditLogger:
    """离散化事件日志 — 每次 Agent 操作都记录为独立事件"""

    def __init__(self, db_path: str):
        self.conn = sqlite3.connect(db_path)
        self.conn.execute("""
            CREATE TABLE IF NOT EXISTS discrete_events (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                event_type TEXT NOT NULL,     -- 'user_message', 'llm_call',
                                              -- 'tool_call', 'tool_result',
                                              -- 'approval', 'error'
                session_id TEXT NOT NULL,
                user_id TEXT,
                timestamp REAL NOT NULL,
                data JSON,                    -- 事件详细信息
                hash TEXT                     -- 数据完整性哈希
            )
        """)
        self.conn.commit()

    def log_event(self, event_type: str, session_id: str,
                  user_id: str, data: dict):
        """记录一个离散事件"""
        import hashlib, json, time
        data_json = json.dumps(data, default=str)
        event_hash = hashlib.sha256(data_json.encode()).hexdigest()[:16]

        self.conn.execute(
            """INSERT INTO discrete_events
               (event_type, session_id, user_id, timestamp, data, hash)
               VALUES (?, ?, ?, ?, ?, ?)""",
            (event_type, session_id, user_id, time.time(), data_json, event_hash)
        )
        self.conn.commit()

    def replay_session(self, session_id: str) -> list[dict]:
        """重放某个会话的所有离散事件（完整审计追踪）"""
        rows = self.conn.execute(
            "SELECT * FROM discrete_events WHERE session_id = ? ORDER BY id",
            (session_id,)
        ).fetchall()
        return [dict(row) for row in rows]

    def integrity_check(self, session_id: str):
        """数据完整性校验 — 验证事件链未被篡改"""
        events = self.replay_session(session_id)
        issues = []
        for e in events:
            recalculated = hashlib.sha256(
                e["data"].encode()
            ).hexdigest()[:16]
            if recalculated != e["hash"]:
                issues.append(f"Event {e['id']}: hash mismatch")
        return issues if issues else "All events verified"
```

---

## 九、关键数据流

### 9.1 用户会话完整流程

```
  用户浏览器                    Operator API               Hermes Core
  ──────────                   ─────────────               ───────────
      │                             │                          │
      │  POST /auth/login           │                          │
      │  {username, password}       │                          │
      │ ──────────────────────────→ │                          │
      │                             │ 查询 users.db            │
      │  ←──── JWT token ──────────│                          │
      │                             │                          │
      │  POST /api/chat             │                          │
      │  Bearer: <token>            │                          │
      │  {message: "帮我分析..."}    │                          │
      │ ──────────────────────────→ │                          │
      │                             │ JWT 验证 + 权限检查       │
      │                             │                          │
      │                             │ 创建/恢复 AIAgent        │
      │                             │ session_id="op_xxx"      │
      │                             │ source="operator"        │
      │                             │ ────────────────────────→│
      │                             │                          │
      │                             │     AIAgent.chat(msg)    │
      │                             │     tool_calls → ...     │
      │                             │     最终回复              │
      │                             │ ←────────────────────────│
      │                             │                          │
      │                             │ 记录离散化事件           │
      │                             │ 写入 state.db            │
      │                             │                          │
      │  ←── SSE stream / JSON ────│                          │
      │     回复 + session_id       │                          │
      │                             │                          │
```

### 9.2 升级数据流

```
  Cron Job / 手动                    Hermes Update                Operator
  ──────────────                    ──────────────                ────────
       │                                  │                          │
       │  hermes update                   │                          │
       │ ────────────────────────────────→│                          │
       │                                  │                          │
       │                  ① git stash    │                          │
       │                     (核心代码的   │                          │
       │                      本地修改)    │                          │
       │                                  │                          │
       │                  ② git pull      │ Operator 代码            │
       │                     upstream     │ 在 ~/.hermes/plugins/    │
       │                     main         │ 完全不受影响             │
       │                                  │                          │
       │                  ③ pip install   │ Operator 依赖            │
       │                     -e .         │ (fastapi等)不被卸载     │
       │                                  │                          │
       │                  ④ git stash     │                          │
       │                     apply        │                          │
       │                                  │                          │
       │  ←── 升级完成 ────────────────── │                          │
       │                                  │                          │
       │  systemctl restart               │                          │
       │  hermes-operator                 │                          │
       │ ─────────────────────────────────────────────────────────→│
       │                                  │                          │
       │                                  │  ←── 服务重启，           │
       │                                  │      加载新版 Hermes     │
       │                                  │      Operator 代码不变   │
```

---

## 十、安全注意事项

1. **API Key 隔离**: Operator JWT Secret ≠ Hermes API_SERVER_KEY
2. **网络隔离**: Hermes API Server 建议只监听 `127.0.0.1:8642`，由 Nginx 反代
3. **数据库访问**: Operator 只读写 state.db 中 `source='operator'` 的会话，不接触其他会话
4. **权限模型**: 最小权限原则 — 默认 user 只能操作自己的会话
5. **升级安全**: 升级前自动备份 Operator 配置和插件目录
6. **审计链**: 离散化事件表保留哈希链，支持完整性校验

---

## 十一、实施路径

| 阶段 | 内容 | 预估时间 |
|------|------|----------|
| **Phase 1** | Operator Profile + 插件骨架 + JWT 认证 | 2-3 天 |
| **Phase 2** | FastAPI 服务 + AIAgent 集成 + 会话 CRUD | 3-4 天 |
| **Phase 3** | 审核/离散化模块 + 扩展指令注入 | 2-3 天 |
| **Phase 4** | Web Dashboard 前端 | 3-5 天 |
| **Phase 5** | 部署 + systemd + Nginx + 自动升级 cron | 1-2 天 |
| **Phase 6** | 测试 + 压力测试 + 安全审计 | 2-3 天 |

---

## 十二、备选方案对比

| 方案 | 优势 | 劣势 | 推荐度 |
|------|------|------|:---:|
| **独立 Profile + 插件 (本方案)** | 分离彻底，升级无干扰，共享所有资源 | 需自行开发 Web 层 | ⭐⭐⭐⭐⭐ |
| 使用 Hermes API Server 作为 Operator 后端 | 免开发 API | 认证单一（Bearer Token），无角色管理，无审核功能 | ⭐⭐⭐ |
| Fork Hermes 仓库 | 完全控制 | 无法自动升级，维护成本极高 | ⭐ |
| 独立安装两个 Hermes | 完全隔离 | 浪费资源，插件/技能无法共享 | ⭐⭐ |

---

**结论**: 推荐 **独立 Profile + 插件架构**，这是 Hermes 原生支持的扩展方式，完全满足分离、共享、可持续升级三大核心需求。
