# 📘 SpecIndex 系统架构设计规范 (Unified v4.0)

> **Software Product Knowledge Management System**
> **定位**：面向 AI 原生开发的无头语义数据库与治理平台

---

## 1. 系统定义 (System Definition)

### 1.1 核心理念
SpecIndex 是一个**“Git 原生双模态知识库”**。它将软件产品的知识结构化为图谱，并提供治理机制。
*   **对于 Git**：它是标准的 YAML 文件集合（Source of Truth）。
*   **对于 AI**：它是极速响应的语义 API（Runtime Cache）。
*   **对于人类**：它是防止架构腐化的“立法机构”（Governance）。

### 1.2 系统边界
*   ✅ **包含**：数据存储（YAML/SQLite）、图谱结构定义、读写 API、提案审核机制、同步器。
*   ❌ **不包含**：AI Agent 的思考逻辑、IDE 插件 UI、任务调度。

---

## 2. 存储架构：双模态 + 提案缓冲

我们采用 **“冷热分离 + 写入缓冲”** 的三级架构。

```mermaid
graph TD
    User[超级个体/AI] -->|1. Query| API
    User -->|2. Propose| ProposalMgr
    
    subgraph SIS [SpecIndex System]
        API[API Gateway]
        ProposalMgr[Proposal Manager]
        Syncer[Index Syncer]
        
        subgraph L1 [Layer 1: Governance]
            Pending[Pending Proposals]
            DiffEng[Diff Engine]
        end
        
        subgraph L2 [Layer 2: Truth (Git)]
            YAML[YAML Files]
        end
        
        subgraph L3 [Layer 3: Runtime (Cache)]
            SQLite[(SQLite DB)]
            NetworkX[Memory Graph]
        end
        
        ProposalMgr -->|Write JSON| Pending
        Pending -->|Human Approve| YAML
        YAML -->|Sync| Syncer
        Syncer -->|Write| SQLite
        SQLite -->|Read| API
    end
```

### 2.1 物理层级
1.  **持久层 (L1-Truth)**：`YAML` 文件。纳入 Git 版本控制。
2.  **运行时层 (L2-Runtime)**：`SQLite` + `NetworkX`。`.gitignore` 忽略。
3.  **提案缓冲层 (L3-Buffer)**：`pending_proposals/*.json`。临时存储 AI 的修改建议，等待人类批准。

---

## 3. 全景数据模型 (The Grand Schema)

采用您的 **L1/L2/L3 模型**，并增加 **基质层 (L0)**。

### 3.1 节点类型矩阵

| 层级 | 英文标识 | ID前缀 | 职责 | 示例 |
| :--- | :--- | :--- | :--- | :--- |
| **L0 基质** | **Standard** | `std_` | **环境上下文**。定义的通用规范，不参与连线，但随域注入。 | `std_logging` (日志规范) |
| **L1 概念** | **Feature** | `feat_` | **产品意图**。WHY 和 WHAT。 | `feat_login` |
| | **UserStory** | `us_` | 验收标准。 | `us_login_mobile` |
| **L2 结构** | **API** | `api_` | **契约定义**。L2 是最核心的图谱锚点。 | `api_auth_login` |
| | **DataModel** | `model_` | 数据库 Schema。 | `model_users` |
| | **Component** | `comp_` | 逻辑组件。 | `comp_login_form` |
| **L3 实现** | **Function** | `fn_` | **代码影子**。只存签名和副作用，不存代码体。 | `fn_login_handler` |
| **跨层** | **Rule** | `rule_` | 硬性约束。 | `rule_pwd_complexity` |
| | **Doc** | `doc_` | 文档索引。 | `doc_prd_auth` |

### 3.2 关键 Schema 定义补充

#### Standard (基质 - 新增)
用于解决“AI 不知道日志该怎么打”的问题。

```yaml
# .specindex/L0/standards/std_logging.yaml
id: std_logging
type: Standard
scope: ["backend", "api"]  # 注入范围
priority: critical
content: |
  所有错误日志必须包含 trace_id。
  禁止打印 PII（敏感信息）。
  格式必须为 JSON: {"level": "info", "msg": "..."}
```

#### Feature & API & Function
*(直接采纳您文档中的 Schema 定义，那是完美的。特别是 `side_effects` 枚举，必须保留。)*

---

## 4. 目录结构规范

```text
/project_root
  ├── .specindex/
  │   ├── schema/                 # JSON Schema 校验文件
  │   ├── .runtime/               # [GitIgnore] SQLite & Logs
  │   ├── .pending/               # [GitIgnore] 待审核的提案 JSON
  │   │
  │   ├── L0/                     # 基质层
  │   │   └── standards/
  │   ├── L1/                     # 概念层
  │   │   ├── features/
  │   │   └── flows/
  │   ├── L2/                     # 结构层
  │   │   ├── apis/
  │   │   └── data_models/
  │   ├── L3/                     # 实现层
  │   │   └── functions/
  │   └── rules/                  # 规则层
```

---

## 5. 核心组件设计

### 5.1 Index Syncer (同步器)
*(沿用您的 Python 实现，增加 Standard 处理)*

*   **职责**：YAML $\rightarrow$ SQLite 单向同步。
*   **触发**：启动时、提案批准后、Git Checkout 后。
*   **增强**：在写入 SQLite `nodes` 表时，计算节点的 Embedding（如果配置了向量库），存入 `vector` 字段。

### 5.2 Query Engine (查询引擎)
*(沿用您的 Python 实现，增加 Context Bubble)*

我们需要在基础查询之上，增加一个**“智能组装”**方法：

```python
    def get_context_bubble(self, focus_id: str) -> Dict:
        """
        构建关注气泡：核心节点 + 依赖契约 + 环境基质
        """
        # 1. 获取核心节点
        node = self.get_node(focus_id)
        
        # 2. 获取 L1/L2 依赖 (深度=1)
        deps = self.get_dependencies(focus_id, depth=1)
        
        # 3. 获取相关基质 (Standard)
        # 比如：如果是 API 节点，自动拉取 std_logging, std_security
        standards = self.list_nodes(node_type='Standard')
        relevant_standards = [
            s for s in standards 
            if node['category'] in s['content']['scope']
        ]
        
        return {
            "focus": node,
            "contracts": deps,       # 只给接口签名，不给实现
            "ambient": relevant_standards
        }
```

### 5.3 Proposal Manager (提案管理器 - 新增核心)
这是**治理**的核心。替代直接的 `create_node`。

```python
class ProposalManager:
    def __init__(self, spec_root: Path):
        self.pending_dir = spec_root / ".pending"
        self.pending_dir.mkdir(exist_ok=True)

    def propose_change(self, change_type: str, data: Dict, reason: str) -> str:
        """
        提交提案
        Args:
            change_type: 'CREATE', 'UPDATE', 'DELETE'
            data: 节点的 YAML 内容
            reason: AI 解释为什么要改
        """
        proposal_id = f"prop_{uuid.uuid4().hex[:8]}"
        proposal = {
            "id": proposal_id,
            "timestamp": datetime.now().isoformat(),
            "type": change_type,
            "reason": reason,
            "data": data,
            "status": "PENDING"
        }
        
        # 写入临时 JSON
        with open(self.pending_dir / f"{proposal_id}.json", "w") as f:
            json.dump(proposal, f, indent=2)
            
        return proposal_id

    def list_proposals(self) -> List[Dict]:
        # 列出所有待审核提案
        pass

    def approve_proposal(self, proposal_id: str, syncer: IndexSyncer):
        """
        批准提案：JSON -> YAML -> Sync
        """
        p_path = self.pending_dir / f"{proposal_id}.json"
        with open(p_path) as f:
            prop = json.load(f)
            
        # 1. 执行写入 YAML 操作 (利用 API 中的 _get_file_path 逻辑)
        if prop['type'] == 'CREATE' or prop['type'] == 'UPDATE':
            # Write to YAML...
            pass
        elif prop['type'] == 'DELETE':
            # Delete YAML...
            pass
            
        # 2. 删除提案文件
        p_path.unlink()
        
        # 3. 触发局部同步
        syncer.sync_file(...) 
```

---

## 6. API 接口定义 (SpecIndex API v1)

对外暴露的标准化接口。

| 方法 | 端点 | 描述 |
| :--- | :--- | :--- |
| **Context** | `GET /context/bubble?id={id}` | **AI 最常用**。获取任务的完整上下文包。 |
| **Search** | `GET /search?q={query}` | 全文/语义搜索。 |
| **Read** | `GET /node/{id}` | 获取单个节点详情。 |
| **Graph** | `GET /dependencies/{id}` | 获取依赖树。 |
| **Write** | `POST /proposal` | **AI 唯一写入口**。提交变更提案。 |
| **Admin** | `POST /proposal/{id}/approve` | 人类批准提案。 |
| **Admin** | `POST /sync` | 强制全量同步。 |

---

## 7. 数据库设计 (SQLite)

采用您的 SQL 设计，完全无需修改。它非常完美。

```sql
-- 仅展示核心，沿用您的设计
CREATE TABLE nodes (
    id TEXT PRIMARY KEY,
    type TEXT NOT NULL,
    layer TEXT NOT NULL,
    content JSON NOT NULL, -- 完整 YAML 镜像
    search_text TEXT,      -- FTS 索引源
    -- ...
);
CREATE TABLE edges (...);
```

---

## 8. 实施路线图

### 阶段一：骨架与只读 (Week 1)
1.  **Schema 定义**：建立 11 种 YAML 模板 (10种基础 + 1种 Standard)。
2.  **Syncer 开发**：实现 `YAML -> SQLite` 的单向同步。
3.  **Read API**：实现 `get_node`, `search`, `list_nodes`。
*   *产出：您可以手动写 YAML，通过脚本查询数据库。*

### 阶段二：治理闭环 (Week 2)
1.  **Proposal Manager**：开发提案的 JSON 读写逻辑。
2.  **CLI 工具**：开发 `spec propose list` 和 `spec propose approve` 命令行工具。
3.  **Write API**：封装 `POST /proposal`。
*   *产出：AI 可以通过 API 申请修改，您通过 CLI 批准。*

### 阶段三：上下文智能 (Week 3)
1.  **Bubble 算法**：实现 `get_context_bubble`，智能抓取依赖和基质。
2.  **Code Scanner**：引入 Tree-sitter，自动扫描代码文件，更新 L3 `FunctionSummary` 节点。
*   *产出：完全体。AI 获得代码感知道具。*

---

### 💡 架构师总结

这份文档是 **User-Claude 的工程精度** 与 **Gemini 的架构治理** 的完美结合。

1.  **它足够简单**：基于文件和 SQLite，单人即可维护。
2.  **它足够安全**：提案机制确保了 AI 不会把知识库搞乱。
3.  **它足够智能**：分层结构和上下文气泡设计，专为 LLM 的认知特性优化。

这就是您需要的最终版 **SpecIndex** 设计。