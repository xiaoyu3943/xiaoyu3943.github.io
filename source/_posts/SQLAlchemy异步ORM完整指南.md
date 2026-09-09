---
title: SQLAlchemy 异步 ORM 完整指南
date: 2026-06-07
tags: [SQLAlchemy, 异步, 数据库, Python, ORM, 连接池]
categories: Python
---

## 第一部分：异步数据库配置

### 一、核心组件导入

```python
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine, async_sessionmaker
from sqlalchemy.orm import DeclarativeBase
```

| 组件 | 作用 |
|------|------|
| `create_async_engine` | 创建异步数据库引擎，管理连接池，是数据库通信的核心入口 |
| `AsyncSession` | 异步会话类，替代同步版的 `Session`，所有数据库操作都在会话中进行 |
| `async_sessionmaker` | 异步会话工厂，封装了引擎和会话配置，方便批量创建 Session |
| `DeclarativeBase` | 声明式映射基类，所有 ORM 模型类的父类 |

---

### 二、异步引擎 (`create_async_engine`)

```python
engine = create_async_engine(
    settings.database_url,       # 数据库连接 URL
    pool_size=settings.DB_POOL_SIZE,   # 连接池核心大小
    max_overflow=settings.DB_MAX_OVERFLOW,  # 连接池溢出上限
    echo=settings.DEBUG,          # SQL 日志打印开关
    pool_pre_ping=True,           # 连接"活性探测"
    pool_recycle=3600,            # 连接最大存活时间（秒）
)
```

#### 2.1 连接池参数

| 参数 | 含义 | 典型值 |
|------|------|--------|
| `pool_size` | 连接池中**常驻**的连接数（核心连接数） | 5~20 |
| `max_overflow` | 超出 `pool_size` 后**额外可创建**的连接数上限 | 10~30 |
| **最大连接数** | `pool_size + max_overflow` | — |

> **工作机制**：当请求数 ≤ `pool_size` 时，使用已有连接；当请求数超过 `pool_size` 时，创建额外连接（最多 `max_overflow` 个）；达到上限后新请求进入等待队列。

#### 2.2 `pool_pre_ping=True`

- 每次从连接池**取出连接前**，先发送一条 `SELECT 1` 探测语句
- **作用**：检测连接是否仍有效（避免因 MySQL 8小时超时断开导致的 `MySQL server has gone away` 错误）
- **代价**：每次获取连接多一次网络往返，适合**连接空闲时间较长**的场景

#### 2.3 `pool_recycle=3600`

- 连接自创建起**3600 秒（1 小时）**后自动回收重建
- **作用**：防止数据库端主动断开长时间未活动的连接（如 MySQL 的 `wait_timeout`）
- **与 `pool_pre_ping` 的区别**：
  - `pool_recycle`：**主动**定时回收，时间一到立即重建
  - `pool_pre_ping`：**被动**检测，取出连接时才检查是否可用

#### 2.4 `echo=settings.DEBUG`

- 设置为 `True` 时，所有 SQL 语句及参数会打印到控制台
- **适用场景**：开发调试阶段开启，生产环境务必关闭

---

### 三、异步会话工厂 (`async_sessionmaker`)

```python
AsyncSessionLocal = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,
)
```

#### 3.1 `class_=AsyncSession`

- 指定工厂生成的会话类型为 `AsyncSession`
- 所有数据库操作（增删改查）都在此会话中执行

#### 3.2 `expire_on_commit=False`

| 设置 | 行为 |
|------|------|
| `True`（默认） | `commit()` 后，会话中所有 ORM 对象的属性**标记为过期**，下次访问属性会触发懒加载查询 |
| `False`（推荐） | `commit()` 后对象属性**保持有效**，可继续访问，避免无意中触发额外的数据库查询 |

> **为什么推荐 `False`？**  
> 在 Web 请求场景中，数据通常在一次响应中返回给客户端。提交后不再需要重新加载对象，`expire_on_commit=False` 避免了不必要的懒加载查询，提升性能。

---

### 四、声明式基类 (`DeclarativeBase`)

```python
class Base(DeclarativeBase):
    """所有 ORM 模型继承此类"""
    pass
```

- 所有数据表模型类都**继承自 `Base`**
- SQLAlchemy 通过 `Base.metadata` 感知所有表结构，用于自动建表（配合 Alembic 迁移）

---

### 五、依赖注入获取数据库会话 (`get_db`)

```python
async def get_db():
    async with AsyncSessionLocal() as session:
        try:
            yield session            # ① 将会话交给请求处理
            await session.commit()   # ② 请求成功 → 提交事务
        except Exception:
            await session.rollback() # ③ 请求失败 → 回滚事务
            raise                    # ④ 继续向上抛出异常
        finally:
            await session.close()    # ⑤ 归还连接到连接池
```

#### 5.1 执行流程

```
请求到达 → 创建 Session → yield session（处理业务逻辑）
                              ├── 成功 → commit() → close()
                              └── 异常 → rollback() → raise → close()
```

#### 5.2 关键知识点

| 步骤 | 操作 | 说明 |
|------|------|------|
| `yield session` | 生成器函数 | FastAPI 的 `Depends(get_db)` 会在此处"暂停"，等待请求处理完成 |
| `commit()` | 提交事务 | 将本次请求的所有数据库变更**原子性地**写入数据库 |
| `rollback()` | 回滚事务 | 发生异常时撤销所有未提交的变更，保证数据一致性 |
| `close()` | 归还连接 | 将连接归还到连接池（不是关闭 TCP 连接），供下一个请求复用 |

#### 5.3 在 FastAPI 中使用

```python
from fastapi import Depends
from app.database import get_db

@app.get("/users")
async def get_users(db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User))
    return result.scalars().all()
```

---

### 六、架构设计要点总结

| 设计点 | 实践 | 原因 |
|--------|------|------|
| 异步引擎 | `create_async_engine` + `asyncmy` 驱动 | 支持高并发，不阻塞事件循环 |
| 连接池 | `pool_size` + `max_overflow` | 平衡资源消耗与并发能力 |
| 连接健康检查 | `pool_pre_ping` + `pool_recycle` | 防止"僵尸连接"导致请求失败 |
| 会话工厂 | `async_sessionmaker` | 统一管理 Session 配置，避免散落各处 |
| 事务管理 | `commit` / `rollback` 由依赖注入统一处理 | 业务代码无需手动管理事务 |
| 连接回收 | `finally` 中 `close()` | 确保无论成功或失败，连接都会归还 |
| 过期策略 | `expire_on_commit=False` | 避免提交后不必要的数据重载 |

---

## 第二部分：ORM 模型设计

### 七、模型文件结构

```
app/models/
├── user.py          # 用户模型
├── account.py       # 账户模型
└── transaction.py   # 交易模型
```

三个模型通过 `relationship` 形成关联：

```
User (1) ──── (N) Account (1) ──── (N) Transaction (N) ──── (1) Account
                   │                                               │
                   └── outbound_transactions (from_account)        │
                   └── inbound_transactions  (to_account)  ←───────┘
```

---

### 八、用户模型

```python
class User(Base):
    __tablename__ = "users"

    id: Mapped[str] = mapped_column(
        String(36), primary_key=True,
        default=lambda: str(uuid.uuid4())
    )
    username: Mapped[str] = mapped_column(
        String(50), unique=True, nullable=False, index=True
    )
    email: Mapped[str] = mapped_column(
        String(100), unique=True, nullable=False
    )
    hashed_password: Mapped[str] = mapped_column(
        String(255), nullable=False
    )
    is_active: Mapped[bool] = mapped_column(
        Boolean, default=True
    )
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now()
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
        onupdate=func.now(),
    )

    accounts: Mapped[list["Account"]] = relationship(
        back_populates="user"
    )
```

#### 8.1 字段解析

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | `String(36)` | PK | UUID 字符串，由 Python 层生成 |
| `username` | `String(50)` | UNIQUE, NOT NULL, INDEX | 登录名，唯一索引 |
| `email` | `String(100)` | UNIQUE, NOT NULL | 邮箱，唯一 |
| `hashed_password` | `String(255)` | NOT NULL | 哈希后的密码（不存明文） |
| `is_active` | `Boolean` | default=True | 软删除/禁用标记 |
| `created_at` | `DateTime(tz=True)` | server_default=func.now() | 数据库自动生成时间戳 |
| `updated_at` | `DateTime(tz=True)` | onupdate=func.now() | 每次更新自动刷新 |

#### 8.2 `relationship` 反向引用

```python
accounts: Mapped[list["Account"]] = relationship(back_populates="user")
```

- `back_populates="user"` 与 Account 模型中的 `user` 属性**双向绑定**
- 通过 `user.accounts` 可获取该用户的所有账户列表

---

### 九、账户模型

```python
class AccountType(str, enum.Enum):
    SAVINGS = "savings"
    CHECKING = "checking"

class Account(Base):
    __tablename__ = "accounts"

    user: Mapped["User"] = relationship(back_populates="accounts")
    outbound_transactions: Mapped[list["Transaction"]] = relationship(
        foreign_keys="Transaction.from_account_id", back_populates="from_account"
    )
    inbound_transactions: Mapped[list["Transaction"]] = relationship(
        foreign_keys="Transaction.to_account_id", back_populates="to_account"
    )
```

#### 9.1 枚举字段设计

```python
class AccountStatus(str, enum.Enum):
    ACTIVE = "active"     # 正常
    FROZEN = "frozen"     # 冻结（禁止出入账）
    CLOSED = "closed"     # 注销
```

| 设计要点 | 说明 |
|---------|------|
| 继承 `str` | 使得枚举值可以直接用于字符串比较和 JSON 序列化 |
| 值用小写 | 数据库中存 `"active"` 而非 `"ACTIVE"`，更易读 |
| 使用 `SAEnum` | SQLAlchemy 的 Enum 类型，自动映射 Python 枚举 ↔ 数据库值 |

#### 9.2 双向 Relationship（同一模型关联同一张表两次）

```python
outbound_transactions = relationship(
    foreign_keys="Transaction.from_account_id",  # ← 必须指定 foreign_keys
    back_populates="from_account"
)
inbound_transactions = relationship(
    foreign_keys="Transaction.to_account_id",
    back_populates="to_account"
)
```

> **为什么需要 `foreign_keys`？** Account 通过**两个不同外键**关联 Transaction（`from_account_id` 和 `to_account_id`），必须显式指定用哪个外键。

---

### 十、交易模型

```python
class TransactionStatus(str, enum.Enum):
    PENDING = "pending"         # 待处理
    PROCESSING = "processing"   # 处理中
    SUCCESS = "success"         # 成功
    FAILED = "failed"           # 失败
    REVERSED = "reversed"       # 已冲正

class TransactionType(str, enum.Enum):
    DEPOSIT = "deposit"
    WITHDRAWAL = "withdrawal"
    TRANSFER = "transfer"

class Transaction(Base):
    __tablename__ = "transactions"

    id: Mapped[str] = mapped_column(
        String(36), primary_key=True, default=lambda: str(uuid.uuid4())
    )
    transaction_no: Mapped[str] = mapped_column(
        String(32), unique=True, nullable=False, index=True,
        comment="对外交易流水号: TXN+时间戳+随机数"
    )
    idempotency_key: Mapped[str] = mapped_column(
        String(64), unique=True, nullable=False, index=True,
    )
    type: Mapped[TransactionType] = mapped_column(SAEnum(TransactionType))
    status: Mapped[TransactionStatus] = mapped_column(
        SAEnum(TransactionStatus), default=TransactionStatus.PENDING
    )
    from_account_id: Mapped[str | None] = mapped_column(
        String(36), nullable=True
    )
    to_account_id: Mapped[str | None] = mapped_column(
        String(36), nullable=True
    )
    amount: Mapped[Decimal] = mapped_column(
        DECIMAL(18, 2), nullable=False
    )
    fee: Mapped[Decimal] = mapped_column(
        DECIMAL(18, 2), default=Decimal("0.00")
    )
    currency: Mapped[str] = mapped_column(
        String(3), default="CNY"
    )
    description: Mapped[str | None] = mapped_column(
        Text, nullable=True
    )
    error_message: Mapped[str | None] = mapped_column(
        Text, nullable=True, comment="失败原因"
    )
    completed_at: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True), nullable=True
    )
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now()
    )

    from_account: Mapped["Account | None"] = relationship(
        foreign_keys=[from_account_id], back_populates="outbound_transactions"
    )
    to_account: Mapped["Account | None"] = relationship(
        foreign_keys=[to_account_id], back_populates="inbound_transactions"
    )

    __table_args__ = (
        Index("idx_from_account_created", "from_account_id", "created_at"),
        Index("idx_to_account_created", "to_account_id", "created_at"),
        Index("idx_status_created", "status", "created_at"),
    )
```

#### 10.1 幂等键设计

```python
idempotency_key: Mapped[str] = mapped_column(
    String(64), unique=True, nullable=False, index=True,
)
```

| 特性 | 说明 |
|------|------|
| `idempotency_key` | 交易唯一身份证，保证同一个请求无论执行多少次，最终结果都只生效一次 |
| `unique=True` | 数据库强制：同一个 idempotency_key 只能出现一次，从根源杜绝重复交易 |
| `nullable=False` | 强制每一笔交易必须携带幂等键 |
| `index=True` | 加速幂等键查询（每次交易前都要查） |
| 长度 64 | 足够容纳 UUID 或自定义格式 |

> 为什么交易系统必须用它？用户重复点击、网络超时重试、接口回调重复通知。没有它，你的账户系统一定会出现重复交易，这是金融系统的红线。

#### 10.2 交易状态机

```
PENDING ──→ PROCESSING ──→ SUCCESS
                    │
                    ├──→ FAILED
                    │       │
                    │       └──→ PROCESSING (重试)
                    │
                    └──→ REVERSED (冲正)
```

#### 10.3 复合索引

```python
__table_args__ = (
    Index("idx_from_account_created", "from_account_id", "created_at"),
    Index("idx_to_account_created", "to_account_id", "created_at"),
    Index("idx_status_created", "status", "created_at"),
)
```

| 索引 | 优化场景 |
|------|---------|
| `idx_from_account_created` | 查询某账户的所有**转出**记录，按时间排序 |
| `idx_to_account_created` | 查询某账户的所有**转入**记录，按时间排序 |
| `idx_status_created` | 查询某状态的交易列表（如"所有失败交易"） |

> 为什么都要 + created_at？因为交易表数据量会非常大！你永远不会查"一个账户的所有转出记录"，你只会查"一个账户最近 3 个月的转出记录"。时间范围 = 大数据表的必备筛选条件，联合索引 = 让范围查询飞起来。

---

### 十一、金额字段：DECIMAL vs FLOAT

#### 11.1 为什么 FLOAT 不能用？

```python
# FLOAT — 二进制无法精确表示十进制小数
0.1 + 0.2        # 0.30000000000000004 ← 不对！
0.1 + 0.2 == 0.3 # False ← 金融系统绝不能出现

# DECIMAL — 定点数，精确存储
Decimal("0.1") + Decimal("0.2")  # Decimal('0.3') ← 精确
Decimal("0.1") + Decimal("0.2") == Decimal("0.3")  # True
```

#### 11.2 类型对比

| 类型 | 存储方式 | 精度 | 适用场景 |
|------|---------|------|---------|
| `FLOAT` / `DOUBLE` | IEEE 754 二进制浮点数 | 近似值 | 科学计算、统计分析 |
| `DECIMAL(18,2)` | 定点数（整数部分 + 小数部分） | **精确** | 金额、财务数据 |

#### 11.3 DECIMAL 参数含义

```sql
DECIMAL(18, 2)
       ↑   ↑
   总位数 小数位数

-- 最大: 9999999999999999.99 (16位整数 + 2位小数)
-- 最小: -9999999999999999.99
```

> **金融铁律**：所有金额字段（余额、交易金额、手续费）一律使用 `DECIMAL`。

---

### 十二、乐观锁（Optimistic Locking）

#### 12.1 原理

```
时间线：  用户A                          用户B
─────────────────────────────────────────────────
T1      SELECT version=3, balance=1000
T2                                       SELECT version=3, balance=1000
T3      UPDATE ... SET balance=900,
          version=4 WHERE id=X AND version=3
        → rowcount=1 ✅ 成功
T4                                       UPDATE ... SET balance=800,
                                           version=4 WHERE id=X AND version=3
                                         → rowcount=0 ❌ 冲突！重试
```

#### 12.2 关键 SQL

```sql
-- 更新时带 version 条件
UPDATE accounts
SET balance = balance - 100,
    version = version + 1
WHERE id = ? AND version = ?   -- 如果 version 已变，rowcount=0
```

#### 12.3 乐观锁 vs 悲观锁

| 维度 | 乐观锁 | 悲观锁 (`SELECT ... FOR UPDATE`) |
|------|--------|------|
| 实现 | 版本号 / 时间戳 | 数据库行锁 |
| 冲突检测 | 提交时检测（rowcount=0） | 读取时锁定（阻塞其他事务） |
| 适用场景 | **读多写少**、冲突概率低 | **写多**、冲突概率高 |
| 性能 | 高（无锁等待） | 低（有锁等待和死锁风险） |
| 本项目用途 | 账户余额更新 | 转账时锁定账户行 |

> **本项目同时使用两种锁**：行级锁防止并发读到脏数据，乐观锁兜底检测并发冲突。

---

### 十三、`default` vs `server_default`

| 维度 | `default` | `server_default` |
|------|-----------|-------------------|
| **执行位置** | Python 侧 | 数据库侧 |
| **实现方式** | ORM 插入前调用 Python 函数 | SQL 的 `DEFAULT` 子句 |
| **适用场景** | UUID 生成等 Python 逻辑 | `NOW()`, `CURRENT_TIMESTAMP` |
| **裸 SQL 插入** | 不生效 | 生效 |

```python
# default — Python 侧生成
id = mapped_column(String(36), default=lambda: str(uuid.uuid4()))

# server_default — 数据库侧生成
created_at = mapped_column(DateTime, server_default=func.now())

# 两者可以同时使用
updated_at = mapped_column(
    DateTime,
    server_default=func.now(),  # INSERT 时数据库生成
    onupdate=func.now(),        # UPDATE 时数据库生成
)
```

> **为什么 UUID 用 `default` 而不是 `server_default`？** 因为 UUID 生成逻辑在 Python 中更灵活，且 ORM 插入后需要立即拿到 ID，`server_default` 需要额外 SELECT 才能获取。

---

### 十四、`Optional` 外键字段

```python
from_account_id: Mapped[str | None] = mapped_column(
    String(36), nullable=True  # 存款时为空（无转出方）
)
to_account_id: Mapped[str | None] = mapped_column(
    String(36), nullable=True  # 取款时为空（无转入方）
)
```

| 类型注解 | 含义 |
|---------|------|
| `str \| None` | Python 侧可为 None |
| `nullable=True` | 数据库侧允许 NULL |

> 存款交易只有 `to_account_id`（转入方），取款交易只有 `from_account_id`（转出方），转账交易两者都有。

---

### 十五、核心设计原则总结

| 原则 | 实现 | 说明 |
|------|------|------|
| **金额精确** | `DECIMAL(18,2)` | 绝不用 FLOAT |
| **幂等性** | `idempotency_key` UNIQUE | 数据库层面防重复 |
| **并发安全** | 乐观锁 `version` + 行级锁 | 双重保护 |
| **索引优化** | 复合索引 | 覆盖高频查询，减少回表 |
| **时间统一** | `server_default=func.now()` | 以数据库时间为准 |
| **枚举规范** | Python enum + SAEnum | 类型安全 + 数据库约束 |

---

### 常见问题

**Q1: 为什么需要 `asyncmy` 驱动？**
传统的 `pymysql` 是同步驱动，会阻塞事件循环。`asyncmy` 是纯异步的 MySQL 驱动，配合 `async_engine` 实现真正的异步 I/O。

**Q2: 连接池多大合适？**
经验公式：`pool_size ≈ 2 * CPU核心数`，`max_overflow ≈ pool_size * 2`。实际需要根据数据库服务器承载能力和 QPS 调整。

**Q3: `pool_pre_ping` 会影响性能吗？**
会有一点点影响（每次取连接多一次网络往返），但在 Web 应用中通常可以忽略。如果连接池回收策略得当（如 `pool_recycle` 设置为合理的值），可以关闭此选项。
