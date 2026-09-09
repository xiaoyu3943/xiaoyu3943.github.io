---
title: Pydantic 数据验证与配置管理完整指南
date: 2026-06-07
tags: [pydantic, pydantic-settings, 配置管理, 数据验证, Python, Schema]
categories: Python
---

## 第一部分：pydantic-settings 配置管理

### 一、核心组件

```python
from pydantic_settings import BaseSettings
from functools import lru_cache
```

| 组件 | 来源 | 作用 |
|------|------|------|
| `BaseSettings` | `pydantic_settings` | 配置基类，自动从环境变量和 `.env` 文件读取配置 |
| `lru_cache` | `functools` | 实现缓存装饰器，确保 `Settings` 实例全局唯一（单例模式） |

> `pydantic_settings` 是 pydantic 生态的扩展库，专为配置管理设计。底层依赖 python-dotenv 解析 .env 文件。
> `env_file = ".env"` — 指定配置文件路径
> `@property` — 计算属性（如拼接数据库URL）

---

### 二、配置优先级

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   环境变量     │ ──→ │ pydantic-    │ ──→ │  Settings()  │
│   .env 文件   │     │  settings     │     │   实例       │
│   默认值       │     │  (解析+验证)  │     │  (类型安全)  │
└──────────────┘     └──────────────┘     └──────────────┘

优先级：环境变量 > .env 文件 > 代码默认值
```

| 优先级（高→低） | 来源 | 适用场景 |
|:---:|------|------|
| 1 | **环境变量** | 生产环境（K8s/Docker 注入） |
| 2 | **`.env` 文件** | 本地开发环境 |
| 3 | **代码默认值** | 兜底，通常是开发默认值 |

> **关键原则**：环境变量优先级最高，确保生产部署时可以通过容器平台覆盖所有敏感配置。

---

### 三、内部配置类 `Config`

```python
class Config:
    env_file = ".env"         # 自动加载 .env 文件
    case_sensitive = True     # 环境变量名大小写敏感
```

#### 3.1 `env_file`

| 设置 | 效果 |
|------|------|
| `env_file = ".env"` | 自动从项目根目录的 `.env` 文件加载环境变量 |
| 可选多个文件 | `env_file = (".env", ".env.prod")` 可指定多个文件，前者优先级更高 |

#### 3.2 `case_sensitive`

| 设置 | 行为 |
|------|------|
| `True` | 环境变量名严格区分大小写，`DB_HOST` 与 `db_host` 是不同的变量 |
| `False`（默认） | 大小写不敏感 |

---

### 四、计算属性：拼接连接字符串

```python
@property
def database_url(self) -> str:
    """拼接异步 MySQL 连接字符串"""
    return (
        f"mysql+asyncmy://{self.DB_USER}:{self.DB_PASSWORD}"
        f"@{self.DB_HOST}:{self.DB_PORT}/{self.DB_NAME}"
    )

@property
def redis_url(self) -> str:
    """拼接 Redis 连接字符串"""
    return (
        f"redis://:{self.REDIS_PASSWORD}"
        f"@{self.REDIS_HOST}:{self.REDIS_PORT}/{self.REDIS_DB}"
    )
```

#### 4.1 设计优势

| 优势 | 说明 |
|------|------|
| **一处定义，全局使用** | 连接字符串格式变更只需修改一处 |
| **类型安全** | `@property` 返回 `str`，调用方无需手动拼接 |
| **避免重复** | 各模块直接 `settings.database_url`，不重复写拼接逻辑 |

#### 4.2 连接字符串格式

```
异步 MySQL： mysql+asyncmy://用户名:密码@主机:端口/数据库名
Redis：      redis://:密码@主机:端口/数据库编号
```

| 部分 | 说明 |
|------|------|
| `mysql+asyncmy` | 数据库类型 + 异步驱动名 |
| `用户名:密码@主机:端口` | 认证与连接信息 |
| `/数据库名` | 目标数据库 |

---

### 五、单例模式：`@lru_cache()`

```python
@lru_cache()
def get_settings() -> Settings:
    """单例模式 — 整个应用只创建一次 Settings 实例"""
    return Settings()
```

#### 5.1 `lru_cache` 工作原理

```
第一次调用 get_settings()
  → 创建 Settings() 实例
  → 读取 .env + 环境变量
  → 缓存实例

后续调用 get_settings()
  → 直接返回缓存的实例（不再读取文件、不再创建对象）
```

#### 5.2 为什么用 `lru_cache` 而不是 `global` 变量？

| 方式 | 问题 |
|------|------|
| `global` 变量 | 模块加载时立即执行，可能早于 `.env` 加载 |
| `lru_cache` | **惰性求值**——首次调用时才创建实例，时序更可控 |

#### 5.3 `@lru_cache()` vs `@lru_cache(maxsize=1)`

- `@lru_cache()` 无参调用默认 `maxsize=128`
- 对于返回单例的函数，两者效果相同（只缓存一个结果）
- 显式写 `maxsize=1` 更语义化，但不是必须的

---

### 六、`.env` 文件规范

```env
# .env — 本地开发环境变量（不提交到 Git！）
DEBUG=true
DB_HOST=localhost
DB_PORT=3306
DB_USER=app_user
DB_PASSWORD=app_pass
DB_NAME=financial_db
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=redis_pass
JWT_SECRET_KEY=dev-secret-key-do-not-use-in-production
```

#### 6.1 文件规范

| 规则 | 说明 |
|------|------|
| 格式 | `KEY=VALUE`，每行一对 |
| 引号 | 值不需要引号（`KEY=hello` 而非 `KEY="hello"`） |
| 布尔值 | `true`/`false`（小写），会被 pydantic 自动转为 `True`/`False` |

#### 6.2 安全注意事项

| 原则 | 说明 |
|------|------|
| **不提交到 Git** | `.env` 必须加入 `.gitignore` |
| **提供模板文件** | 提供 `.env.example` 标注必填项，方便团队成员复制 |
| **不使用生产密钥** | 敏感信息（密钥、密码）在 `.env` 中只放开发用假值 |
| **生产用环境变量** | K8s Secret / Docker `--env` / CI/CD 变量注入 |

---

### 七、在应用中的使用

```python
from app.config import get_settings

settings = get_settings()

# 使用示例
print(settings.APP_NAME)       # "Financial Transfer System"
print(settings.database_url)   # "mysql+asyncmy://app_user:app_pass@localhost:3306/financial_db"
print(settings.redis_url)      # "redis://:redis_pass@localhost:6379/0"
```

无论在哪里调用 `get_settings()`，都返回同一个实例，配置在应用生命周期内保持一致。

---

## 第二部分：Pydantic Schemas 数据验证

### 八、Schema 分层设计

```
app/schemas/
├── user.py          # 用户相关请求/响应
├── account.py       # 账户相关请求/响应
└── transaction.py   # 交易相关请求/响应
```

每个模块遵循 **"请求入、响应出"** 的命名规范：

| 命名模式 | 示例 | 用途 |
|---------|------|------|
| `XxxCreate` | `UserCreate`, `TransferCreate` | 创建/提交的请求体 |
| `XxxLogin` | `UserLogin` | 登录请求体 |
| `XxxResponse` | `UserResponse`, `TransactionResponse` | 返回给客户端的响应体 |

---

### 九、用户 Schema

```python
from pydantic import BaseModel, EmailStr, Field
from datetime import datetime

class UserCreate(BaseModel):
    """注册请求体"""
    username: str = Field(..., min_length=3, max_length=50)
    email: EmailStr
    password: str = Field(..., min_length=8, max_length=128)

class UserLogin(BaseModel):
    """登录请求体"""
    username: str
    password: str

class UserResponse(BaseModel):
    """用户信息响应体"""
    id: str
    username: str
    email: str
    is_active: bool
    created_at: datetime
    model_config = {"from_attributes": True}

class TokenResponse(BaseModel):
    """登录响应体"""
    access_token: str
    refresh_token: str
    token_type: str = "bearer"
    expires_in: int
```

#### 9.1 `Field(...)` — 必填 vs 可选

```python
username: str = Field(...)            # ... 表示必填（等于没有默认值）
password: str = Field(..., min_length=8)  # 必填 + 最小长度校验
```

| 写法 | 含义 |
|------|------|
| `Field(...)` | **必填**，无默认值 |
| `Field(default="CNY")` | 可选，有默认值 |
| `Field(default=None)` | 可选，默认为 None |

#### 9.2 `EmailStr` — 内置邮箱校验

```python
email: EmailStr  # Pydantic 自动校验 email 格式
```

- 安装依赖：`pip install pydantic[email]` 或 `pip install email-validator`
- 无效格式（如 `"not-an-email"`）会直接拒收，返回 422 错误

#### 9.3 `Field` 常用校验参数

| 参数 | 适用类型 | 示例 |
|------|---------|------|
| `min_length` / `max_length` | `str` | `Field(..., max_length=50)` |
| `gt` / `lt` | `int`, `float`, `Decimal` | `Field(..., gt=0)` (>0) |
| `ge` / `le` | `int`, `float`, `Decimal` | `Field(..., ge=0)` (>=0) |
| `pattern` | `str` | `Field(..., pattern=r"^\d{11}$")` |

#### 9.4 `from_attributes=True` — ORM → Pydantic 自动映射

```python
class UserResponse(BaseModel):
    model_config = {"from_attributes": True}  # Pydantic v2 写法
```

| 设置 | 效果 |
|------|------|
| `from_attributes=True` | 允许 `Schema.model_validate(orm_obj)` 从 ORM 对象自动映射 |
| 未设置 | 只能从 `dict` / JSON 构造，不能从 ORM 对象自动映射 |

**使用示例：**

```python
user = await db.get(User, user_id)           # ORM 对象
return UserResponse.model_validate(user)      # 自动映射 → Pydantic 对象 → JSON
```

> 能直接把数据库查出来的用户/账户/交易，丢给响应模型自动转 JSON！

---

### 十、账户 Schema

```python
from pydantic import BaseModel, Field
from decimal import Decimal
from datetime import datetime
from app.models.account import AccountType

class AccountCreate(BaseModel):
    """创建账户请求体"""
    account_type: AccountType = AccountType.SAVINGS
    currency: str = Field(default="CNY", min_length=3, max_length=3)

class AccountResponse(BaseModel):
    """账户信息响应体"""
    id: str
    account_number: str
    account_type: AccountType
    balance: Decimal
    currency: str
    status: str
    created_at: datetime
    model_config = {"from_attributes": True}
```

#### 10.1 `Decimal` 的 JSON 序列化

```python
balance: Decimal  # Pydantic 自动将 Decimal 序列化为 JSON 字符串
```

```
Python: Decimal("1000.50")
  → JSON: "1000.50"    # 字符串形式，确保精度不丢失
```

---

### 十一、交易 Schema

```python
from pydantic import BaseModel, Field, condecimal
from decimal import Decimal
from datetime import datetime
from typing import Optional

class TransferCreate(BaseModel):
    """转账请求体"""
    from_account_id: str
    to_account_id: str
    amount: condecimal(gt=Decimal("0"), max_digits=18, decimal_places=2)
    currency: str = Field(default="CNY")
    description: Optional[str] = Field(default=None, max_length=200)
    idempotency_key: str = Field(..., min_length=1, max_length=64)

class TransactionResponse(BaseModel):
    """交易响应体"""
    id: str
    transaction_no: str
    type: str
    status: str
    from_account_id: Optional[str]
    to_account_id: Optional[str]
    amount: Decimal
    fee: Decimal
    currency: str
    description: Optional[str]
    error_message: Optional[str]
    completed_at: Optional[datetime]
    created_at: datetime
    model_config = {"from_attributes": True}
```

#### 11.1 `condecimal` — Decimal 约束

```python
amount: condecimal(gt=Decimal("0"), max_digits=18, decimal_places=2)
```

| 参数 | 含义 |
|------|------|
| `gt=Decimal("0")` | 金额必须 **大于 0**（转账金额不能为 0 或负数） |
| `max_digits=18` | 最多 18 位有效数字 |
| `decimal_places=2` | 最多 2 位小数 |

#### 11.2 Pydantic 常用约束类型一览

| 类型 | 用途 | 示例 |
|------|------|------|
| `conint(gt=0, lt=100)` | 约束整数 | 分页参数 |
| `confloat(ge=0.0)` | 约束浮点数 | 利率 |
| `condecimal(gt=0)` | 约束 Decimal | 金额 |
| `constr(min_length=1, max_length=100)` | 约束字符串 | 用户名 |

---

### 十二、Pydantic 数据流全景

```
┌─────────────────────────────────────────────────────┐
│                    请求阶段                           │
├─────────────────────────────────────────────────────┤
│                                                      │
│  HTTP 请求体 (JSON)                                  │
│  {                                                   │
│    "username": "alice",                              │
│    "amount": 100.50,                                 │
│    "email": "alice@example.com"                      │
│  }                                                   │
│        │                                             │
│        ▼                                             │
│  ┌──────────────────────────┐                        │
│  │  Pydantic 自动验证        │                       │
│  │  · 类型转换: "100.50" → Decimal("100.50")        │
│  │  · 格式校验: email 是否合法                       │
│  │  · 长度校验: username 是否 3-50 字符              │
│  │  · 范围校验: amount 是否 > 0                     │
│  └──────────────────────────┘                        │
│        │                                             │
│        ▼                                             │
│  Python 对象（已验证，可直接安全使用）                │
│  UserCreate(username="alice", email="...", ...)      │
│                                                      │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│                    响应阶段                           │
├─────────────────────────────────────────────────────┤
│                                                      │
│  ORM 对象 (SQLAlchemy Model)                         │
│  User(id="uuid-123", username="alice", ...)          │
│        │                                             │
│        ▼                                             │
│  ┌──────────────────────────┐                        │
│  │  from_attributes=True     │                       │
│  │  ORM 属性 → Schema 字段    │                      │
│  └──────────────────────────┘                        │
│        │                                             │
│        ▼                                             │
│  Pydantic 对象                                       │
│  UserResponse(id="uuid-123", username="alice", ...)  │
│        │                                             │
│        ▼                                             │
│  HTTP 响应体 (JSON)                                  │
│  {                                                   │
│    "id": "uuid-123",                                 │
│    "username": "alice",                              │
│    "is_active": true                                 │
│  }                                                   │
│                                                      │
└─────────────────────────────────────────────────────┘
```

---

### 十三、Request Body vs Response Schema 对比原则

| 维度 | Request Schema (Create) | Response Schema |
|------|------------------------|-----------------|
| **目的** | 校验输入 | 控制输出 |
| **是否暴露全部字段** | 否，仅需要的字段 | 是，或按需选取 |
| **敏感字段** | 接收但不返回（如 `password`） | 绝不返回（如 `hashed_password`） |
| **默认值** | 常用（简化客户端请求） | 无（服务端给出完整数据） |
| **验证严格度** | **严格**（`Field(...)`） | 宽松（只做类型转换） |

```python
# Request: 接收 password，不返回 hashed_password
class UserCreate(BaseModel):
    password: str = Field(..., min_length=8)  # 接收明文

# Response: 只返回安全字段，不暴露 hashed_password
class UserResponse(BaseModel):
    id: str
    username: str
    email: str
    # 没有 password 和 hashed_password！
```

---

### 十四、核心要点速查

| 知识点 | 关键代码 | 说明 |
|--------|---------|------|
| 必填字段 | `Field(...)` | `...` 表示无默认值 |
| 邮箱校验 | `EmailStr` | 自动验证格式 |
| 字符串约束 | `Field(min_length=3, max_length=50)` | 长度限制 |
| Decimal 约束 | `condecimal(gt=0, max_digits=18, decimal_places=2)` | 金额精确约束 |
| ORM 映射 | `model_config = {"from_attributes": True}` | 从 ORM 对象自动转换 |
| 可选字段 | `str \| None` + `Field(default=None)` | 允许为空 |
| 默认值 | `Field(default="CNY")` | 客户端可省略此字段 |
| 单例配置 | `@lru_cache()` | 全局唯一 Settings 实例 |
| 配置优先级 | 环境变量 > .env > 默认值 | 生产环境可覆盖 |
