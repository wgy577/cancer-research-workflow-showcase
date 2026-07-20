# Architecture & Communication

本文公开癌种诊断任务调研 Agent 的设计结构，不包含生产源码、Prompt、内部知识内容或部署凭据。

## 1. 为什么采用 Workflow Harness

一次癌种调研同时包含知识检索、任务拆分、医学文本生成、事实审核、医生编辑和长期记忆更新。让一个模型端到端完成会产生三个问题：上下文过长、职责冲突、错误无法定位。因此系统将不同能力拆成阶段，并让每个阶段通过结构化合同通信。

```text
Intent
  → Retrieval Context
  → Task Skeleton
  → Per-task Details
  → Review Findings
  → Corrected Draft
  → Final Audit
  → Human Decision
  → Long-term Memory
```

## 2. 逻辑项目结构

```text
workflow-showcase/
├── experience/
│   ├── admin_workspace       新建、历史记录、整合与导出
│   ├── physician_workspace   分享入口、多种编辑方式与提交
│   └── mcp_entry             外部 Agent 的结构化入口
├── orchestration/
│   ├── input_validation      白名单与规范癌种解析
│   ├── b1_context_builder    RAG、规则和标杆上下文
│   ├── b2_generation         B2A 框架 + B2B 逐项并行扩写
│   ├── b3_review             程序检查、双审核、分片修正
│   └── b4_final_audit        联网终审与引用核查
├── memory/
│   ├── base_kb               基础癌种知识
│   ├── doctor_kb             医生定稿
│   ├── rules                 冻结规则与可演化规则
│   └── retrieval_index       cancer schema、keywords、RAG index
├── flywheel/
│   ├── diff_engine           AI 版与医生版差异
│   ├── rule_curator          差异归纳为可复用规则
│   ├── guarded_rewrite       只更新可演化区
│   └── index_rebuild         写入后立即重建索引
└── shared/
    ├── record_store          token 对应的完整记录
    ├── url_verifier         引用验证与状态标记
    ├── schema_contracts      七字段任务与五字段审计合同
    └── excel_export          最终交付
```

这是逻辑结构，用来说明职责边界；公开仓库不分发这些生产模块的源码。

## 3. 阶段通信合同

| 发送方 | 接收方 | 主要载荷 | 为什么需要合同 |
|---|---|---|---|
| Input Validator | B1 | canonical cancer profile | 避免同一癌种的多种叫法污染检索 |
| B1 | B2A/B2B | 检索摘要、标杆任务、适用规则 | 只给当前阶段需要的知识 |
| B2A | B2B | 稳定子序号、任务名、任务说明 | 逐项并行后仍能确定性合并 |
| B3 Auditors | B3 Fixer | 问题类型、目标字段、修改理由 | 审与改分离，不让修正模型重新审题 |
| B4 | UI/MCP | 状态、理由、两个建议字段、引用 | 支持人工选择或自动化消费 |
| Physician UI | Flywheel | 原始版、医生版、submission 元数据 | diff 可重放、可归因 |
| Flywheel | B1 | Doctor KB、规则、重建索引 | 下一次运行读取最新长期记忆 |

## 4. State 与 Memory

### 短期状态

前端 State 只保存 token 和轻量记录索引。需要完整任务时，由后端通过 token 从持久化记录读取。这样可以避免长表格反复穿过 UI 状态，也让刷新、返回页面和多轮编辑保持一致。

### 长期记忆

长期记忆由三部分组成：基础知识、医生定稿、可演化规则。Doctor KB 优先级最高；完整命中时可以直接复用医生版本。飞轮不会把每次修改机械追加成无穷历史，而是先归纳成规则，再覆盖同癌种当前最佳版本。

## 5. RAG 数据流

```text
癌种名称
  → canonical profile / aliases / organ / histology hints
  → Base KB + Doctor KB + rebuilt index
  → multi-signal ranking
  → stage-specific context
  → B2 generation / B3 review
```

检索同时考虑语义、任务名称、共享关键词、同器官关系和任务类型。纯向量结果容易被常见分级与分期任务占据，多信号排序用于提高任务粒度匹配。RAG 永远不能覆盖程序冻结规则。

## 6. 多模型协作

- B2A 专注任务框架，B2B 专注每项病理说明，减少联网搜索与长输出争抢上下文。
- B3 两个审核员并行定位问题；修正器只改已确认问题，并按 shard 并行执行。
- B4 独立联网核查事实与引用，不负责增删任务。
- 医生是最终裁判，系统不把模型建议伪装成人类定稿。

## 7. 失败与降级

| 情况 | 处理方式 |
|---|---|
| 输入无法规范化 | 校验阶段直接停止，不创建垃圾记录 |
| Doctor KB 完整命中 | 跳过重复生成，直接复用当前最佳版本 |
| URL 无法探活 | 保留引用并标记未验证，交给医生判断 |
| B3 多轮未收敛 | 达到轮数/时间预算后标记 degraded，继续交付人工审核 |
| B4 暂时不可用 | 不阻塞医生先编辑 B3 版本 |
| 飞轮修改冻结区 | 程序逐字校验失败，整轮回滚 |

## 8. 可观察性

每条调研记录保留阶段状态、模型审核结果、医生 submissions、引用状态和飞轮结果。系统因此可以回答：某一字段来自哪一阶段、为什么被修改、由谁最终确认，以及它是否已经进入下一轮 RAG 记忆。
