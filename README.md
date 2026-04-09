# Smart Purchase Assistant

> 基于 SAP BTP 构建的 AI 辅助采购平台，以学习 SAP BTP 核心技术栈为目的的 Demo/POC 项目。

---

## 项目简介

实现采购申请流程，采用以下技术：

- **SAP CAP (Node.js)** — 后端服务与数据建模
- **SAP Fiori Elements / SAPUI5** — 前端界面
- **SAP HANA Cloud** — 数据持久化
- **SAP BTP XSUAA** — 认证与权限管理
- **SAP Generative AI Hub** — AI / LLM 功能集成

---

## 功能模块

### Phase 1 — 核心采购流程

| 应用 | 目标用户 | 核心功能 |
|------|----------|----------|
| **My Purchase Requests** | 员工 | 创建、提交、跟踪采购申请；查看 AI 质量评分 |
| **Purchase Approval Inbox** | 经理 | 审批待办列表；AI 审批摘要、质量评分、风险评估辅助决策 |

### Phase 2 — AI 辅助申请（计划中）

| 应用 | 目标用户 | 核心功能 |
|------|----------|----------|
| **AI-Assisted Request** | 员工 | 自然语言描述需求，AI 自动生成完整申请草稿 |

### Phase 3 — 管理与分析（计划中）

| 应用 | 目标用户 | 核心功能 |
|------|----------|----------|
| **Procurement Analytics Dashboard** | 采购管理员 | 多维度采购数据分析；AI 自动生成趋势/异常/机会洞察 |
| **Master Data Maintenance** | 采购管理员/系统管理员 | 维护供应商、品类规则、定价参考区间、AI 规则阈值 |

### AI 功能详解

- **AI 申请建议生成**：员工用自然语言描述需求（如"需要 5 台开发用笔记本，预算 1.2 万"），LLM 自动匹配品类、推荐合规供应商、估算价格、生成技术规格与业务理由。
- **AI 质量评分（0–100）**：申请提交后异步计算，从完整性（30%）、合规性（30%）、价格合理性（25%）、理由质量（15%）四个维度评分，结果显示在列表与审批详情中。
- **AI 审批摘要**：审批人打开申请时，LLM 生成自然语言摘要，并给出 Approve / Reject / Review 推荐结论与高亮标签。
- **风险评估**：纯规则引擎（无需 LLM），逐项评估预算可用性、供应商合规、预算消耗率、采购政策符合性，每项给出 Low / Medium / High 等级。
- **分析洞察**：每日定时任务调用 LLM，对近 90 天聚合数据生成 3–5 条趋势、异常、机会类洞察文本。

---

## 技术栈

| 层次 | 技术 | 说明 |
|------|------|------|
| 前端框架 | **SAPUI5 1.136+** | sap_horizon 主题，Fiori Design Guidelines |
| 前端模式 | **SAP Fiori Elements** | List Report、Object Page |
| 后端框架 | **SAP CAP (Node.js)** | OData v4 CRUD + REST AI 端点 |
| 数据库 | **SAP HANA Cloud** | 云托管，HDI 容器部署 |
| 认证授权 | **SAP BTP XSUAA** | OAuth 2.0 + JWT，基于角色的访问控制 |
| AI / LLM | **SAP Generative AI Hub** | 支持 Claude Sonnet、GPT-4o 等模型 |
| ML 模型（可选）| **SAP AI Core** | 自定义质量评分模型，基于历史审批数据训练 |
| 部署平台 | **SAP BTP Cloud Foundry** | MTA 多模块部署（Approuter + CAP srv + DB deployer） |

---

## 系统架构

```
┌─────────────────────────────────────────────┐
│         Browser (SAP Fiori / SAPUI5)         │
│  My Requests │ AI Request │ Approval │ Analytics │
└──────────────────┬──────────────────────────┘
                   │ OData v4 / REST
                   ▼
┌─────────────────────────────────────────────┐
│       SAP CAP Service Layer (Node.js)        │
│       SAP BTP Cloud Foundry                  │
│                                             │
│  PurchaseService   ApprovalService           │
│  AnalyticsService  MasterDataService         │
│                                             │
│  AIService:                                  │
│    /generateSuggestion  → LLM               │
│    /getApprovalSummary  → LLM               │
│    /getQualityScore     → 规则引擎           │
│    /getRiskAssessment   → 规则引擎           │
│    /getAnalyticsInsights→ LLM               │
└────────┬──────────────────┬────────────────┘
         │                  │
    ┌────▼────┐    ┌────────▼────────────┐
    │  HANA   │    │ SAP Gen AI Hub      │
    │  Cloud  │    │ (Claude / GPT-4o)   │
    └─────────┘    └─────────────────────┘
```

---

## 数据模型（核心实体）

```
PurchaseRequests ──── ApprovalRecords
       │                    │
       ├── Categories        ├── (AI 摘要缓存)
       ├── Suppliers         └── (风险评估缓存)
       ├── CostCenters ──── Budgets
       └── Users

Categories ──── CategoryRules ──── PricingReferences
           └─── SupplierCategories ──── Suppliers

AIRuleConfigs  (质量评分权重、风险阈值，管理员可配置)
```

---

## 角色与权限

| 角色 | 可访问应用 |
|------|-----------|
| Employee（员工） | My Purchase Requests、AI-Assisted Request |
| Manager（经理） | 员工权限 + Purchase Approval Inbox |
| ProcurementAdmin（采购管理员） | 经理权限 + Analytics Dashboard + Master Data |
| SystemAdmin（系统管理员） | 全部应用 |

权限通过 **SAP BTP XSUAA** 与 CAP 授权注解（`@restrict`）结合实现，员工只能读写自己的申请，经理只能审批权限范围内的申请。

---

## 设计原则

- **AI 辅助，人工决策**：所有 AI 输出均为建议，最终决定权在人。
- **适当分工**：自然语言生成用 LLM，确定性判断（评分/风险）用规则引擎，降低成本和延迟。
- **可解释性**：每个评分维度和风险项均附带说明文字。
- **可配置性**：评分权重、风险阈值通过 Master Data 配置，无需改代码。
- **降级处理**：LLM 不可用时，AI 模块隐藏，核心审批流程不受影响。
- **数据安全**：发送给 LLM 的数据脱敏（姓名替换为工号），数据不离开 SAP BTP 区域。

---

## 学习目标

本项目涵盖以下 SAP BTP 核心技术的实践：

- **SAP CAP**：CDS 数据建模、OData 服务、自定义 Action/Function、授权注解、异步任务
- **SAP HANA Cloud**：HDI 容器、CDS 投影视图、聚合查询
- **SAP BTP XSUAA**：角色模板定义、OAuth 2.0 集成、JWT 鉴权
- **SAP Generative AI Hub**：Prompt 工程、结构化 JSON 输出、LLM 调用缓存策略
- **SAP Fiori Elements**：List Report、Object Page、注解驱动 UI
- **SAPUI5**：JSONModel 状态管理、Dialog 对话框、FeedListItem 对话式 UI
- **Cloud Foundry MTA**：多模块部署（Approuter + CAP srv + DB deployer）

---

## UI Mockup

基于纯 HTML + SAPUI5 CDN 实现的静态原型，可直接在浏览器中打开预览：

| 文件 | 对应功能 |
|------|----------|
| [index.html](ui-mockup/index.html) | 导航入口 |
| [01_my_purchase_requests.html](ui-mockup/01_my_purchase_requests.html) | My Purchase Requests |
| [02_ai_assisted_request.html](ui-mockup/02_ai_assisted_request.html) | AI-Assisted Request |
| [03_approval_inbox.html](ui-mockup/03_approval_inbox.html) | Purchase Approval Inbox |
| [04_analytics_dashboard.html](ui-mockup/04_analytics_dashboard.html) | Procurement Analytics Dashboard |

---

## 文档

- [需求说明文档](docs/requirements.md)
- [技术设计文档](docs/design.md)
- [AI 功能设计](docs/ai-feature-design.md)
