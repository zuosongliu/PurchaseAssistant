# AI Feature Design — Smart Purchase Assistant

本文档描述 Purchase Approval Inbox 中三个 AI 功能模块的设计方案，包括数据来源、计算方式与技术架构。

---

## 1. AI Approval Assistant（AI 审批摘要）

### 功能描述

在审批详情页顶部展示一段 AI 生成的自然语言摘要，说明申请的关键信息、合规状态，并给出审批推荐结论（Approve / Reject）。

### 输入数据

| 数据项 | 来源 |
|--------|------|
| 采购申请完整字段（标题、金额、品类、供应商、规格、业务理由） | HANA Cloud — 采购申请表 |
| 申请人信息、部门、成本中心 | HANA Cloud — 员工主数据 |
| 预算余额与使用率 | HANA Cloud — 预算数据 / ERP 集成 |
| 供应商资质状态 | HANA Cloud — 供应商主数据 |
| 历史同类审批记录 | HANA Cloud — 审批历史表 |

### 获取方式

```
CAP Service (/getAIApprovalSummary)
  → 组装 Prompt（包含申请详情 + 上下文数据）
  → 调用 SAP Generative AI Hub（LLM：Claude / GPT-4o）
  → 解析返回的 JSON：{ summary, recommendation, highlights[] }
  → 写入缓存（避免重复调用）
  → 返回给 Fiori 前端
```

### Prompt 设计要点

- 提供结构化上下文：申请字段、预算状态、供应商资质
- 要求输出 JSON 格式，包含 `summary`（自然语言段落）、`recommendation`（Approve/Reject/Review）、`highlights`（标签数组）
- 语气保持中立客观，不替代人工判断

### 注意事项

- 输出内容**可被审批人手动覆盖**，AI 仅作参考
- 推荐结论不具法律效力，最终决定权在审批人
- 需记录 AI 推荐与实际审批结果，用于模型效果评估

---

## 2. AI Quality Score（AI 质量评分）

### 功能描述

对每条采购申请计算一个 0–100 的质量分，反映申请的完整性、价格合理性、合规性与业务理由质量，辅助审批人快速判断申请质量。

### 输入数据

| 数据项 | 用途 |
|--------|------|
| 申请字段完整性（规格、理由、成本中心等是否填写） | 完整性维度评分 |
| 估价与市场参考价区间对比 | 价格合理性维度评分 |
| 供应商是否在批准名单 | 合规性维度评分 |
| 品类是否符合成本中心采购规则 | 合规性维度评分 |
| 业务理由文字长度与关键词覆盖 | 理由质量维度评分 |
| 历史同类申请的审批通过率 | （可选）ML 预测维度 |

### 获取方式

采用**规则引擎 + 可选 ML 模型**两层评分：

#### 第一层：规则引擎（CAP 后端，确定性评分）

```javascript
// 伪代码示意
function calcQualityScore(request) {
  let score = 0;

  // 完整性 (满分 30)
  if (request.specs)          score += 10;
  if (request.justification)  score += 10;
  if (request.costCenter)     score += 10;

  // 价格合理性 (满分 25)
  const priceRatio = request.unitPrice / marketRef.median;
  if (priceRatio >= 0.7 && priceRatio <= 1.3) score += 25;
  else if (priceRatio < 1.6)                  score += 15;

  // 合规性 (满分 30)
  if (isApprovedVendor(request.supplier))     score += 15;
  if (isCategoryAllowed(request.category, request.costCenter)) score += 15;

  // 理由质量 (满分 15)
  if (request.justification.length > 50)     score += 8;
  if (hasKeywords(request.justification))    score += 7;

  return score; // 0–100
}
```

#### 第二层：ML 模型（可选，SAP AI Core 部署）

- 基于历史审批数据训练分类模型（特征：金额、品类、部门、供应商等）
- 输出审批通过概率，叠加到规则分之上
- 适用于数据积累足够后的进阶场景

### 分项维度展示

| 维度 | 权重 | 说明 |
|------|------|------|
| 完整性 (Completeness) | 30% | 必填字段覆盖率 |
| 合规性 (Compliance) | 30% | 供应商、品类规则符合度 |
| 价格合理性 (Price Reasonableness) | 25% | 与市场参考价区间的偏差 |
| 理由质量 (Justification Quality) | 15% | 业务理由的清晰度与完整性 |

---

## 3. Risk Assessment（风险评估）

### 功能描述

对采购申请逐项评估风险，每个风险项给出 Low / Medium / High 三级结论，并汇总整体风险等级，帮助审批人快速识别需要关注的问题。

### 输入数据

| 数据项 | 来源 |
|--------|------|
| 当前预算余额与使用率 | HANA Cloud — 预算表 / ERP 集成 |
| 供应商资质与黑名单状态 | HANA Cloud — 供应商主数据 |
| 申请金额与审批权限阈值 | HANA Cloud — 采购政策配置表 |
| 申请人/部门历史驳回率 | HANA Cloud — 审批历史表 |
| 同一供应商近期集中采购情况 | HANA Cloud — 采购申请历史 |

### 获取方式

**纯规则引擎**，在 CAP Service 后端计算，无需调用 LLM：

```javascript
// 伪代码示意
function calcRiskItems(request, context) {
  return [
    {
      title: "Budget Availability",
      level: context.budgetRemaining > request.total ? "Low" : "High",
      detail: `¥${context.budgetRemaining} remaining in budget`
    },
    {
      title: "Vendor Compliance",
      level: isApprovedVendor(request.supplier) ? "Low" : "High",
      detail: isApprovedVendor(request.supplier)
        ? "Supplier on approved vendor list"
        : "Supplier NOT on approved vendor list"
    },
    {
      title: "Budget Consumption Rate",
      level: context.budgetUsageRate > 0.9 ? "High"
           : context.budgetUsageRate > 0.75 ? "Medium" : "Low",
      detail: `${Math.round(context.budgetUsageRate * 100)}% of budget committed`
    },
    {
      title: "Policy Compliance",
      level: isCategoryAllowed(request.category, request.costCenter) ? "Low" : "Medium",
      detail: "Category vs cost center policy check"
    }
  ];
}
```

整体风险等级 = 所有风险项中最高级别（任一 High → 整体 High）。

### 注意事项

- 规则引擎评估**不依赖 LLM**，响应快、结果确定、可完整审计
- 风险规则阈值（如预算使用率 75%/90%）由采购管理员在 Master Data Maintenance 中配置
- 所有风险项均附带说明文字，便于审批人理解原因

---

## 整体技术架构

```
Browser (SAP Fiori / SAPUI5)
         │
         ▼
CAP Service — Node.js (SAP BTP Cloud Foundry)
         │
         ├── /getAIApprovalSummary ──► SAP Generative AI Hub
         │                              (LLM: Claude / GPT-4o)
         │                              返回: 自然语言摘要 + 推荐
         │
         ├── /getQualityScore ───────► 规则引擎 (CAP 内部)
         │                         └─► (可选) SAP AI Core ML 模型
         │                              返回: 总分 + 各维度分项
         │
         └── /getRiskAssessment ─────► 规则引擎 (CAP 内部)
                                        返回: 各风险项 + 整体等级
         │
         ▼
    HANA Cloud
    ├── 采购申请数据
    ├── 预算数据
    ├── 供应商主数据
    ├── 历史审批记录
    └── 采购政策配置
```

---

## 设计原则

| 原则 | 说明 |
|------|------|
| **AI 辅助，人工决策** | 所有 AI 输出均为建议，最终审批由人完成 |
| **可解释性** | 每个评分/风险项均附带说明，用户可理解依据 |
| **可覆盖性** | 审批人可忽略或否定任何 AI 推荐 |
| **适当分工** | 自然语言生成用 LLM，确定性判断用规则引擎 |
| **性能考量** | 风险评估和质量评分在申请提交时异步预计算并缓存，审批时直接读取 |
| **可配置性** | 规则阈值由管理员维护，无需修改代码 |
