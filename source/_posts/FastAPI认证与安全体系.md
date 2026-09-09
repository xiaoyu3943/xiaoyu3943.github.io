---
title: FastAPI 认证与安全体系
date: 2026-06-07
tags: [FastAPI, JWT, bcrypt, 认证, 安全, 依赖注入, 中间件]
categories: Python
---

## 第一部分：JWT 认证与 bcrypt 密码安全

### 一、核心组件

```python
from jose import jwt                       # JWT 编码/解码
from passlib.context import CryptContext    # 密码哈希
from app.config import get_settings

settings = get_settings()
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
```

| 组件 | 作用 |
|------|------|
| `python-jose` | JWT（JSON Web Token）的签发和验证 |
| `passlib` + `bcrypt` | 密码哈希和验证明文 |
| `CryptContext` | passlib 的上下文管理器，统一管理多种哈希方案 |

---

### 二、密码哈希（bcrypt）

#### 2.1 哈希与验证

```python
def hash_password(password: str) -> str:
    """对明文密码进行 bcrypt 哈希"""
    return pwd_context.hash(password)

def verify_password(plain: str, hashed: str) -> bool:
    """验证明文密码是否匹配哈希值"""
    return pwd_context.verify(plain, hashed)
```

#### 2.2 bcrypt 为什么安全？

```python
# 同样的密码，每次哈希结果都不同（因为内嵌了随机 salt）
hash_password("mypassword")  # $2b$12$abc...salt1...hash1
hash_password("mypassword")  # $2b$12$xyz...salt2...hash2  ← 完全不同！

# 但验证时都能匹配
verify_password("mypassword", hash1)  # True
verify_password("mypassword", hash2)  # True
```

#### 2.3 bcrypt 哈希串结构

```
$2b$12$LJ3m4ys3GZ0bq8vY5XtQrOK7Xm1kpVh9yRfTn2wCsDlFj8eWxK6ua
│  │  │                                            │
│  │  │                                            └── 哈希值 (31 字符)
│  │  └── Salt (22 字符)
│  └── Cost Factor (2^12 = 4096 轮迭代)
└── 算法标识 (2b = bcrypt)
```

| 组成部分 | 含义 |
|---------|------|
| `$2b$` | 算法版本标识 |
| `$12$` | 计算轮数 = 2^12 = 4096 次迭代（越高越慢，越安全） |
| salt | 22 字符的随机盐，每次自动生成 |
| hash | 最终的哈希值 |

#### 2.4 bcrypt 的三重防护

| 防护 | 原理 | 防御的攻击 |
|------|------|-----------|
| **盐 (Salt)** | 每个密码独立生成随机盐 | 彩虹表攻击（预计算哈希→明文的表） |
| **多轮迭代** | 4096 次迭代，故意让计算变慢 | 暴力破解（试一个密码要 0.3 秒） |
| **不可逆** | 只能 hash → verify，不能反向 | 数据库泄露也无法恢复明文 |

---

### 三、JWT 令牌签发

#### 3.1 Access Token（短期）

```python
def create_access_token(
    subject: str,                        # 通常是 user_id
    extra_claims: dict[str, Any] | None = None,
) -> str:
    now = datetime.now(timezone.utc)
    expire = now + timedelta(minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES)  # 30分钟
    claims = {
        "sub": subject,       # 主题（Subject）— 用户标识
        "iat": now,           # 签发时间（Issued At）
        "exp": expire,        # 过期时间（Expiration）
        "type": "access",     # 令牌类型 — 用于区分 access/refresh
    }
    if extra_claims:
        claims.update(extra_claims)
    return jwt.encode(claims, settings.JWT_SECRET_KEY, algorithm=settings.JWT_ALGORITHM)
```

#### 3.2 Refresh Token（长期）

```python
def create_refresh_token(subject: str) -> str:
    now = datetime.now(timezone.utc)
    expire = now + timedelta(days=settings.REFRESH_TOKEN_EXPIRE_DAYS)  # 7天
    claims = {
        "sub": subject,
        "iat": now,
        "exp": expire,
        "type": "refresh",
    }
    return jwt.encode(claims, settings.JWT_SECRET_KEY, algorithm=settings.JWT_ALGORITHM)
```

#### 3.3 JWT 标准声明（Registered Claims）

| 缩写 | 全称 | 含义 |
|------|------|------|
| `sub` | Subject | 令牌主体（通常是 user_id） |
| `iat` | Issued At | 签发时间戳 |
| `exp` | Expiration Time | 过期时间戳 |
| `type` | （自定义） | 区分 access token 和 refresh token |
| `HS256` |  HMAC-SHA256 签名算法（对称密钥）   |    |

---

### 四、JWT 解码与验证

```python
def decode_token(token: str) -> dict[str, Any]:
    """解码并验证 JWT，无效时抛出 JWTError"""
    return jwt.decode(
        token,
        settings.JWT_SECRET_KEY,
        algorithms=[settings.JWT_ALGORITHM],
    )
```

#### 自动验证的项

| 验证项 | 说明 |
|--------|------|
| **签名** | 验证 token 是否被篡改（用 `JWT_SECRET_KEY` 验签） |
| **过期** | 验证 `exp` 是否已过 |
| **算法** | 验证签发算法是否匹配 |

---

### 五、JWT 双 Token 机制

#### 5.1 为什么需要两个 Token？

```
┌─────────┐                      ┌─────────┐
│  客户端  │                      │  服务端  │
└────┬────┘                      └────┬────┘
     │                                │
     │  POST /login (username + pwd)  │
     │───────────────────────────────→│
     │  {access_token, refresh_token} │
     │←───────────────────────────────│
     │                                │
     │  GET /api/accounts             │
     │  Authorization: Bearer {access}│ ← access_token 30分钟有效
     │───────────────────────────────→│
     │                                │
     │  ... 30分钟后，access 过期 ...  │
     │                                │
     │  POST /api/refresh             │
     │  {refresh_token}               │ ← refresh_token 7天有效
     │───────────────────────────────→│
     │  {新的 access_token}            │
     │←───────────────────────────────│
```

#### 5.2 双 Token 对比

| 维度 | Access Token | Refresh Token |
|------|-------------|---------------|
| **有效期** | 短（30分钟） | 长（7天） |
| **使用频率** | 每次 API 请求都携带 | 只在续期时使用 |
| **泄露风险** | 高（频繁传输） | 低（很少传输） |
| **泄露影响** | 30 分钟内有效 | 7 天内可换新 token，需支持撤销 |
| **存储位置** | 客户端内存 | 客户端安全存储（httpOnly cookie 等） |

#### 5.3 为什么"泄露影响窗口不同"是核心优势？

```
单一长期 Token 方案：
  Token 泄露 → 攻击者可在 7 天内冒充用户 → ❌

双 Token 方案：
  Access Token 泄露 → 攻击者只能在 30 分钟内冒充用户 → ✅
  Refresh Token 泄露 → 需要额外的 token 撤销机制 → ✅
```

---

### 六、令牌类型校验

在依赖注入中校验令牌类型，防止用 Refresh Token 访问 API：

```python
# app/dependencies.py 中
if payload.get("type") != "access":
    raise HTTPException(status_code=401, detail="无效的令牌类型")
```

| token 类型 | 允许的操作 |
|-----------|-----------|
| `access` | 访问所有受保护的 API |
| `refresh` | **只能**访问 `/api/refresh` 续期接口 |

---

### 七、JWT 与 Session 对比

| 维度 | JWT | 服务端 Session |
|------|-----|---------------|
| **状态** | 无状态（token 自包含用户信息） | 有状态（服务端存储 session） |
| **扩展性** | 好（无需共享 session 存储） | 需要 Redis 等共享存储 |
| **吊销** | 难（需要黑名单机制） | 易（删除 session 即可） |
| **体积** | 较大（每次请求携带完整 token） | 小（只传 session_id） |
| **适用** | 微服务、移动端 API | 传统 Web 应用 |

---

## 第二部分：FastAPI 依赖注入与认证中间件

### 八、依赖注入

FastAPI 的 `Depends()` 是一种**声明式依赖管理**机制：路由函数声明它需要什么，FastAPI 自动注入。

```python
@router.get("/api/v1/accounts")
async def list_accounts(
    current_user: User = Depends(get_current_user),  # ← 需要当前用户
    db: AsyncSession = Depends(get_db),              # ← 需要数据库会话
):
    # current_user 和 db 已经被注入，可直接使用
    ...
```

> `Depends` 就是"自动帮你准备好东西"，你不用自己创建、不用自己管、不用自己传，你只需要声明：我要这个 → 系统自动给你注入。函数式依赖：用 `yield` 提供资源，`finally` 保证清理

---

### 九、认证流程

```
┌────────────────────────────────────────────────────────────┐
│  HTTP 请求                                                  │
│  Authorization: Bearer eyJhbGciOi...                       │
└────────────────────────┬───────────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────────┐
│  Step 1: HTTPBearer 提取 Token                              │
│  ─────────────────────────────                              │
│  · 从 Authorization 头解析 "Bearer <token>"                 │
│  · auto_error=False → token 不存在时不报错，返回 None       │
└────────────────────────┬───────────────────────────────────┘
                         │
              ┌──────────┴──────────┐
              │ token 存在吗？       │
              └──────────┬──────────┘
                 是 │            │ 否
                    ▼            ▼
              ┌──────────┐  ┌──────────────────┐
              │ 继续      │  │ HTTPException(401)│
              └────┬─────┘  │ "缺少认证令牌"     │
                   │        └──────────────────┘
                   ▼
┌────────────────────────────────────────────────────────────┐
│  Step 2: decode_token() 验证签名 + 过期                     │
│  ───────────────────────────────────                        │
│  · 验证签名（用 JWT_SECRET_KEY）                            │
│  · 检查 exp（是否过期）                                     │
│  · 失败 → HTTPException(401, "无效的认证令牌")              │
└────────────────────────┬───────────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────────┐
│  Step 3: 校验 Token 类型                                    │
│  ─────────────────────────                                 │
│  · payload.type 必须是 "access"                            │
│  · 防止用 Refresh Token 访问 API                           │
│  · 失败 → HTTPException(401, "无效的令牌类型")              │
└────────────────────────┬───────────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────────┐
│  Step 4: 从数据库加载用户                                   │
│  ───────────────────────────                                │
│  · payload.sub → user_id                                   │
│  · SELECT * FROM users WHERE id = ?                        │
│  · 检查用户是否存在 + is_active                            │
│  · 失败 → HTTPException(401, "用户不存在或已禁用")          │
└────────────────────────┬───────────────────────────────────┘
                         │
                         ▼
                  返回 User 对象 → 注入到路由函数
```

---

### 十、依赖链（Dependency Chain）

FastAPI 自动解析多层依赖：

```
get_current_user
    ├── Depends(security_scheme)     ← HTTPBearer()
    └── Depends(get_db)              ← 异步数据库会话
            └── AsyncSessionLocal()  ← 连接池
```

#### 依赖解析顺序

```
请求到达
  │
  ├── 1. get_db() 开始
  │     └── AsyncSessionLocal() 创建 session
  │
  ├── 2. security_scheme (HTTPBearer)
  │     └── 从 Header 提取 token
  │
  ├── 3. get_current_user() 执行
  │     ├── 接收 credentials (来自 step 2)
  │     ├── 接收 db (来自 step 1)
  │     └── 返回 User 对象
  │
  ├── 4. 路由函数执行
  │     └── 接收 current_user 和 db
  │
  └── 5. get_db() finally 块
        └── commit / rollback / close
```

---

### 十一、`HTTPBearer` 配置

```python
security_scheme = HTTPBearer(auto_error=False)
```

| 设置 | 行为 |
|------|------|
| `auto_error=True`（默认） | Token 不存在时**自动**抛出 401 |
| `auto_error=False` | Token 不存在时返回 `None`，由开发者自行处理 |

> `HTTPBearer()` — 自动从 `Authorization: Bearer <token>` 头提取令牌
> `auto_error=False` — 令牌缺失时不自动抛401，返回 `None`（用于可选认证）
> `credentials.credentials` — 获取令牌字符串

#### 为什么设置 `auto_error=False`？

```python
# auto_error=False 的好处：可以区分"没传 token"和"token 无效"
if credentials is None:
    raise HTTPException(status_code=401, detail="缺少认证令牌")
# 与 decode 失败的 401 信息不同，便于排查问题
```

---

### 十二、`Depends` 的三种使用场景

#### 12.1 注入依赖

```python
# 场景 1: 注入数据库会话
db: AsyncSession = Depends(get_db)

# 场景 2: 注入当前用户（需要认证的路由）
current_user: User = Depends(get_current_user)

# 场景 3: 多个依赖组合
async def create_account(
    account: AccountCreate,                    # 请求体
    current_user: User = Depends(get_current_user),  # 认证
    db: AsyncSession = Depends(get_db),              # 数据库
): ...
```

#### 12.2 路由级依赖

```python
# 整个路由模块都需要认证
router = APIRouter(
    prefix="/api/v1",
    dependencies=[Depends(get_current_user)],  # 所有路由自动认证
)
```

#### 12.3 全局依赖

```python
# 整个应用都需要数据库会话
app = FastAPI(dependencies=[Depends(get_db)])
```

---

### 十三、依赖注入 vs 手动管理

| 维度 | `Depends()` 注入 | 手动管理 |
|------|-----------------|---------|
| **代码量** | 少（声明即用） | 多（每次手动创建和关闭） |
| **生命周期** | 自动管理（进入/退出） | 手动 try/finally |
| **测试** | 易（可替换依赖） | 难（硬编码） |
| **可读性** | 好（函数签名即文档） | 差（实现细节淹没业务逻辑） |
| **复用** | 好（一个依赖多处使用） | 差（到处复制粘贴） |

---

### 十四、完整认证流程

```
1. 用户注册
   password (明文) ──→ hash_password() ──→ 存入数据库

2. 用户登录
   输入 password ──→ verify_password() ──→ 匹配成功
                                          ├── create_access_token()
                                          └── create_refresh_token()

3. API 请求
   Authorization: Bearer {access_token}
     ──→ HTTPBearer 提取 token
       ──→ decode_token() 验签 + 检查过期
         ──→ 从 claims 中取 user_id
           ──→ 查数据库加载 User 对象

4. Token 续期
   POST /refresh {refresh_token}
     ──→ decode_token() 验签
       ──→ 验证 type == "refresh"
         ──→ 生成新的 access_token
```

---

### 十五、核心要点速查

| 要点 | 实现 |
|------|------|
| 密码存储 | bcrypt 哈希，永不明文 |
| 密码验证 | `pwd_context.verify(plain, hashed)` |
| Token 签发 | `jwt.encode(claims, secret, algorithm)` |
| Token 验证 | `jwt.decode(token, secret, algorithms)` |
| 双 Token | Access（30分钟） + Refresh（7天） |
| Token 类型隔离 | `type: "access"` vs `type: "refresh"` |
| 密钥管理 | 环境变量注入，不硬编码 |
| 时间标准 | UTC 时间（`datetime.now(timezone.utc)`） |
| `Depends()` | FastAPI 声明式依赖注入 |
| `HTTPBearer` | 从 Authorization 头提取 Bearer Token |
| `auto_error=False` | Token 不存在时不自动报错，由自己处理 |
| 依赖链 | FastAPI 自动递归解析多层嵌套依赖 |
| 用户加载 | 每次请求从 DB 加载最新用户状态 |
| `is_active` 检查 | 支持软禁用用户（不删除记录） |
| `scalar_one_or_none()` | 安全查询：不抛异常，返回 None 自行处理 |

---

### 配置参数

```python
# JWT 配置
JWT_SECRET_KEY: str = "change-me-in-production"  # 签名密钥
JWT_ALGORITHM: str = "HS256"                     # 签名算法
ACCESS_TOKEN_EXPIRE_MINUTES: int = 30             # Access Token 30分钟
REFRESH_TOKEN_EXPIRE_DAYS: int = 7                # Refresh Token 7天
```

| 参数 | 生产环境要求 |
|------|------------|
| `JWT_SECRET_KEY` | **必须更换**！使用 `openssl rand -hex 32` 生成 |
| `JWT_ALGORITHM` | HS256 适合单服务；多服务用 RS256（公钥/私钥） |
| `ACCESS_TOKEN_EXPIRE_MINUTES` | 建议 15~60 分钟 |
| `REFRESH_TOKEN_EXPIRE_DAYS` | 建议 7~30 天 |
