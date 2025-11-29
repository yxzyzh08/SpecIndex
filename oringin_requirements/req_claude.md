# SpecIndex：软件产品知识管理系统设计规范

> **Software Product Knowledge Management System**  
> 版本 1.0 | 核心模块

---

## 1. 系统定位

### 1.1 是什么

SpecIndex 是一个**软件产品知识管理系统**，用于结构化存储和查询软件产品的：

- 功能定义（Feature、UserStory、BusinessFlow）
- 技术结构（API、Component、Class、DataModel）
- 代码索引（File、FunctionSummary）
- 约束规则（Rule）
- 文档索引（Doc）

### 1.2 不是什么

本文档**不涉及**：

- AI集成与交互方式
- 开发工作流与审批流程
- 工作量证明机制
- 任务管理系统

这些属于上层模块，将在后续单独设计。

### 1.3 核心价值

| 价值 | 说明 |
|------|------|
| **结构化** | 用统一Schema管理软件知识，而非散落的文档 |
| **可查询** | 支持属性查询、全文搜索、图遍历 |
| **版本化** | 与Git集成，知识随代码分支同步 |
| **人类可读** | YAML格式，可直接编辑和审核Diff |

---

## 2. 双模态存储架构

### 2.1 架构概述

```
┌─────────────────────────────────────────────────────────────────┐
│                    持久化层（Source of Truth）                   │
│                         YAML + Git                              │
│  ────────────────────────────────────────────────────────────── │
│  • 人类可读、Diff友好                                            │
│  • Git天然版本控制，跟随代码分支                                 │
│  • 所有写操作的最终目标                                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ Index Syncer（单向同步）
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    运行时层（Runtime Index）                     │
│                    SQLite + NetworkX（内存）                     │
│  ────────────────────────────────────────────────────────────── │
│  • 复杂查询（SQL + 全文搜索）                                    │
│  • 图遍历算法（依赖分析、影响分析）                              │
│  • 衍生品，放入 .gitignore，可随时重建                           │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 为什么是双模态？

**问题**：纯文件查询慢，纯数据库无法跟随Git分支。

**解决**：

| 层 | 技术 | 职责 | Git管理 |
|----|------|------|---------|
| 持久化层 | YAML文件 | Source of Truth，人类可读 | ✅ 纳入版本控制 |
| 运行时层 | SQLite | 快速查询，全文搜索 | ❌ 放入.gitignore |
| 运行时层 | NetworkX | 内存图，图遍历算法 | ❌ 内存对象 |

### 2.3 数据流向

```
写入：外部系统 → YAML文件 → (Syncer) → SQLite
读取：外部系统 → Query Engine → SQLite / NetworkX
```

**铁律**：
- ✅ 写操作 → 写入 YAML 文件
- ✅ 读操作 → 读取 SQLite / NetworkX
- ❌ 绝不直接写 SQLite

---

## 3. 三层粒度模型

### 3.1 层级总览

| 层级 | 粒度 | 更新频率 | 节点类型 |
|------|------|----------|----------|
| **L1 概念层** | 功能/流程级 | 月级 | Feature, UserStory, BusinessFlow |
| **L2 结构层** | 接口/类/组件级 | 周级 | API, Component, Class, DataModel |
| **L3 实现层** | 函数/文件级 | 日级 | File, FunctionSummary |
| **跨层** | 约束/文档 | 按需 | Rule, Doc |

### 3.2 层级关系

```
L1 概念层（WHY：为什么做）
    │
    │ IMPLEMENTS（实现）
    ▼
L2 结构层（WHAT：做什么）  ← 核心层，80%维护精力
    │
    │ REALIZED_BY（落地于）
    ▼
L3 实现层（HOW：怎么做）

跨层：Rule（约束）、Doc（文档）
```

### 3.3 设计原则

| 原则 | 说明 |
|------|------|
| **L1不含代码** | 只有业务概念，无技术细节 |
| **L2是核心** | 稳定、结构化、最常查询 |
| **L3不存代码** | 只存结构化摘要，不存完整代码 |

---

## 4. 节点类型定义

### 4.1 节点总览

| 层级 | 节点类型 | 英文标识 | ID前缀 | 说明 |
|------|----------|----------|--------|------|
| L1 | 功能 | Feature | `feat_` | 产品核心功能单元 |
| L1 | 用户故事 | UserStory | `us_` | 用户视角的需求 |
| L1 | 业务流程 | BusinessFlow | `flow_` | 跨功能的业务路径 |
| L2 | 接口 | API | `api_` | HTTP接口定义 |
| L2 | 组件 | Component | `comp_` | 前端组件/后端服务 |
| L2 | 类 | Class | `class_` | 核心业务类 |
| L2 | 数据模型 | DataModel | `model_` | 数据库Schema |
| L3 | 文件 | File | `file_` | 源文件元信息 |
| L3 | 函数摘要 | FunctionSummary | `fn_` | 函数签名与副作用 |
| 跨层 | 规则 | Rule | `rule_` | 业务/技术/安全约束 |
| 跨层 | 文档 | Doc | `doc_` | 文档索引 |

### 4.2 L1 节点 Schema

#### Feature（功能）

```yaml
# .specindex/L1/features/feat_order.yaml

id: feat_order_001               # 唯一标识（必填）
type: Feature                    # 节点类型（必填）
title: 订单管理                   # 标题（必填）
description: |                   # 详细描述
  管理用户订单的创建、查询、修改、取消。
  包含订单状态机、库存扣减、价格计算等核心逻辑。

# 层级关系
parent_id: null                  # 父功能ID（可选）
children:                        # 子节点
  - us_order_create
  - us_order_cancel
  - us_order_query

# 依赖关系
depends_on:                      # 依赖的其他功能
  - feat_payment
  - feat_inventory

# 状态
status: implemented              # draft | reviewed | implemented | verified
priority: high                   # high | medium | low
owner: zhangsan                  # 负责人

# 关联
linked_apis:                     # 关联的API
  - api_create_order
  - api_cancel_order
linked_docs:                     # 关联的文档
  - doc_prd_order

# 元信息
created_at: "2024-01-15"
updated_at: "2024-01-20"
tags: [核心功能, 交易]
```

#### UserStory（用户故事）

```yaml
# .specindex/L1/user_stories/us_order_create.yaml

id: us_order_create
type: UserStory
title: 创建订单

# 用户故事格式
as_a: 买家                        # 作为...
i_want: 能够将购物车商品生成订单    # 我想要...
so_that: 可以进行支付完成购买      # 以便于...

# 验收标准
acceptance_criteria:
  - 选择商品后点击下单，生成订单
  - 订单包含商品、数量、价格、收货地址
  - 库存不足时提示并阻止下单
  - 订单创建后跳转到支付页面

# 关系
parent_feature: feat_order_001
implemented_by:                  # 实现此故事的L2节点
  - api_create_order
  - comp_order_form

# 状态
status: implemented
priority: high

# 元信息
created_at: "2024-01-15"
updated_at: "2024-01-18"
```

#### BusinessFlow（业务流程）

```yaml
# .specindex/L1/flows/flow_checkout.yaml

id: flow_checkout
type: BusinessFlow
title: 结账流程

description: 用户从购物车到完成支付的完整流程

# 流程步骤
steps:
  - order: 1
    name: 确认购物车
    feature: feat_cart
  - order: 2
    name: 创建订单
    feature: feat_order
  - order: 3
    name: 选择支付方式
    feature: feat_payment
  - order: 4
    name: 完成支付
    feature: feat_payment
  - order: 5
    name: 发送通知
    feature: feat_notification

# 关系
involves_features:
  - feat_cart
  - feat_order
  - feat_payment
  - feat_notification

triggers:                        # 触发的其他流程
  - flow_fulfillment

# 元信息
created_at: "2024-01-10"
updated_at: "2024-01-15"
```

### 4.3 L2 节点 Schema

#### API（接口）

```yaml
# .specindex/L2/apis/api_create_order.yaml

id: api_create_order
type: API
title: 创建订单接口

# 接口定义
path: /api/v1/orders
method: POST
summary: 创建新订单并扣减库存

# 输入契约
request:
  content_type: application/json
  schema:
    type: object
    required: [user_id, items, address_id]
    properties:
      user_id:
        type: string
        description: 用户ID
      items:
        type: array
        description: 订单商品列表
        items:
          type: object
          properties:
            product_id: { type: string }
            quantity: { type: integer, minimum: 1 }
            sku_id: { type: string }
      address_id:
        type: string
        description: 收货地址ID
      coupon_id:
        type: string
        description: 优惠券ID（可选）

# 输出契约
response:
  success:
    status: 201
    schema:
      type: object
      properties:
        order_id: { type: string }
        order_no: { type: string }
        total_amount: { type: number }
        status: { type: string, enum: [pending_payment] }
  errors:
    - status: 400
      code: INSUFFICIENT_STOCK
      message: 库存不足
    - status: 400
      code: INVALID_ADDRESS
      message: 收货地址无效

# 接口属性
auth_required: true              # 需要认证
idempotent: false                # 非幂等
rate_limit: 100/min              # 限流

# 关系
implements:                      # 实现的L1节点
  - us_order_create
depends_on:                      # 依赖的其他API
  - api_check_inventory
  - api_calculate_price
  - api_get_address
realized_by:                     # 实现此API的函数
  - fn_create_order
constrained_by:                  # 受约束的规则
  - rule_order_amount_limit
  - rule_order_item_limit

# 版本
version: "1.2"
deprecated: false

# 元信息
created_at: "2024-01-15"
updated_at: "2024-01-20"
```

#### Component（组件）

```yaml
# .specindex/L2/components/comp_order_form.yaml

id: comp_order_form
type: Component
title: 订单表单组件
category: frontend               # frontend | backend | shared

description: 订单确认页面的表单组件，展示商品、地址、支付方式

# 组件接口
props:
  - name: cartItems
    type: CartItem[]
    required: true
    description: 购物车商品列表
  - name: onSubmit
    type: "(order: OrderData) => void"
    required: true
    description: 提交回调

emits:
  - name: order-created
    payload: { order_id: string }

slots:
  - name: footer
    description: 底部自定义区域

# 关系
implements:
  - us_order_create
depends_on:
  - comp_address_selector
  - comp_payment_selector
  - api_create_order

# 文件位置
file_path: /src/components/order/OrderForm.vue

# 元信息
created_at: "2024-01-16"
updated_at: "2024-01-19"
```

#### Class（类）

```yaml
# .specindex/L2/classes/class_order_service.yaml

id: class_order_service
type: Class
title: 订单服务类
category: backend

description: 订单领域的核心服务类，处理订单创建、状态变更等业务逻辑

# 类定义
class_name: OrderService
file_path: /src/services/OrderService.ts

# 公开方法
methods:
  - name: createOrder
    visibility: public
    params:
      - name: userId
        type: string
      - name: items
        type: OrderItem[]
      - name: addressId
        type: string
    returns: Promise<Order>
    description: 创建订单
    
  - name: cancelOrder
    visibility: public
    params:
      - name: orderId
        type: string
      - name: reason
        type: string
    returns: Promise<boolean>
    description: 取消订单
    
  - name: getOrderStatus
    visibility: public
    params:
      - name: orderId
        type: string
    returns: Promise<OrderStatus>
    description: 获取订单状态

# 依赖注入
dependencies:
  - name: inventoryService
    type: InventoryService
  - name: paymentService
    type: PaymentService
  - name: orderRepository
    type: OrderRepository

# 关系
implements:
  - feat_order_001
realized_by:
  - fn_create_order
  - fn_cancel_order

# 元信息
created_at: "2024-01-15"
updated_at: "2024-01-20"
```

#### DataModel（数据模型）

```yaml
# .specindex/L2/data_models/model_order.yaml

id: model_order
type: DataModel
title: 订单数据模型
category: database               # database | domain | dto

description: 订单表的数据库Schema定义

# 表定义
table_name: orders
database: mysql

# 字段定义
fields:
  - name: id
    type: bigint
    primary: true
    auto_increment: true
    
  - name: order_no
    type: varchar(32)
    unique: true
    nullable: false
    comment: 订单编号
    
  - name: user_id
    type: varchar(64)
    nullable: false
    index: true
    comment: 用户ID
    
  - name: total_amount
    type: decimal(10,2)
    nullable: false
    comment: 订单总金额
    
  - name: status
    type: tinyint
    nullable: false
    default: 0
    comment: "状态：0-待支付 1-已支付 2-已发货 3-已完成 4-已取消"
    
  - name: address_snapshot
    type: json
    nullable: false
    comment: 收货地址快照
    
  - name: created_at
    type: datetime
    nullable: false
    default: CURRENT_TIMESTAMP
    
  - name: updated_at
    type: datetime
    nullable: false
    on_update: CURRENT_TIMESTAMP

# 索引
indexes:
  - name: idx_user_id
    columns: [user_id]
  - name: idx_order_no
    columns: [order_no]
    unique: true
  - name: idx_status_created
    columns: [status, created_at]

# 关系
belongs_to_feature: feat_order_001
used_by:
  - api_create_order
  - api_query_orders
  - class_order_service

# 元信息
created_at: "2024-01-10"
updated_at: "2024-01-15"
```

### 4.4 L3 节点 Schema

#### File（文件）

```yaml
# .specindex/L3/files/file_order_service_ts.yaml

id: file_order_service_ts
type: File
title: 订单服务文件

# 文件信息
path: /src/services/OrderService.ts
language: typescript
lines: 450

# 包含的函数
contains_functions:
  - fn_create_order
  - fn_cancel_order
  - fn_get_order_status

# 关系
realizes:
  - class_order_service

# 版本追踪
checksum: a1b2c3d4e5f6         # 文件hash，用于检测变更
last_scanned: "2024-01-20"

# 元信息
created_at: "2024-01-15"
updated_at: "2024-01-20"
```

#### FunctionSummary（函数摘要）

```yaml
# .specindex/L3/functions/fn_create_order.yaml

id: fn_create_order
type: FunctionSummary
name: createOrder

# 位置
file: /src/services/OrderService.ts
line_range: [45, 120]

# 语义描述（给外部系统读取）
purpose: |
  创建新订单的核心函数。
  流程：验证库存 → 计算价格 → 创建订单记录 → 扣减库存 → 发送事件。

# 类型签名
signature: "async createOrder(userId: string, items: OrderItem[], addressId: string): Promise<Order>"

inputs:
  - name: userId
    type: string
    required: true
    description: 用户唯一标识
  - name: items
    type: OrderItem[]
    required: true
    description: 订单商品列表
  - name: addressId
    type: string
    required: true
    description: 收货地址ID

output:
  type: Order
  nullable: false
  description: 创建成功的订单对象

# ⚠️ 副作用声明（关键）
side_effects:
  - type: DB_WRITE
    target: orders
    description: 插入订单记录
  - type: DB_WRITE
    target: order_items
    description: 插入订单商品记录
  - type: DB_WRITE
    target: inventory
    description: 扣减商品库存
  - type: EVENT_EMIT
    target: OrderCreatedEvent
    description: 发送订单创建事件
  - type: TRANSACTION
    description: 整个操作在数据库事务中执行

# 调用关系
calls:
  - fn_check_inventory
  - fn_calculate_price
  - fn_create_order_record
  - fn_deduct_inventory
  - fn_emit_order_event
  
called_by:
  - fn_checkout
  - fn_quick_buy

# 实现关系
realizes:
  - api_create_order

# 异常处理
throws:
  - type: InsufficientStockError
    condition: 库存不足时
  - type: InvalidAddressError
    condition: 地址无效时

# 测试覆盖
test_cases:
  - test_create_order_success
  - test_create_order_insufficient_stock
  - test_create_order_invalid_address

# 版本追踪
checksum: x1y2z3w4
last_verified: "2024-01-20"

# 元信息
created_at: "2024-01-15"
updated_at: "2024-01-20"
```

### 4.5 跨层节点 Schema

#### Rule（规则）

```yaml
# .specindex/rules/rule_order_amount_limit.yaml

id: rule_order_amount_limit
type: Rule
title: 订单金额限制

category: Business               # Business | Technical | Security

description: 订单总金额必须大于0，且单笔不超过100万

# 规则表达式
expression: "order.total_amount > 0 && order.total_amount <= 1000000"

# 违反时的处理
on_violation:
  action: reject                 # reject | warn | log
  message: 订单金额必须在0到100万之间

# 约束的节点
constrains:
  - api_create_order
  - api_update_order
  - fn_create_order

# 校验方式
verification: automated          # manual | automated | both

# 来源
source: "PRD v2.3 第4.2节"
source_link: "/docs/prd/order.md#4.2"

# 元信息
severity: error                  # error | warning | info
created_at: "2024-01-10"
updated_at: "2024-01-15"
```

#### Doc（文档）

```yaml
# .specindex/docs/doc_prd_order.yaml

id: doc_prd_order
type: Doc
title: 订单模块产品需求文档

category: PRD                    # PRD | Design | Test | API

# 文件信息
path: /docs/prd/order.md
format: markdown

# 内容摘要
summary: |
  定义订单的创建、查询、修改、取消流程及业务规则。
  包含订单状态机、退款规则、超时处理等。

# 章节索引
sections:
  - id: "4.1"
    title: 订单创建
    anchor: "#41-订单创建"
  - id: "4.2"
    title: 金额规则
    anchor: "#42-金额规则"
  - id: "4.3"
    title: 订单状态机
    anchor: "#43-订单状态机"

# 关联
documents:
  - feat_order_001
  - api_create_order
  - api_cancel_order

# 版本
version: "2.3"
checksum: e5f6g7h8
last_updated: "2024-01-18"

# 变更历史
changelog:
  - version: "2.3"
    date: "2024-01-18"
    changes: 新增订单取消的退款规则
  - version: "2.2"
    date: "2024-01-10"
    changes: 补充订单状态机定义

# 元信息
created_at: "2024-01-05"
updated_at: "2024-01-18"
```

### 4.6 副作用类型枚举

| 类型 | 说明 | 风险等级 |
|------|------|----------|
| `DB_READ` | 读取数据库 | 🟢 低 |
| `DB_WRITE` | 写入数据库 | 🔴 高 |
| `CACHE_READ` | 读取缓存 | 🟢 低 |
| `CACHE_WRITE` | 写入缓存 | 🟡 中 |
| `EVENT_EMIT` | 发送事件/消息 | 🟡 中 |
| `HTTP_CALL` | 发起外部HTTP请求 | 🔴 高 |
| `FILE_READ` | 读取文件 | 🟢 低 |
| `FILE_WRITE` | 写入文件 | 🟡 中 |
| `STATE_MUTATION` | 修改全局/共享状态 | 🔴 高 |
| `TRANSACTION` | 开启数据库事务 | 🔴 高 |

---

## 5. 边（关系）类型定义

### 5.1 关系总览

| 边类型 | 方向 | 语义 | 示例 |
|--------|------|------|------|
| `CONTAINS` | 父→子 | 包含/组成 | Feature → UserStory |
| `DEPENDS_ON` | A→B | A依赖B | API_A → API_B |
| `TRIGGERS` | A→B | A触发B | Flow_A → Flow_B |
| `IMPLEMENTS` | L2→L1 | 实现 | API → UserStory |
| `REALIZED_BY` | L3→L2 | 落地于 | Function → API |
| `DOCUMENTS` | Doc→节点 | 描述 | Doc → Feature |
| `CONSTRAINED_BY` | 节点→Rule | 受约束 | API → Rule |

### 5.2 关系的YAML表示

关系在节点YAML文件中以字段形式声明：

```yaml
# 在 API 节点中
id: api_create_order

# 向上实现（IMPLEMENTS）
implements:
  - us_order_create

# 同层依赖（DEPENDS_ON）
depends_on:
  - api_check_inventory
  - api_calculate_price

# 向下落地（REALIZED_BY）
realized_by:
  - fn_create_order

# 受约束（CONSTRAINED_BY）
constrained_by:
  - rule_order_amount_limit
```

### 5.3 关系字段映射表

| YAML字段 | 边类型 | 方向 |
|----------|--------|------|
| `children` | CONTAINS | 当前→目标 |
| `parent_id` / `parent_feature` | CONTAINS | 目标→当前 |
| `depends_on` | DEPENDS_ON | 当前→目标 |
| `triggers` | TRIGGERS | 当前→目标 |
| `implements` | IMPLEMENTS | 当前→目标 |
| `realized_by` | REALIZED_BY | 目标→当前 |
| `realizes` | REALIZED_BY | 当前→目标 |
| `documents` | DOCUMENTS | 当前→目标 |
| `linked_docs` | DOCUMENTS | 目标→当前 |
| `constrained_by` | CONSTRAINED_BY | 当前→目标 |
| `constrains` | CONSTRAINED_BY | 目标→当前 |

---

## 6. 目录结构

### 6.1 完整目录结构

```
your-project/
├── src/                          # 源代码
├── docs/                         # 项目文档
│
├── .specindex/                   # 📁 知识图谱根目录
│   │
│   ├── schema/                   # Schema定义（校验用）
│   │   ├── feature.schema.json
│   │   ├── api.schema.json
│   │   ├── function.schema.json
│   │   └── ...
│   │
│   ├── L1/                       # 概念层
│   │   ├── features/
│   │   │   ├── feat_order.yaml
│   │   │   └── feat_payment.yaml
│   │   ├── user_stories/
│   │   │   └── us_order_create.yaml
│   │   └── flows/
│   │       └── flow_checkout.yaml
│   │
│   ├── L2/                       # 结构层
│   │   ├── apis/
│   │   │   └── api_create_order.yaml
│   │   ├── components/
│   │   │   └── comp_order_form.yaml
│   │   ├── classes/
│   │   │   └── class_order_service.yaml
│   │   └── data_models/
│   │       └── model_order.yaml
│   │
│   ├── L3/                       # 实现层
│   │   ├── files/
│   │   │   └── file_order_service_ts.yaml
│   │   └── functions/
│   │       └── fn_create_order.yaml
│   │
│   ├── rules/                    # 规则
│   │   └── rule_order_amount_limit.yaml
│   │
│   ├── docs/                     # 文档索引
│   │   └── doc_prd_order.yaml
│   │
│   └── .runtime/                 # ⚠️ 运行时（.gitignore）
│       ├── index.db              # SQLite数据库
│       └── sync.meta             # 同步元数据
│
├── specindex.config.yaml         # 配置文件
└── .gitignore
```

### 6.2 .gitignore 配置

```gitignore
# SpecIndex 运行时文件（衍生品，可重建）
.specindex/.runtime/
```

### 6.3 配置文件

```yaml
# specindex.config.yaml

# 基础配置
spec_root: .specindex
runtime_dir: .specindex/.runtime

# 数据库配置
database:
  path: .specindex/.runtime/index.db
  
# 同步配置
sync:
  auto_sync: true                # 启动时自动同步
  watch_changes: true            # 监听文件变更
  
# Schema校验
validation:
  enabled: true
  strict: false                  # 严格模式：未知字段报错
  
# 日志
logging:
  level: info
  file: .specindex/.runtime/specindex.log
```

---

## 7. SQLite 索引层

### 7.1 表结构

```sql
-- ========== 节点表 ==========
CREATE TABLE nodes (
    id TEXT PRIMARY KEY,              -- feat_order_001
    type TEXT NOT NULL,               -- Feature / API / FunctionSummary
    layer TEXT NOT NULL,              -- L1 / L2 / L3 / Rule / Doc
    
    -- 源文件信息
    file_path TEXT NOT NULL,
    file_hash TEXT NOT NULL,          -- 用于增量同步
    
    -- 完整内容
    content JSON NOT NULL,
    
    -- 冗余字段（加速查询）
    title TEXT,
    status TEXT,
    priority TEXT,
    category TEXT,
    
    -- 全文搜索文本
    search_text TEXT,
    
    -- 时间戳
    created_at TEXT,
    updated_at TEXT,
    synced_at TEXT DEFAULT CURRENT_TIMESTAMP
);

-- ========== 边表 ==========
CREATE TABLE edges (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    source_id TEXT NOT NULL,
    target_id TEXT NOT NULL,
    relation TEXT NOT NULL,           -- DEPENDS_ON / IMPLEMENTS / ...
    
    UNIQUE(source_id, target_id, relation),
    FOREIGN KEY (source_id) REFERENCES nodes(id) ON DELETE CASCADE,
    FOREIGN KEY (target_id) REFERENCES nodes(id) ON DELETE CASCADE
);

-- ========== 索引 ==========
CREATE INDEX idx_nodes_type ON nodes(type);
CREATE INDEX idx_nodes_layer ON nodes(layer);
CREATE INDEX idx_nodes_status ON nodes(status);

CREATE INDEX idx_edges_source ON edges(source_id);
CREATE INDEX idx_edges_target ON edges(target_id);
CREATE INDEX idx_edges_relation ON edges(relation);

-- ========== 全文搜索 ==========
CREATE VIRTUAL TABLE nodes_fts USING fts5(
    id, title, search_text,
    content='nodes'
);
```

### 7.2 常用查询

```sql
-- 按类型查询
SELECT * FROM nodes WHERE type = 'API' AND layer = 'L2';

-- 查询某Feature的所有实现
SELECT n.* FROM nodes n
JOIN edges e ON n.id = e.source_id
WHERE e.target_id = 'feat_order_001' AND e.relation = 'IMPLEMENTS';

-- 查询某API的依赖
SELECT n.* FROM nodes n
JOIN edges e ON n.id = e.target_id
WHERE e.source_id = 'api_create_order' AND e.relation = 'DEPENDS_ON';

-- 全文搜索
SELECT * FROM nodes WHERE id IN (
    SELECT id FROM nodes_fts WHERE nodes_fts MATCH '订单'
);

-- 查询有DB_WRITE副作用的函数
SELECT * FROM nodes 
WHERE type = 'FunctionSummary'
  AND json_extract(content, '$.side_effects') LIKE '%DB_WRITE%';
```

---

## 8. Index Syncer 同步器

### 8.1 职责

- 启动时：扫描YAML文件，同步到SQLite
- 运行时：监听文件变更，增量同步
- 分支切换时：检测并全量重建

### 8.2 核心实现

```python
"""Index Syncer：YAML → SQLite 单向同步器"""

import yaml
import sqlite3
import hashlib
import json
from pathlib import Path
from typing import Dict, List, Optional

class IndexSyncer:
    
    # 关系字段 → 边类型的映射
    RELATION_FIELDS = {
        # 正向关系（source → target）
        'depends_on': ('DEPENDS_ON', 'forward'),
        'implements': ('IMPLEMENTS', 'forward'),
        'triggers': ('TRIGGERS', 'forward'),
        'documents': ('DOCUMENTS', 'forward'),
        'constrained_by': ('CONSTRAINED_BY', 'forward'),
        'realizes': ('REALIZED_BY', 'forward'),
        'children': ('CONTAINS', 'forward'),
        
        # 反向关系（target → source）
        'realized_by': ('REALIZED_BY', 'reverse'),
        'parent_id': ('CONTAINS', 'reverse'),
        'parent_feature': ('CONTAINS', 'reverse'),
        'linked_docs': ('DOCUMENTS', 'reverse'),
        'linked_apis': ('CONTAINS', 'reverse'),
        'constrains': ('CONSTRAINED_BY', 'reverse'),
    }
    
    def __init__(self, spec_root: str, db_path: str):
        self.spec_root = Path(spec_root)
        self.db_path = db_path
        self.conn = sqlite3.connect(db_path)
        self._init_schema()
    
    def _init_schema(self):
        """初始化数据库Schema"""
        self.conn.executescript("""
            CREATE TABLE IF NOT EXISTS nodes (
                id TEXT PRIMARY KEY,
                type TEXT NOT NULL,
                layer TEXT NOT NULL,
                file_path TEXT NOT NULL,
                file_hash TEXT NOT NULL,
                content JSON NOT NULL,
                title TEXT,
                status TEXT,
                priority TEXT,
                category TEXT,
                search_text TEXT,
                created_at TEXT,
                updated_at TEXT,
                synced_at TEXT DEFAULT CURRENT_TIMESTAMP
            );
            
            CREATE TABLE IF NOT EXISTS edges (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                source_id TEXT NOT NULL,
                target_id TEXT NOT NULL,
                relation TEXT NOT NULL,
                UNIQUE(source_id, target_id, relation)
            );
            
            CREATE INDEX IF NOT EXISTS idx_nodes_type ON nodes(type);
            CREATE INDEX IF NOT EXISTS idx_nodes_layer ON nodes(layer);
            CREATE INDEX IF NOT EXISTS idx_edges_source ON edges(source_id);
            CREATE INDEX IF NOT EXISTS idx_edges_target ON edges(target_id);
        """)
        self.conn.commit()
    
    def sync(self, force: bool = False):
        """
        同步YAML到SQLite
        
        Args:
            force: True=全量重建，False=增量同步
        """
        if force:
            self.conn.execute("DELETE FROM nodes")
            self.conn.execute("DELETE FROM edges")
        
        # 扫描所有YAML文件
        yaml_files = list(self.spec_root.glob("**/*.yaml"))
        yaml_files = [f for f in yaml_files if '.runtime' not in str(f)]
        
        synced_paths = set()
        
        for yaml_file in yaml_files:
            if self._sync_file(yaml_file):
                synced_paths.add(str(yaml_file))
        
        # 清理已删除的节点
        self._remove_orphans(synced_paths)
        
        # 重建边表
        self._rebuild_edges()
        
        self.conn.commit()
    
    def _sync_file(self, yaml_file: Path) -> bool:
        """同步单个文件，返回是否成功"""
        try:
            file_hash = self._hash_file(yaml_file)
            
            # 检查是否需要更新
            existing = self.conn.execute(
                "SELECT file_hash FROM nodes WHERE file_path = ?",
                (str(yaml_file),)
            ).fetchone()
            
            if existing and existing[0] == file_hash:
                return True  # 无变更
            
            # 解析YAML
            with open(yaml_file, 'r', encoding='utf-8') as f:
                data = yaml.safe_load(f)
            
            if not data or 'id' not in data:
                return False
            
            # 构建搜索文本
            search_text = self._build_search_text(data)
            
            # 插入或更新
            self.conn.execute("""
                INSERT OR REPLACE INTO nodes 
                (id, type, layer, file_path, file_hash, content,
                 title, status, priority, category, search_text,
                 created_at, updated_at)
                VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
            """, (
                data['id'],
                data.get('type', ''),
                self._detect_layer(yaml_file),
                str(yaml_file),
                file_hash,
                json.dumps(data, ensure_ascii=False),
                data.get('title', ''),
                data.get('status', ''),
                data.get('priority', ''),
                data.get('category', ''),
                search_text,
                data.get('created_at', ''),
                data.get('updated_at', '')
            ))
            
            return True
            
        except Exception as e:
            print(f"Error syncing {yaml_file}: {e}")
            return False
    
    def _rebuild_edges(self):
        """从节点内容重建边表"""
        self.conn.execute("DELETE FROM edges")
        
        for row in self.conn.execute("SELECT id, content FROM nodes"):
            node_id, content_str = row
            content = json.loads(content_str)
            
            for field, (relation, direction) in self.RELATION_FIELDS.items():
                targets = content.get(field, [])
                
                if targets is None:
                    continue
                if isinstance(targets, str):
                    targets = [targets]
                if not isinstance(targets, list):
                    continue
                
                for target_id in targets:
                    if not target_id:
                        continue
                    
                    if direction == 'forward':
                        src, tgt = node_id, target_id
                    else:
                        src, tgt = target_id, node_id
                    
                    try:
                        self.conn.execute(
                            "INSERT OR IGNORE INTO edges (source_id, target_id, relation) VALUES (?, ?, ?)",
                            (src, tgt, relation)
                        )
                    except Exception:
                        pass  # 忽略无效边
    
    def _hash_file(self, path: Path) -> str:
        return hashlib.md5(path.read_bytes()).hexdigest()
    
    def _detect_layer(self, path: Path) -> str:
        path_str = str(path)
        if '/L1/' in path_str:
            return 'L1'
        elif '/L2/' in path_str:
            return 'L2'
        elif '/L3/' in path_str:
            return 'L3'
        elif '/rules/' in path_str:
            return 'Rule'
        elif '/docs/' in path_str:
            return 'Doc'
        return 'Unknown'
    
    def _build_search_text(self, data: dict) -> str:
        parts = [
            str(data.get('id', '')),
            str(data.get('title', '')),
            str(data.get('description', '')),
            str(data.get('summary', '')),
            str(data.get('purpose', '')),
        ]
        return ' '.join(filter(None, parts))
    
    def _remove_orphans(self, current_paths: set):
        """删除已不存在的文件对应的节点"""
        for row in self.conn.execute("SELECT id, file_path FROM nodes").fetchall():
            if row[1] not in current_paths:
                self.conn.execute("DELETE FROM nodes WHERE id = ?", (row[0],))
```

---

## 9. Query Engine 查询引擎

### 9.1 架构

```
QueryEngine
├── SQLite查询：属性查询、全文搜索
└── NetworkX查询：图遍历、依赖分析
```

### 9.2 核心实现

```python
"""Query Engine：统一查询接口"""

import sqlite3
import json
import networkx as nx
from typing import List, Dict, Optional

class QueryEngine:
    
    def __init__(self, db_path: str):
        self.db = sqlite3.connect(db_path)
        self.db.row_factory = sqlite3.Row
        self._graph: Optional[nx.DiGraph] = None
    
    # ========== 节点查询 ==========
    
    def get_node(self, node_id: str) -> Optional[Dict]:
        """获取单个节点"""
        row = self.db.execute(
            "SELECT * FROM nodes WHERE id = ?", (node_id,)
        ).fetchone()
        
        if row:
            result = dict(row)
            result['content'] = json.loads(result['content'])
            return result
        return None
    
    def list_nodes(self,
                   node_type: str = None,
                   layer: str = None,
                   status: str = None,
                   limit: int = 100) -> List[Dict]:
        """按条件查询节点"""
        conditions = ["1=1"]
        params = []
        
        if node_type:
            conditions.append("type = ?")
            params.append(node_type)
        if layer:
            conditions.append("layer = ?")
            params.append(layer)
        if status:
            conditions.append("status = ?")
            params.append(status)
        
        query = f"SELECT * FROM nodes WHERE {' AND '.join(conditions)} LIMIT ?"
        params.append(limit)
        
        rows = self.db.execute(query, params).fetchall()
        
        results = []
        for row in rows:
            r = dict(row)
            r['content'] = json.loads(r['content'])
            results.append(r)
        
        return results
    
    def search(self, query: str, limit: int = 20) -> List[Dict]:
        """全文搜索"""
        rows = self.db.execute("""
            SELECT * FROM nodes 
            WHERE search_text LIKE ? OR title LIKE ?
            LIMIT ?
        """, (f'%{query}%', f'%{query}%', limit)).fetchall()
        
        results = []
        for row in rows:
            r = dict(row)
            r['content'] = json.loads(r['content'])
            results.append(r)
        
        return results
    
    # ========== 关系查询 ==========
    
    def get_related(self,
                    node_id: str,
                    relation: str = None,
                    direction: str = 'out') -> List[Dict]:
        """
        获取关联节点
        
        Args:
            node_id: 节点ID
            relation: 关系类型（可选）
            direction: 'out'=出边, 'in'=入边, 'both'=双向
        """
        conditions = []
        params = []
        
        if direction in ('out', 'both'):
            conditions.append("source_id = ?")
            params.append(node_id)
        if direction in ('in', 'both'):
            conditions.append("target_id = ?")
            params.append(node_id)
        
        where = " OR ".join(conditions)
        
        if relation:
            where = f"({where}) AND relation = ?"
            params.append(relation)
        
        edges = self.db.execute(
            f"SELECT * FROM edges WHERE {where}", params
        ).fetchall()
        
        # 获取关联节点
        related_ids = set()
        for edge in edges:
            if edge['source_id'] != node_id:
                related_ids.add(edge['source_id'])
            if edge['target_id'] != node_id:
                related_ids.add(edge['target_id'])
        
        return [self.get_node(nid) for nid in related_ids if self.get_node(nid)]
    
    # ========== 图遍历（NetworkX） ==========
    
    def _ensure_graph(self):
        """懒加载图"""
        if self._graph is not None:
            return
        
        self._graph = nx.DiGraph()
        
        for row in self.db.execute("SELECT id, type, layer FROM nodes"):
            self._graph.add_node(row[0], type=row[1], layer=row[2])
        
        for row in self.db.execute("SELECT source_id, target_id, relation FROM edges"):
            self._graph.add_edge(row[0], row[1], relation=row[2])
    
    def get_dependencies(self,
                         node_id: str,
                         relation: str = 'DEPENDS_ON',
                         depth: int = 3) -> Dict:
        """获取依赖树（向外遍历）"""
        self._ensure_graph()
        
        result = {"root": node_id, "dependencies": {}, "depth": depth}
        visited = set()
        queue = [(node_id, 0)]
        
        while queue:
            current, current_depth = queue.pop(0)
            
            if current in visited or current_depth > depth:
                continue
            visited.add(current)
            
            deps = []
            if current in self._graph:
                for _, target, data in self._graph.out_edges(current, data=True):
                    if relation is None or data.get('relation') == relation:
                        deps.append(target)
                        if target not in visited:
                            queue.append((target, current_depth + 1))
            
            if deps:
                result["dependencies"][current] = deps
        
        return result
    
    def get_dependents(self,
                       node_id: str,
                       relation: str = 'DEPENDS_ON',
                       depth: int = 3) -> Dict:
        """获取被依赖树（向内遍历）：谁依赖了我"""
        self._ensure_graph()
        
        result = {"root": node_id, "dependents": {}, "depth": depth}
        visited = set()
        queue = [(node_id, 0)]
        
        while queue:
            current, current_depth = queue.pop(0)
            
            if current in visited or current_depth > depth:
                continue
            visited.add(current)
            
            deps = []
            if current in self._graph:
                for source, _, data in self._graph.in_edges(current, data=True):
                    if relation is None or data.get('relation') == relation:
                        deps.append(source)
                        if source not in visited:
                            queue.append((source, current_depth + 1))
            
            if deps:
                result["dependents"][current] = deps
        
        return result
    
    def get_impact_analysis(self, node_id: str) -> Dict:
        """影响分析：修改此节点会影响哪些节点"""
        self._ensure_graph()
        
        # BFS查找所有入边可达节点
        impacted = set()
        queue = [node_id]
        
        while queue:
            current = queue.pop(0)
            if current in self._graph:
                for source, _ in self._graph.in_edges(current):
                    if source not in impacted:
                        impacted.add(source)
                        queue.append(source)
        
        # 按层级分类
        result = {
            "node": node_id,
            "total_impact": len(impacted),
            "by_layer": {"L1": [], "L2": [], "L3": [], "Other": []}
        }
        
        for nid in impacted:
            if nid in self._graph.nodes:
                layer = self._graph.nodes[nid].get('layer', 'Other')
                key = layer if layer in result["by_layer"] else "Other"
                result["by_layer"][key].append(nid)
        
        return result
    
    def reload_graph(self):
        """强制重新加载图"""
        self._graph = None
```

---

## 10. API 设计

### 10.1 API 总览

```
SpecIndex API
├── 节点操作
│   ├── get_node(id) → Node
│   ├── list_nodes(filters) → List[Node]
│   ├── create_node(data) → Node
│   ├── update_node(id, data) → Node
│   └── delete_node(id) → bool
│
├── 搜索
│   ├── search(query) → List[Node]
│   └── search_by_field(field, value) → List[Node]
│
├── 关系查询
│   ├── get_related(id, relation, direction) → List[Node]
│   ├── get_dependencies(id, depth) → DependencyTree
│   ├── get_dependents(id, depth) → DependentTree
│   └── get_impact(id) → ImpactAnalysis
│
└── 系统操作
    ├── sync(force) → SyncResult
    └── validate(id) → ValidationResult
```

### 10.2 API 实现

```python
"""SpecIndex API：对外统一接口"""

from pathlib import Path
from typing import List, Dict, Optional
import yaml
from datetime import datetime

class SpecIndexAPI:
    
    def __init__(self, spec_root: str):
        self.spec_root = Path(spec_root)
        self.runtime_dir = self.spec_root / '.runtime'
        self.runtime_dir.mkdir(parents=True, exist_ok=True)
        
        db_path = str(self.runtime_dir / 'index.db')
        
        self.syncer = IndexSyncer(str(spec_root), db_path)
        self.query = QueryEngine(db_path)
        
        # 启动时同步
        self.syncer.sync()
    
    # ========== 节点操作 ==========
    
    def get_node(self, node_id: str) -> Optional[Dict]:
        """获取节点"""
        return self.query.get_node(node_id)
    
    def list_nodes(self,
                   node_type: str = None,
                   layer: str = None,
                   status: str = None,
                   limit: int = 100) -> List[Dict]:
        """列出节点"""
        return self.query.list_nodes(node_type, layer, status, limit)
    
    def create_node(self, 
                    node_type: str,
                    data: Dict) -> Dict:
        """
        创建节点
        
        写入YAML文件 → 触发同步 → 返回节点
        """
        if 'id' not in data:
            raise ValueError("Node must have an 'id' field")
        
        if 'type' not in data:
            data['type'] = node_type
        
        # 确定文件路径
        file_path = self._get_file_path(node_type, data['id'])
        file_path.parent.mkdir(parents=True, exist_ok=True)
        
        # 添加时间戳
        now = datetime.now().isoformat()
        data.setdefault('created_at', now)
        data['updated_at'] = now
        
        # 写入YAML
        with open(file_path, 'w', encoding='utf-8') as f:
            yaml.dump(data, f, allow_unicode=True, sort_keys=False)
        
        # 同步到SQLite
        self.syncer._sync_file(file_path)
        self.syncer._rebuild_edges()
        self.syncer.conn.commit()
        
        # 刷新图
        self.query._graph = None
        
        return self.get_node(data['id'])
    
    def update_node(self, 
                    node_id: str, 
                    updates: Dict) -> Optional[Dict]:
        """
        更新节点
        """
        node = self.get_node(node_id)
        if not node:
            return None
        
        file_path = Path(node['file_path'])
        
        # 合并更新
        content = node['content']
        content.update(updates)
        content['updated_at'] = datetime.now().isoformat()
        
        # 写入YAML
        with open(file_path, 'w', encoding='utf-8') as f:
            yaml.dump(content, f, allow_unicode=True, sort_keys=False)
        
        # 同步
        self.syncer._sync_file(file_path)
        self.syncer._rebuild_edges()
        self.syncer.conn.commit()
        
        self.query._graph = None
        
        return self.get_node(node_id)
    
    def delete_node(self, node_id: str) -> bool:
        """删除节点"""
        node = self.get_node(node_id)
        if not node:
            return False
        
        file_path = Path(node['file_path'])
        
        if file_path.exists():
            file_path.unlink()
        
        # 从数据库删除
        self.syncer.conn.execute("DELETE FROM nodes WHERE id = ?", (node_id,))
        self.syncer._rebuild_edges()
        self.syncer.conn.commit()
        
        self.query._graph = None
        
        return True
    
    # ========== 搜索 ==========
    
    def search(self, query: str, limit: int = 20) -> List[Dict]:
        """全文搜索"""
        return self.query.search(query, limit)
    
    # ========== 关系查询 ==========
    
    def get_related(self,
                    node_id: str,
                    relation: str = None,
                    direction: str = 'out') -> List[Dict]:
        """获取关联节点"""
        return self.query.get_related(node_id, relation, direction)
    
    def get_dependencies(self,
                         node_id: str,
                         depth: int = 3) -> Dict:
        """获取依赖树"""
        return self.query.get_dependencies(node_id, depth=depth)
    
    def get_dependents(self,
                       node_id: str,
                       depth: int = 3) -> Dict:
        """获取被依赖树"""
        return self.query.get_dependents(node_id, depth=depth)
    
    def get_impact(self, node_id: str) -> Dict:
        """影响分析"""
        return self.query.get_impact_analysis(node_id)
    
    # ========== 系统操作 ==========
    
    def sync(self, force: bool = False) -> Dict:
        """
        同步YAML到SQLite
        
        Args:
            force: True=全量重建
        """
        self.syncer.sync(force)
        self.query._graph = None
        
        count = self.syncer.conn.execute("SELECT COUNT(*) FROM nodes").fetchone()[0]
        
        return {
            "success": True,
            "node_count": count,
            "force": force
        }
    
    # ========== 辅助方法 ==========
    
    def _get_file_path(self, node_type: str, node_id: str) -> Path:
        """根据类型确定文件路径"""
        type_mapping = {
            'Feature': ('L1', 'features'),
            'UserStory': ('L1', 'user_stories'),
            'BusinessFlow': ('L1', 'flows'),
            'API': ('L2', 'apis'),
            'Component': ('L2', 'components'),
            'Class': ('L2', 'classes'),
            'DataModel': ('L2', 'data_models'),
            'File': ('L3', 'files'),
            'FunctionSummary': ('L3', 'functions'),
            'Rule': ('rules', ''),
            'Doc': ('docs', ''),
        }
        
        layer, folder = type_mapping.get(node_type, ('L2', 'misc'))
        
        if folder:
            return self.spec_root / layer / folder / f"{node_id}.yaml"
        else:
            return self.spec_root / layer / f"{node_id}.yaml"
```

### 10.3 使用示例

```python
# 初始化
api = SpecIndexAPI('.specindex')

# 创建节点
api.create_node('Feature', {
    'id': 'feat_payment',
    'title': '支付功能',
    'description': '处理订单支付',
    'status': 'draft'
})

# 查询节点
node = api.get_node('feat_order_001')
print(node['content']['title'])

# 列出所有API
apis = api.list_nodes(node_type='API', layer='L2')

# 搜索
results = api.search('订单')

# 获取依赖
deps = api.get_dependencies('api_create_order', depth=2)

# 影响分析
impact = api.get_impact('model_order')
print(f"修改会影响 {impact['total_impact']} 个节点")

# 更新节点
api.update_node('feat_order_001', {'status': 'implemented'})

# 全量同步
api.sync(force=True)
```

---

## 11. 技术栈

| 组件 | 技术 | 说明 |
|------|------|------|
| 语言 | Python 3.10+ | |
| 持久化 | YAML | PyYAML |
| 数据库 | SQLite | 标准库 sqlite3 |
| 图算法 | NetworkX | 内存图计算 |
| Schema校验 | Pydantic（可选） | 类型安全 |
| 文件监听 | watchdog（可选） | 实时同步 |

---

## 12. 实现计划

| 阶段 | 内容 | 时间 | 产出 |
|------|------|------|------|
| Week 1 | Schema设计 + YAML模板 | 3天 | 11种节点模板 |
| Week 2 | Index Syncer | 2天 | ~150行 |
| Week 3 | Query Engine | 2天 | ~200行 |
| Week 4 | SpecIndex API | 2天 | ~200行 |
| Week 5 | 测试 + 文档 | 2天 | 完整MVP |

**总计**：约 550 行核心代码，5 周完成 MVP

---

## 附录：快速参考

### 节点类型速查

```
L1（概念层）：Feature, UserStory, BusinessFlow
L2（结构层）：API, Component, Class, DataModel
L3（实现层）：File, FunctionSummary
跨层：Rule, Doc
```

### 关系类型速查

```
层内：CONTAINS, DEPENDS_ON, TRIGGERS
层间：IMPLEMENTS, REALIZED_BY
跨层：DOCUMENTS, CONSTRAINED_BY
```

### ID前缀速查

```
feat_  → Feature
us_    → UserStory
flow_  → BusinessFlow
api_   → API
comp_  → Component
class_ → Class
model_ → DataModel
file_  → File
fn_    → FunctionSummary
rule_  → Rule
doc_   → Doc
```

---

*文档版本：v1.0 | 软件产品知识管理系统核心模块*
