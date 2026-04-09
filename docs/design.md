# Smart Purchase Assistant — 技术设计文档

**版本：** v1.0  
**日期：** 2026-04-07  
**状态：** 初稿

---

## 目录

1. [系统架构](#1-系统架构)
2. [数据模型](#2-数据模型)
3. [后端服务设计（CAP）](#3-后端服务设计cap)
4. [AI 模块设计](#4-ai-模块设计)
5. [前端设计（Fiori / SAPUI5）](#5-前端设计fiori--sapui5)
6. [安全与权限设计](#6-安全与权限设计)
7. [部署架构](#7-部署架构)

---

## 1. 系统架构

### 1.1 整体架构图

```
┌─────────────────────────────────────────────────────────────┐
│                    Browser (SAP Fiori / SAPUI5)             │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ My Purchase  │  │ AI-Assisted  │  │ Approval Inbox   │  │
│  │ Requests     │  │ Request      │  │                  │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
│  ┌──────────────┐  ┌──────────────────────────────────────┐ │
│  │ Analytics    │  │ Master Data Maintenance              │ │
│  │ Dashboard    │  │                                      │ │
│  └──────────────┘  └──────────────────────────────────────┘ │
└───────────────────────────┬─────────────────────────────────┘
                            │ OData v4 / REST
                            ▼
┌─────────────────────────────────────────────────────────────┐
│            SAP CAP Service Layer (Node.js)                  │
│                  SAP BTP Cloud Foundry                      │
│                                                             │
│  ┌─────────────────┐   ┌──────────────────────────────┐    │
│  │ PurchaseService │   │ AIService                     │    │
│  │ (OData CRUD)    │   │  - /generateSuggestion        │    │
│  │                 │   │  - /getApprovalSummary         │    │
│  │ ApprovalService │   │  - /getQualityScore            │    │
│  │                 │   │  - /getRiskAssessment          │    │
│  │ AnalyticsService│   │  - /getAnalyticsInsights       │    │
│  └────────┬────────┘   └──────────────┬───────────────┘    │
│           │                           │                     │
└───────────┼───────────────────────────┼─────────────────────┘
            │                           │
     ┌──────▼──────┐            ┌───────▼────────────────┐
     │ HANA Cloud  │            │ SAP Generative AI Hub  │
     │             │            │ (Claude / GPT-4o)      │
     │ - 采购申请  │            └────────────────────────┘
     │ - 审批记录  │
     │ - 预算数据  │            ┌────────────────────────┐
     │ - 供应商    │            │ SAP AI Core (可选)     │
     │ - 品类规则  │            │ (ML 质量评分模型)      │
     │ - 定价参考  │            └────────────────────────┘
     └─────────────┘
```

### 1.2 技术选型

| 层次 | 技术 | 版本/说明 |
|------|------|-----------|
| 前端框架 | SAPUI5 | 1.136+，sap_horizon 主题 |
| 前端模式 | SAP Fiori Elements | List Report、Object Page |
| 后端框架 | SAP CAP | Node.js runtime |
| 数据库 | SAP HANA Cloud | 云托管 HANA |
| 认证授权 | SAP BTP XSUAA | OAuth 2.0 + JWT |
| AI / LLM | SAP Generative AI Hub | 支持 Claude Sonnet、GPT-4o |
| ML 模型（可选） | SAP AI Core | 自定义质量评分模型 |
| 部署平台 | SAP BTP Cloud Foundry | 多实例部署 |
| API 协议 | OData v4（CRUD）+ REST（AI 端点） | — |

---

## 2. 数据模型

### 2.1 CDS 实体定义

#### PurchaseRequest（采购申请）

```cds
entity PurchaseRequests : cuid, managed {
  requestNo       : String(20);        // PR-2026-XXXX，系统自动生成
  title           : String(200);
  description     : String(2000);      // 自然语言原始描述
  category        : Association to Categories;
  quantity        : Decimal(10, 2);
  unit            : String(10);        // 个/台/套/张
  unitPrice       : Decimal(15, 2);    // 估算单价（CNY）
  totalAmount     : Decimal(15, 2);    // 计算字段：quantity × unitPrice
  currency        : String(3) default 'CNY';
  supplier        : Association to Suppliers;
  specifications  : LargeString;       // 技术规格
  justification   : String(2000);      // 业务理由
  costCenter      : Association to CostCenters;
  requiredByDate  : Date;
  requester       : Association to Users;
  status          : String(20);        // Draft/Submitted/PendingApproval/Approved/Rejected
  submittedAt     : Timestamp;
  aiQualityScore  : Integer;           // 0–100，提交后异步写入
  aiScoreDetails  : LargeString;       // JSON，各维度分项
  aiGenerated     : Boolean default false; // 是否由 AI 辅助创建
}
```

#### ApprovalRecord（审批记录）

```cds
entity ApprovalRecords : cuid, managed {
  request         : Association to PurchaseRequests;
  approver        : Association to Users;
  action          : String(20);        // Approved/Rejected/Returned
  comment         : String(2000);
  approvedAt      : Timestamp;
  aiSummary       : LargeString;       // JSON，AI 审批摘要缓存
  aiRecommendation: String(20);        // Approve/Reject/Review
  riskLevel       : String(10);        // Low/Medium/High
  riskDetails     : LargeString;       // JSON，风险项列表
}
```

#### Supplier（供应商）

```cds
entity Suppliers : cuid, managed {
  name            : String(200);
  code            : String(20);
  categories      : Composition of many SupplierCategories on categories.supplier = $self;
  status          : String(20);        // Active/Inactive/Blacklisted
  rating          : Decimal(3, 1);     // 0.0–5.0
  contactEmail    : String(200);
  contractExpiry  : Date;
  leadTimeDays    : Integer;
}
```

#### Category（品类）

```cds
entity Categories : cuid {
  code            : String(20);
  name            : String(200);
  parentCategory  : Association to Categories;
  rules           : Composition of many CategoryRules on rules.category = $self;
}

entity CategoryRules : cuid {
  category        : Association to Categories;
  costCenter      : Association to CostCenters;
  allowed         : Boolean default true;
  requiresExtraApproval : Boolean default false;
  pricingRef      : Association to PricingReferences;
}
```

#### PricingReference（定价参考）

```cds
entity PricingReferences : cuid, managed {
  category        : Association to Categories;
  description     : String(200);
  priceMin        : Decimal(15, 2);
  priceMedian     : Decimal(15, 2);
  priceMax        : Decimal(15, 2);
  currency        : String(3) default 'CNY';
  validFrom       : Date;
  validTo         : Date;
  updatedBy       : String(100);
}
```

#### AIRuleConfig（AI 规则配置）

```cds
entity AIRuleConfigs : cuid, managed {
  key             : String(100);       // 规则键，如 'budget.mediumRiskThreshold'
  value           : String(200);       // 规则值，如 '0.75'
  description     : String(500);
  category        : String(50);        // QualityScore / RiskAssessment / Approval
}
```

#### Budget（预算）

风险评估和 AI 审批摘要的核心数据源。记录各成本中心的季度预算总额、已提交金额与已审批支出，用于计算预算余额和使用率。

```cds
entity Budgets : cuid, managed {
  costCenter      : Association to CostCenters;
  fiscalYear      : Integer;           // 财年，如 2026
  quarter         : Integer;           // 季度 1–4
  totalBudget     : Decimal(15, 2);    // 季度预算总额（CNY）
  committed       : Decimal(15, 2);    // 已提交未审批的金额（PendingApproval 状态申请之和）
  spent           : Decimal(15, 2);    // 已审批支出（Approved 状态申请之和）
  currency        : String(3) default 'CNY';
  // 虚拟计算字段（由 CAP 计算，不持久化）
  virtual remaining    : Decimal(15, 2);  // totalBudget - committed - spent
  virtual usageRate    : Decimal(5, 4);   // (committed + spent) / totalBudget
}
```

**与 AI 功能的关联：**

| 字段 | 使用场景 | 风险判断逻辑 |
|------|----------|-------------|
| `remaining` | 风险评估 — 预算可用性 | `remaining < request.totalAmount` → High Risk |
| `usageRate` | 风险评估 — 预算消耗率 | `> 0.9` → High；`0.75–0.9` → Medium；`< 0.75` → Low |
| `remaining` + `usageRate` | AI 审批摘要 | 作为 LLM Prompt 上下文，生成"Q2 IT 预算使用率 78%，余额充足"等描述 |

**阈值配置（存储于 AIRuleConfigs）：**

| 配置键 | 默认值 | 说明 |
|--------|--------|------|
| `budget.mediumRiskThreshold` | `0.75` | 使用率超过此值 → Medium Risk |
| `budget.highRiskThreshold` | `0.90` | 使用率超过此值 → High Risk |
| `budget.insufficientFactor` | `1.0` | 余额 < 申请金额 × 此系数 → High Risk |

#### User（用户）

```cds
entity Users : cuid {
  userID          : String(50);        // 工号，如 EMP-0042
  name            : String(100);
  email           : String(200);
  department      : String(100);
  costCenter      : Association to CostCenters;
  role            : String(30);        // Employee / Manager / ProcurementAdmin / SystemAdmin
  managedBy       : Association to Users;
}
```

> **说明：** 在向 LLM 发送数据时，`name` 替换为 `userID`（工号），不传输真实姓名，满足数据脱敏要求。

### 2.2 实体关系图

```
Users ──────────────── PurchaseRequests ──── ApprovalRecords
  │                         │    │                │
  │                         │    └── Categories   │
  │                         │    └── Suppliers    │
  │                         │    └── CostCenters ─┼── Budgets
  └─── (approver) ──────────┘                     │
                                                   └── (风险评估读取)

Categories ──── CategoryRules ──── CostCenters
           └─── PricingReferences
           └─── SupplierCategories ──── Suppliers
```

### 2.3 各 AI 功能数据依赖汇总

| 实体 | 建议生成 | 质量评分 | 审批摘要 | 风险评估 | 分析洞察 |
|------|:-------:|:-------:|:-------:|:-------:|:-------:|
| PurchaseRequests | ✓ 写入 | ✓ 读/写 | ✓ 读 | ✓ 读 | ✓ 读 |
| Suppliers | ✓ 读 | ✓ 读 | ✓ 读 | ✓ 读 | ✓ 读 |
| Categories | ✓ 读 | ✓ 读 | — | ✓ 读 | ✓ 读 |
| CategoryRules | — | ✓ 读 | — | ✓ 读 | — |
| PricingReferences | ✓ 读 | ✓ 读 | — | — | — |
| ApprovalRecords | — | — | ✓ 读/写 | ✓ 读/写 | ✓ 读 |
| **Budgets** | — | — | ✓ 读 | **✓ 核心** | — |
| AIRuleConfigs | — | ✓ 读 | — | ✓ 读 | — |
| Users | — | — | ✓ 读 | ✓ 读 | — |

---

## 3. 后端服务设计（CAP）

### 3.1 PurchaseService（采购申请服务）

**OData 端点：** `/odata/v4/purchase`

| 操作 | 端点 | 说明 |
|------|------|------|
| 查询申请列表 | `GET /PurchaseRequests` | 自动过滤当前用户 |
| 查询申请详情 | `GET /PurchaseRequests(guid)` | — |
| 创建申请 | `POST /PurchaseRequests` | 自动生成 requestNo |
| 更新申请 | `PATCH /PurchaseRequests(guid)` | 仅 Draft 状态可编辑 |
| 提交申请 | `POST /PurchaseRequests(guid)/submit` | 触发状态变更 + 异步 AI 评分 |
| 撤回申请 | `POST /PurchaseRequests(guid)/withdraw` | Submitted → Draft |

**关键业务逻辑：**

```javascript
// 提交申请时的处理流程
srv.on('submit', 'PurchaseRequests', async (req) => {
  const { ID } = req.params[0];
  
  // 1. 校验必填字段
  await validateRequiredFields(ID);
  
  // 2. 更新状态
  await UPDATE(PurchaseRequests, ID).with({ status: 'PendingApproval', submittedAt: new Date() });
  
  // 3. 确定审批人（基于金额和组织层级）
  const approver = await resolveApprover(ID);
  await createApprovalTask(ID, approver);
  
  // 4. 异步触发 AI 质量评分（不阻塞提交）
  triggerAsyncQualityScore(ID);
  
  return { success: true };
});
```

### 3.2 ApprovalService（审批服务）

**OData 端点：** `/odata/v4/approval`

| 操作 | 端点 | 说明 |
|------|------|------|
| 查询待审批列表 | `GET /PendingApprovals` | 自动过滤当前审批人 |
| 批准申请 | `POST /PendingApprovals(guid)/approve` | 记录审批结果 |
| 驳回申请 | `POST /PendingApprovals(guid)/reject` | 必须包含 comment |

### 3.3 AnalyticsService（分析服务）

**端点：** `/odata/v4/analytics`

提供聚合数据视图，供仪表板使用：

```cds
service AnalyticsService {
  @readonly
  entity SpendByDepartment as projection on PurchaseRequests {
    costCenter.department as department,
    sum(totalAmount) as totalSpend,
    count(*) as requestCount
  } group by costCenter.department;

  @readonly
  entity SpendByCategory as projection on PurchaseRequests {
    category.name as categoryName,
    sum(totalAmount) as totalSpend
  } group by category.name;

  @readonly
  entity MonthlyTrend as projection on PurchaseRequests {
    year(submittedAt) as year,
    month(submittedAt) as month,
    sum(totalAmount) as actualSpend
  } group by year(submittedAt), month(submittedAt);
}
```

---

## 4. AI 模块设计

### 4.1 AI 申请建议生成

**触发：** 用户在 AI 对话框发送自然语言描述

**处理流程：**

```
用户输入
  │
  ▼
CAP AIService.generateSuggestion(userInput)
  │
  ├── 1. 加载上下文数据（品类列表、供应商名单、定价参考）
  │
  ├── 2. 构建 Prompt
  │       System: "你是企业采购助理，根据员工描述生成结构化采购申请草稿..."
  │       User: userInput + 上下文数据（JSON）
  │       Output format: { title, category, quantity, unitPrice, supplier, specs, justification }
  │
  ├── 3. 调用 SAP Gen AI Hub（LLM）
  │
  ├── 4. 解析并校验返回的 JSON
  │       - 校验品类是否在系统中存在
  │       - 校验供应商是否在批准名单
  │       - 校验价格是否在参考区间内
  │
  └── 5. 返回结构化建议 + 合规标签 + 预估质量分
```

**Prompt 模板（示例）：**

```
System:
你是一个企业采购申请助理。根据员工的采购需求描述，
生成一个结构化的采购申请草稿。
严格按照以下 JSON 格式输出，不要输出其他内容。

User:
员工需求描述：{userInput}

可用品类：{categoriesJson}
批准供应商名单：{suppliersJson}
定价参考区间：{pricingJson}

请生成采购申请草稿，JSON 格式：
{
  "title": "...",
  "categoryCode": "...",
  "quantity": 数字,
  "unitPrice": 数字,
  "supplierCode": "...",
  "specifications": "...",
  "justification": "..."
}
```

### 4.2 AI 质量评分

**触发：** 申请提交后异步执行，结果写入 `PurchaseRequests.aiQualityScore`

**评分算法：**

```javascript
async function calcQualityScore(requestId) {
  const request = await SELECT.one.from(PurchaseRequests).where({ ID: requestId });
  const config = await loadAIConfig('QualityScore');
  const pricingRef = await loadPricingRef(request.category_ID);

  let score = 0;

  // ── 完整性（满分 30）────────────────────────────────────────
  const completenessWeight = config['completeness.weight'] || 30;
  let completeness = 0;
  if (request.specifications && request.specifications.length > 20) completeness += 10;
  if (request.justification   && request.justification.length > 30)  completeness += 10;
  if (request.costCenter_ID)                                          completeness += 5;
  if (request.requiredByDate)                                         completeness += 5;
  score += Math.round(completeness * completenessWeight / 30);

  // ── 价格合理性（满分 25）────────────────────────────────────
  const priceWeight = config['price.weight'] || 25;
  if (pricingRef) {
    const ratio = request.unitPrice / pricingRef.priceMedian;
    let priceScore = 0;
    if (ratio >= 0.7 && ratio <= 1.3)      priceScore = 25;
    else if (ratio > 1.3 && ratio <= 1.6)  priceScore = 15;
    else if (ratio > 1.6)                  priceScore = 5;
    else                                   priceScore = 20; // 低于参考，加分
    score += Math.round(priceScore * priceWeight / 25);
  } else {
    score += Math.round(15 * priceWeight / 25); // 无参考价，给中等分
  }

  // ── 合规性（满分 30）────────────────────────────────────────
  const complianceWeight = config['compliance.weight'] || 30;
  let compliance = 0;
  const isApproved = await checkApprovedSupplier(request.supplier_ID);
  const isCategoryOk = await checkCategoryRule(request.category_ID, request.costCenter_ID);
  if (isApproved)    compliance += 15;
  if (isCategoryOk)  compliance += 15;
  score += Math.round(compliance * complianceWeight / 30);

  // ── 理由质量（满分 15）─────────────────────────────────────
  const justWeight = config['justification.weight'] || 15;
  let justScore = 0;
  if (request.justification) {
    if (request.justification.length > 50)  justScore += 8;
    if (request.justification.length > 100) justScore += 4;
    if (hasBusinessKeywords(request.justification)) justScore += 3;
  }
  score += Math.round(justScore * justWeight / 15);

  // 写回数据库
  await UPDATE(PurchaseRequests, requestId).with({
    aiQualityScore: Math.min(100, score),
    aiScoreDetails: JSON.stringify({ completeness, price: priceScore, compliance, justification: justScore })
  });
}
```

**分项维度权重（可配置）：**

| 维度 | 默认权重 | 配置键 |
|------|----------|--------|
| 完整性 | 30% | `completeness.weight` |
| 合规性 | 30% | `compliance.weight` |
| 价格合理性 | 25% | `price.weight` |
| 理由质量 | 15% | `justification.weight` |

### 4.3 AI 审批摘要

**触发：** 审批人打开申请详情时，若缓存不存在则异步生成

**Prompt 模板：**

```
System:
你是一个企业采购审批助理。根据提供的采购申请详情，
生成一段简洁的审批摘要，帮助审批人快速了解申请要点。
严格按照 JSON 格式输出。

User:
采购申请详情：{requestJson}
申请人部门预算状态：{budgetJson}
供应商资质信息：{supplierJson}
近期同类申请参考：{historicalJson}

输出 JSON 格式：
{
  "summary": "（2-3句自然语言摘要）",
  "recommendation": "Approve | Reject | Review",
  "highlights": [
    { "type": "positive|warning|info", "text": "..." }
  ]
}
```

**缓存策略：**
- 生成后写入 `ApprovalRecords.aiSummary`
- 申请数据变更（如金额修改）后标记缓存失效
- LLM 调用失败时，前端显示降级提示，不影响审批操作

**`{budgetJson}` 的组装逻辑：**

```javascript
// 根据申请的 costCenter_ID + 当前财年季度查询 Budgets 表
const budget = await SELECT.one.from(Budgets)
  .where({ costCenter_ID: request.costCenter_ID,
           fiscalYear: currentYear,
           quarter: currentQuarter });

const budgetJson = {
  totalBudget:  budget.totalBudget,
  committed:    budget.committed,
  spent:        budget.spent,
  remaining:    budget.totalBudget - budget.committed - budget.spent,
  usageRate:    (budget.committed + budget.spent) / budget.totalBudget,
  requestAmount: request.totalAmount
};
// budgetJson 作为 Prompt 上下文传入 LLM，
// 由 LLM 生成"Q2 IT 预算使用率 78%，余额 ¥88,000，足以覆盖本次申请"等自然语言描述
```

### 4.4 风险评估

**触发：** 与 AI 审批摘要同时计算，结果缓存于 `ApprovalRecords`

**预算数据读取：**

风险评估中"预算可用性"和"预算消耗率"两项均来自 `Budgets` 表。读取逻辑与审批摘要相同（按 costCenter_ID + 财年季度），但由规则引擎直接判断，不经过 LLM：

```javascript
async function calcRiskItems(request) {
  // 读取预算数据
  const budget = await SELECT.one.from(Budgets)
    .where({ costCenter_ID: request.costCenter_ID,
             fiscalYear: currentYear,
             quarter: currentQuarter });

  const remaining  = budget.totalBudget - budget.committed - budget.spent;
  const usageRate  = (budget.committed + budget.spent) / budget.totalBudget;

  // 读取风险阈值配置
  const config = await loadAIConfig('RiskAssessment');
  const medThreshold  = parseFloat(config['budget.mediumRiskThreshold'] || '0.75');
  const highThreshold = parseFloat(config['budget.highRiskThreshold']   || '0.90');

  return [
    {
      title:  "Budget Availability",
      level:  remaining >= request.totalAmount ? "Low" : "High",
      detail: `¥${remaining.toLocaleString()} remaining in budget`
    },
    {
      title:  "Budget Consumption Rate",
      level:  usageRate > highThreshold ? "High"
            : usageRate > medThreshold  ? "Medium" : "Low",
      detail: `${Math.round(usageRate * 100)}% of budget committed`
    },
    // ... 其他风险项
  ];
}
```

**风险项列表与规则：**

| 风险项 | 数据来源 | Low | Medium | High |
|--------|---------|-----|--------|------|
| 预算可用性 | **Budgets.remaining** | 余额 ≥ 申请金额 | — | 余额 < 申请金额 |
| 供应商合规 | Suppliers.status | 批准名单内 | — | 不在批准名单 |
| 预算消耗率 | **Budgets.usageRate** | < 75% | 75%–90% | > 90% |
| 采购政策符合性 | CategoryRules | 规则通过 | 需额外审批 | 不允许采购 |
| 申请人历史驳回率 | ApprovalRecords | < 20% | 20%–50% | > 50% |

阈值数值从 `AIRuleConfigs` 表中读取，管理员可在 Master Data Maintenance 中调整。

### 4.5 AI 分析洞察

**触发：** 每日定时任务（凌晨 2 点），或管理员手动刷新

**处理流程：**

```
定时触发
  │
  ▼
加载聚合数据（近 90 天采购统计）
  │
  ▼
计算统计指标：
  - 各部门支出同比变化
  - 非合规供应商使用次数
  - 跨部门同类采购合并潜力
  │
  ▼
构建 Prompt（传入统计数据，要求生成洞察文本）
  │
  ▼
调用 SAP Gen AI Hub
  │
  ▼
解析并存储洞察结果（最多 5 条，标注类型：Trend/Anomaly/Opportunity）
```

---

## 5. 前端设计（Fiori / SAPUI5）

### 5.1 应用结构

```
demos/
├── index.html                        # 导航入口
├── 01_my_purchase_requests.html      # 我的采购申请（SAPUI5）
├── 02_ai_assisted_request.html       # AI 辅助申请（纯 HTML Demo）
├── 03_approval_inbox.html            # 审批收件箱（纯 HTML Demo）
└── 04_analytics_dashboard.html       # 分析仪表板（纯 HTML Demo）
```

### 5.2 My Purchase Requests — 组件结构

```
sap.m.Page
  ├── customHeader: sap.m.Bar（Shell Bar）
  ├── subHeader: sap.m.Bar（页面标题）
  └── content:
       ├── sap.m.HBox (KPI 卡片行)
       │    └── × 4 sap.m.VBox (单个 KPI)
       ├── sap.m.OverflowToolbar (操作栏)
       │    ├── Button: "+ New Request"
       │    ├── Button: "✨ AI-Assisted Request"  → 打开 AI 对话框
       │    └── sap.m.SearchField
       └── sap.m.Table (申请列表)
            └── bindAggregation("items", { path: "pr>/requests", factory: ... })
```

### 5.3 AI 对话框 — 组件结构

```
sap.m.Dialog (contentWidth: "90vw", height: "85vh", resizable, draggable)
  └── sap.m.HBox (aiDialogLayout)
       ├── 左侧: sap.m.VBox (aiLeftPanel, flex: 54%)
       │    ├── sap.m.Bar (aiDialogSubBar) — 模型信息 + 设置按钮
       │    ├── sap.m.MessageStrip — AI 处理进度（可见性绑定 state>/progressVisible）
       │    ├── sap.m.ScrollContainer — 对话消息列表
       │    │    └── sap.m.List (FeedListItem × N)
       │    └── sap.m.FeedInput — 用户输入框
       │         └── post 事件 → _runGenerateSuggestion(text)
       │
       └── 右侧: sap.m.VBox (aiRightPanel, flex: 46%)
            ├── 空状态: sap.m.VBox (aiEmptyState)  ← visible: resultPanel === "empty"
            └── 建议内容: sap.m.ScrollContainer    ← visible: resultPanel === "proposals"
                 └── 动态构建的建议卡片
                      ├── 合规标签行
                      ├── 字段详情（标题/品类/数量/价格/供应商/规格/理由）
                      └── Button: "✓ Confirm & Add to List"
                           └── press → 插入列表首行，关闭对话框
```

### 5.4 状态模型（JSONModel）

```javascript
// oStateModel — 控制 AI 对话框 UI 状态
{
  jobRunning:         false,   // AI 处理中
  progressVisible:    false,   // 进度条可见
  progressText:       "",      // 进度文字
  resultPanel:        "empty", // "empty" | "proposals"
  toolButtonsEnabled: true     // 输入框是否可用
}

// oPRModel — 采购申请数据
{
  requests: [ { id, title, category, amount, submitted, status, statusState, aiScore } ]
}

// oChatMessages — 对话消息
{
  items: [ { sender, senderName, text, timestamp } ]
}
```

### 5.5 UI 主题与样式规范

- **主题：** `sap_horizon`（SAP Fiori 3 Horizon 主题）
- **CDN：** `https://ui5.sap.com/1.136.8/resources/sap-ui-core.js`
- **AI 元素标识：** 使用 `sap-icon://ai`（✦ 图标）和 `✨` emoji 标注 AI 生成内容
- **状态颜色：**

| 场景 | 颜色 | CSS 变量 |
|------|------|----------|
| AI 质量高（≥85） | 绿色 | `#ebf5e0 / #1a7a1a` |
| AI 质量中（70–84） | 橙色 | `#fff4e0 / #c75300` |
| AI 质量低（<70） | 红色 | `#ffeaea / #c0392b` |
| AI 生成内容背景 | 浅蓝 | `#f0f7ff` |
| AI 辅助按钮 | 蓝紫渐变 | `linear-gradient(135deg, #0a6ed1, #6366f1)` |

---

## 6. 安全与权限设计

### 6.1 角色定义（XSUAA）

```json
{
  "role-templates": [
    {
      "name": "Employee",
      "description": "采购申请员工",
      "scope-references": ["$XSAPPNAME.Employee"]
    },
    {
      "name": "Manager",
      "description": "采购审批经理",
      "scope-references": ["$XSAPPNAME.Employee", "$XSAPPNAME.Manager"]
    },
    {
      "name": "ProcurementAdmin",
      "description": "采购管理员",
      "scope-references": ["$XSAPPNAME.Employee", "$XSAPPNAME.Manager", "$XSAPPNAME.ProcurementAdmin"]
    },
    {
      "name": "SystemAdmin",
      "description": "系统管理员",
      "scope-references": ["$XSAPPNAME.SystemAdmin"]
    }
  ]
}
```

### 6.2 CAP 授权注解

```cds
service PurchaseService @(requires: 'Employee') {
  entity PurchaseRequests @(restrict: [
    { grant: ['READ'],   where: 'requester.userID = $user' },
    { grant: ['CREATE'], to: 'Employee' },
    { grant: ['UPDATE'], to: 'Employee', where: 'status = ''Draft''' },
    { grant: ['READ'],   to: ['Manager', 'ProcurementAdmin'] }
  ]) { ... }
}

service ApprovalService @(requires: 'Manager') {
  entity PendingApprovals @(restrict: [
    { grant: ['READ', 'WRITE'], where: 'approver.userID = $user' }
  ]) { ... }
}

service AnalyticsService @(requires: 'ProcurementAdmin') { ... }
service MasterDataService @(requires: ['ProcurementAdmin', 'SystemAdmin']) { ... }
```

### 6.3 AI 数据安全

- 发送给 LLM 的数据中，申请人姓名替换为工号（`EMP-XXXX`）
- 不传输身份证号、手机号等个人敏感信息
- LLM 调用通过 SAP Generative AI Hub，数据不离开 SAP BTP 区域
- 所有 LLM 调用记录日志，保留 90 天用于审计

---

## 7. 部署架构

### 7.1 Cloud Foundry 部署配置（mta.yaml 概要）

```yaml
modules:
  - name: smart-purchase-approuter
    type: approuter.nodejs
    requires:
      - name: smart-purchase-xsuaa
      - name: smart-purchase-destination

  - name: smart-purchase-srv
    type: nodejs
    path: srv
    requires:
      - name: smart-purchase-db
      - name: smart-purchase-xsuaa
      - name: smart-purchase-aicore

  - name: smart-purchase-db-deployer
    type: hdb
    path: db
    requires:
      - name: smart-purchase-db

resources:
  - name: smart-purchase-db
    type: com.sap.xs.hdi-container

  - name: smart-purchase-xsuaa
    type: org.cloudfoundry.managed-service
    parameters:
      service: xsuaa
      service-plan: application

  - name: smart-purchase-aicore
    type: org.cloudfoundry.managed-service
    parameters:
      service: aicore
      service-plan: extended
```

### 7.2 环境变量

| 变量名 | 说明 |
|--------|------|
| `AI_MODEL_NAME` | LLM 模型名称（如 `claude-sonnet-4-6`） |
| `AI_MAX_TOKENS` | LLM 最大输出 Token 数（默认 1000） |
| `AI_TEMPERATURE` | LLM 温度参数（默认 0.3，偏确定性） |
| `QUALITY_SCORE_ASYNC` | 质量评分是否异步（默认 true） |
| `INSIGHTS_CRON` | 洞察生成定时任务表达式（默认 `0 2 * * *`） |

### 7.3 性能优化策略

| 策略 | 说明 |
|------|------|
| AI 评分预计算 | 申请提交后立即异步计算，审批时直接读缓存 |
| AI 摘要缓存 | 首次生成后缓存于数据库，申请未变更则不重复调用 |
| 分析数据预聚合 | 每小时定时预聚合统计数据，仪表板直接读聚合表 |
| LLM 调用限流 | 同一申请 10 分钟内不重复调用 LLM |
| 降级处理 | LLM 不可用时，隐藏 AI 模块，核心流程正常运行 |
