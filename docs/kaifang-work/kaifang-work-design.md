# 开放档案智能体（Kaifang Work）产品与插件架构设计

面向档案行业企业内部部署的桌面智能体，基于 DeepSeek Harness（dsh）的全插件架构构建。

## Summary

本文给出「开放档案智能体」的产品定位、角色场景、插件架构、包目录划分与落地节奏。核心结论：以独立仓库 + 独立 npm scope 的形式，通过 dsh 的 profile/bundle 分层叠加在 `dsh-base` 之上，不 fork 上游；所有档案能力按 dsh 的能力接缝（Service Definition / Service Provider / Consumer）三角色拆包；四性检测、敏感筛查等确定性判定由工具产出，模型只负责编排与表述。

知识库采用双层模型：桌面本地库单人自用，在线业务库多人共享，两者在同一检索接缝下并存。桌面本地库经评估可行，目标区间为 10 万页级别，瓶颈在 OCR 而非向量化，建议按全文检索先行、向量后置的三阶段路线实施。

## Table of Contents

- [一、dsh 现状盘点](#一dsh-现状盘点)
- [二、产品设计](#二产品设计)
- [三、插件架构](#三插件架构)
- [四、能力逐项设计](#四能力逐项设计)
- [五、扩展能力候选](#五扩展能力候选)
- [六、风险与落地节奏](#六风险与落地节奏)
- [七、待决策项](#七待决策项)

-----

<a id="一dsh-现状盘点"></a>
## 一、dsh 现状盘点

### 1.1 分层骨架

```
dsh --profile <name>          # 唯一合法的应用启动入口（禁止自建 bin）
  └─ profile = 有序 bundle 列表 + cordis.patch.yml
       └─ bundle = 一组 Cordis 配置行 + 它挂载的代码
            └─ plugin = 服务 / 工具 / 事件监听 / UI 插槽
```

dsh 没有特权内核：模型适配器、工具注册表、会话日志、agent-loop 本身都是插件，都能被上层 patch 替换。详见 [architecture.md](../architecture.md)。

配置叠加顺序为：bundle（按 profile 声明顺序）→ profile 的 `cordis.patch.yml` → home 级 patch → `--patch` 覆盖层。这意味着企业定制不需要 fork，只要在 `dsh-base` 之上再叠一层自己的 bundle。

现有 profile 为 `web`、`headless`、`sdk`、`sdk-minimal`、`acp`；Desktop 独占 `$DSH_HOME/profiles/desktop`。

### 1.2 必须遵守的包约定

| 约定 | 内容 |
|---|---|
| 目录 | `packages/<group>/<pkg>/`，group 是纯容器（无 package.json、无源码） |
| 命名 | 每个 npm 包 `@deepseek-ai/dsh-<name>`；`@deepseek-ai/cordis` 是每个包的 peerDependency |
| 能力接缝 | 一个能力 = Service Definition（接口）+ Service Provider（实现）+ Consumer（通常是模型工具），三角色齐全才叫接缝，角色独立演进才拆包 |
| 注册即效果 | 所有贡献走 `ctx.effect()` / `ctx.on()`，`register()` 返回 disposer |
| 角色命名 | 接口包命名能力本身；实现包加机制、协议、环境或厂商后缀；单数 ctx key 表示单引擎，复数表示注册表 |
| 无硬编码可调参数 | 随部署变化的选择必须是 `Config` 字段，可从 cordis.yml 修改 |
| 模型可见即已记录 | 任何进入模型请求的内容都必须能从 session log 重建 |

现成的三件套模板：[`shell/`](../../packages/shell/README.md)、[`fs/`](../../packages/fs/README.md)、[`skill/`](../../packages/skill/README.md)、[`web/`](../../packages/web/README.md)、[`llm/`](../../packages/llm/README.md)。完整包组地图见 [packages/README.md](../../packages/README.md)。

### 1.3 可直接复用的现有能力

| 需求方向 | 现有资产 |
|---|---|
| 模型接入 | `ctx.llm` 适配器接缝，已有 deepseek / pi-ai / retry / token-meter |
| 工具注册 | `ctx.tools` 与 `tools/pre-execute` → `tools/execute` → `tools/post-execute` 瀑布，可做横切拦截 |
| 技能 | `ctx.skills` provider 注册表（filesystem / badge / office）与 `tool-skill` 目录加载器 |
| 专家（子智能体） | `ctx.subagents` provider 注册表，架构文档明确支持「委托给另一个产品的一次回合」 |
| 连接器 | `mcp/mcp-client` 把外部 MCP server 暴露为原生工具 |
| 批量与后台 | `ctx.jobs` 后台作业、`ctx.workflowEngine` PTC 工作流、`ctx.webhookRuntime` 外部事件起会话 |
| 权限与审批 | `ctx.approval`、`ctx.permissionPresets`、`tool-ask-user`、`sandbox` 进程隔离 |
| 凭据 | `ctx.credentials`（配置只写名字不写值）与 `ctx.authorization`（需要人参与的授权流） |
| 持久化 | append-only session log、投影接缝 `ctx.sessionProjections`、`storage-sqlite` / `storage-json` / `storage-domain` |
| 附件与溢出 | `ctx.attachments` 内容寻址存储、`ctx.spillStore` 大结果不灌上下文 |
| 交付物 | `deliverables` 回合产出文件与工作区变更记录 |
| 组合 | `preset/agent-preset` 每会话不同能力集、`plugin-manager` 运行时装卸插件 |
| 前端 | `client/ui-*` 与 `ui-slots` 四种插槽（single / list / keyed / chain）、`ui-theme` 的 `--dsw-*` token |

### 1.4 需要我们补齐的空白

仓库中没有 embedding、向量检索、rerank、知识库、OCR 的任何实现；`session-query-sqlite` 只提供 FTS5 全文检索。

没有企业身份：只有 `identity`（匿名标识）和 `deepseek-account`（公有云账号），没有 SSO、组织、角色。

没有数据级 RBAC：现有权限解决的是「工具能不能执行」（沙箱与审批），不是「这条档案你能不能看」。

皮肤能力不足：`ui-theme` 只支持 light / dark / system 三值，第三方主题只能注册 alias token 覆盖层，且第三方 theme id 不进内置 settings schema，没有一等的皮肤目录与选择界面。

没有档案领域模型：全宗、案卷、件、档号、密级、开放状态这些概念完全缺失。

-----

<a id="二产品设计"></a>
## 二、产品设计

### 2.1 定位与形态

内网优先的桌面档案智能体：Electron 桌面端（复用 `apps/desktop`）为主，数据不出内网，模型可切内网部署；同时保留 Web 形态供档案室集中部署。

价值主张：把档案人员从「移交—著录—鉴定—编研—利用」全链条的重复劳动中解放出来，且每一步都可追溯、可审计、可复核。

### 2.2 角色与核心场景

| 角色 | 核心场景 | 当前痛点 |
|---|---|---|
| 档案接收员 | 进馆移交、四性检测 | 批量检测靠人工抽检，各省标准不一 |
| 著录员 | OCR、自动著录、校对 | 逐件敲元数据，最耗工时 |
| 鉴定与开放审核员 | 敏感信息筛查、开放意见 | 逐页审阅，怕漏怕错，要双人复核 |
| 编研人员 | 专题汇编、统计分析可视化 | 跨库检索难，出图靠导出 Excel |
| 利用窗口 | 查档问答、出具证明 | 找不到、找不全，权限边界模糊 |
| 档案室主任 | 进度、质量、利用率看板与审计 | 看不到过程，出事查不清 |

### 2.3 三条不可动摇的产品原则

**结论由工具产出，模型只做编排与表述。** 四性检测、敏感筛查、去重、格式合规都是确定性计算，不能让模型「看一眼判断」。模型的职责是拆解任务、调用工具、汇总成人话、给出引用。

**每条结论必须带证据与引用**（档号、页码、判定依据条款）。这直接对应 dsh 的「模型可见即已记录」约束。

**权限在检索层过滤，不在生成层过滤。** 模型永远不应看到用户无权看的内容，而不是「看到了但不说」。

-----

<a id="三插件架构"></a>
## 三、插件架构

### 3.1 顶层决策：独立仓库，不 fork dsh

```
kaifang-archive-agent/            # 新仓库，npm scope @kaifang/aa-*
  packages/<group>/<pkg>/
  profiles/archive-desktop/       # dsh --profile archive-desktop
```

dsh 的 profile / bundle 层正是为此设计；fork 会在每次上游升级时付出巨大成本。只有当我们需要新的能力接缝本身时才向上游提 PR（下文标注 `[上游]` 的三处）。

启动方式严格遵守 dsh 的 application-launch 规则：只能通过 `dsh --profile archive-desktop` 启动，不自建可执行文件。

### 3.2 包目录划分

```
packages/
  archive/                        # 档案领域内核（所有包的共同词汇）
    archive-model/                  档号/全宗/案卷/件/门类/密级/开放状态；Branded ids；DA/T 术语表
    archive-schema/                 著录规则 schema 注册表（按门类可插拔）
    archive-classification/         ctx.classification 密级标记与传播策略

  inspection/                     # 四性检测
    inspection/                     ctx.inspections 检测项注册表与报告模型（Service Definition）
    inspection-authenticity/        真实性：签章、哈希链、时间戳、元数据一致性
    inspection-integrity/           完整性：件数、页码连续、附件齐全、校验和
    inspection-usability/           可用性：可打开、清晰度、偏斜、字符可提取、格式合规（PDF/A、OFD）
    inspection-security/            安全性：病毒扫描、加密状态、水印、访问控制
    inspection-bizsys/              业务系统下发的地方性检测项（远端执行）
    tool-inspection/                Consumer：archive_inspect 工具与 /四性检测 命令

  screening/                      # 敏感信息筛查
    screening/                      ctx.screening 筛查接缝与 Finding 模型
    screening-rules/                词表、正则、规则包 provider（规则包可由业务系统下发）
    screening-ner-local/            本地 NER 模型 provider（离线）
    screening-bizsys/               业务系统审查服务 provider
    screening-egress-guard/         横切：tools/pre-execute 拦截敏感内容外发
    tool-screening/                 Consumer 工具与复核卡片元数据

  ocr/
    ocr/                            ctx.ocr Service Definition（带坐标的结构化结果）
    ocr-local/                      本地引擎 provider（离线包）
    ocr-bizsys/                     业务系统 OCR 服务 provider
    tool-ocr/

  catalog/                        # 自动著录
    catalog/                        ctx.catalogExtractors 抽取器注册表
    catalog-rules/                  规则抽取（公文版式、红头、文号、成文日期）
    catalog-llm/                    模型抽取（约束到 schema，强制 JSON 校验）
    catalog-bizsys/
    tool-catalog/                   catalog_extract / catalog_validate / catalog_diff

  ingest/                         # 解析与切分管线（知识库与著录共用）
    ingest/                         ctx.ingestPipeline loader → normalize → chunk 接缝
    ingest-pdf/  ingest-ofd/  ingest-office/  ingest-image/
    ingest-chunk-archive/           按卷—件—页结构切分，保留引用锚点

  retrieval/                      # 知识库检索
    retrieval/                      ctx.retrieval 统一检索接缝（带 principal 与 ACL 过滤）
    retrieval-local-sqlite/         本地：FTS5 + 向量（sqlite-vec）+ RRF 融合
    retrieval-bizsys/               业务系统知识库 provider（用户令牌透传）
    retrieval-rerank-local/         可选 rerank provider
    embedding/                      [上游] ctx.embeddings 向量化接缝，值得提给 dsh
    embedding-local-onnx/           本地 onnxruntime 嵌入模型（内网离线）
    embedding-bizsys/
    tool-retrieval/                 kb_search / kb_cite 工具

  bizsys/                         # 业务系统集成总线
    bizsys/                         ctx.bizsys 租户、端点、会话令牌、健康检查
    bizsys-auth/                    SSO / OAuth2 / 国密登录，注册到 ctx.authorization
    bizsys-directory/               组织、角色、全宗授权同步
    bizsys-writeback/               著录结果与检测报告回写业务系统
    connector/                      ctx.connectors 描述式连接器（OpenAPI/HTTP 动态生成 tools）
    connector-bizsys/               连接器定义由业务系统下发
    （MCP 连接器直接复用 dsh-mcp-client）

  authz/                          # 数据级权限
    authz/                          ctx.principals / ctx.policy 主体与 ABAC 策略接缝
    authz-bizsys/                   策略来自业务系统
    authz-tool-guard/               横切：tools/pre-execute 按主体裁剪工具与参数
    authz-retrieval-filter/         横切：检索硬过滤（pre-filter）

  expert/                         # 专家
    expert/                         ctx.experts 专家目录（本地 preset 与远端下发）
    expert-bizsys/                  注册为 ctx.subagents provider，委托远端专家回合
    tool-expert/                    ask_expert 工具与 @专家 提及

  workbench/                      # 工作台
    workbench/                      ctx.workbenchApps 工作台应用注册表（入口表单 → 会话/preset/goal）
    workbench-batch/                批处理：一批档案 × 一条流程，基于 ctx.jobs 与 workflowEngine
    workbench-bizsys-todo/          业务系统待办 → webhook → 会话

  analytics/                      # 数据分析可视化
    analytics/                      ctx.archiveQueries 受权数据集查询接缝
    analytics-bizsys/  analytics-local/
    tool-analytics/                 产出图表 spec（非模型画图），由客户端渲染

  preservation/                   # 长期保存（可后置）
    format-convert/                 PDF/A、OFD 转换与迁移
    dedupe/                         simhash 与向量重复件检测

  audit/                          # 合规留痕
    audit/                          session log → 审计投影（ctx.sessionProjections）
    audit-hashchain/                日志逐条哈希链与可选 CA 时间戳
    audit-export/                   审计报告导出、对接企业审计系统
    dual-control/                   四眼原则：高风险结论强制二人复核（复用 ctx.approval）

  skin/                           # 皮肤
    skin/                           [上游] ctx.skins 皮肤目录接缝（tokens/字体/纹理/圆角）
    skin-vermilion/                 朱红（默认）
    skin-kraft/                     牛皮纸
    client/ui-skin-settings/        皮肤选择界面

  policy/                         # 组织策略下发
    skill-bizsys/                   组织级 skill 包下发（版本、签名、适用角色）
    preset-bizsys/                  agent preset 下发
    model-policy/                   横切：按会话密级选路或拒绝模型（agent/request）
    package-signature/              插件与技能签名校验，拒绝未签名

  client/                         # 全部前端插件
    ui-workbench/  ui-inspection/  ui-screening/  ui-catalog/  ui-kb/
    ui-expert/  ui-analytics/  ui-audit/  ui-archive-brand/  ui-classification-badge/

  bundle/
    archive-base/                   企业内核层（叠在 dsh-base 之上）
    archive-desktop-app/            桌面形态
    archive-headless/               批处理与服务端形态
```

分组原则：按能力族分组，不按技术层分组，与 dsh 一致。每族内部严格保持 `<cap>/` + `<cap>-<机制>/` + `tool-<cap>/` 三角色。

### 3.3 profile 与 bundle 组合

```yaml
# profiles/archive-desktop/package.json
{ "dsh": { "profile": { "bundles": [
    "@deepseek-ai/dsh-base",
    "@deepseek-ai/dsh-web-app",
    "@kaifang/aa-archive-base",
    "@kaifang/aa-archive-desktop-app"
] } } }
```

```yaml
# packages/bundle/archive-base/cordis.patch.yml（片段）
- id: bizsys
  name: '@kaifang/aa-bizsys'
  config:
    endpoint: ${ARCHIVE_BIZSYS_ENDPOINT}
    credential: bizsys-token          # 只写名字，值在 ctx.credentials
- id: authz-bizsys
  name: '@kaifang/aa-authz-bizsys'
- id: retrieval
  name: '@kaifang/aa-retrieval'
  config: { fusion: rrf, topK: 12 }
- id: retrieval-bizsys
  name: '@kaifang/aa-retrieval-bizsys'
  config: { delegation: on-behalf-of }   # 用户令牌透传，禁止服务账号
- id: retrieval-local-sqlite
  name: '@kaifang/aa-retrieval-local-sqlite'
  config: { embeddings: local-onnx, index: $DSH_HOME/archive/kb.db }
- id: skin-vermilion
  name: '@kaifang/aa-skin-vermilion'
  config: { default: true }
- id: inspection
  name: '@kaifang/aa-inspection'
  config:
    profile: DA-T-70                   # 检测规程可切换（省级细则不同）
    checks: [authenticity, integrity, usability, security]
```

每个检测项、每个筛查规则包、每个著录门类都是独立插件行，各省细则差异靠修改 YAML 解决而不改代码，正好落在 dsh 的「无硬编码可调参数」约束上。

-----

<a id="四能力逐项设计"></a>
## 四、能力逐项设计

### 4.1 档案行业工具

**四性检测。** `ctx.inspections` 是检测项注册表，每个检测项一个插件，返回 `{ check, verdict, evidence[], basis }`。工具 `archive_inspect` 只负责调度与汇总，报告落 `deliverables`。模型不参与判定，只解释结论和给整改建议。

**敏感信息筛查。** 多 provider 叠加（规则包、本地 NER、业务系统），输出带位置与置信度的 Finding。关键横切件是 `screening-egress-guard`：挂在 `tools/pre-execute` 瀑布上，拦截把敏感内容送往外发工具（web_fetch、邮件、外网模型）的调用。

**OCR。** 结果带坐标，原文落 `ctx.attachments`，超长走 `ctx.spillStore`，只把摘要与定位信息送入上下文。必须支持 OFD（国产电子文件格式，档案行业刚需）与多页 TIFF。

**自动著录。** `catalog-rules` 先跑确定性抽取（文号、成文日期、密级、责任者），剩余字段交 `catalog-llm` 并强制 schema 校验；`catalog_diff` 给出机器值与人工值的对照供校对，校对动作走 `ctx.userQuestions` 或客户端卡片。

**数据分析可视化。** `tool-analytics` 产出图表规格（vega-lite 风格 spec）与数据引用，由 `ui-analytics` 渲染，不让模型手写 SVG。这符合 dsh 的「Host presenter 保持纯函数，Web 卡片从原始事件与持久化结果元数据推导」约定。

### 4.2 知识库：本地向量化与权限

两个接缝：`ctx.retrieval`（检索）与 `ctx.embeddings`（向量化）。检索接缝统一签名 `search(query, { principal, scopes, filters, topK })`，业务系统知识库与本地知识库是同一接缝下的两个 provider，模型只看到一个 `kb_search` 工具。

本地向量化方案：

| 环节 | 方案 |
|---|---|
| 解析 | `ingest-*` 按格式分插件；扫描件先过 OCR |
| 切分 | 按卷—件—页结构切分，chunk 携带 `{ 全宗号, 目录号, 案卷号, 件号, 页码 }` 作为引用锚点 |
| 嵌入 | 本地 onnxruntime 运行 bge 系列模型，离线随桌面端分发；批量建索引走 `ctx.jobs` 后台作业 |
| 存储 | SQLite 加向量扩展（单文件、可随桌面端分发、无需额外服务）；大规模场景换 Milvus 或 ES-kNN provider |
| 检索 | FTS5（BM25）与向量双路召回，RRF 融合，可选本地 rerank |
| 增量 | 文件指纹加 mtime；索引库使用单调 `SCHEMA_VERSION`，遵循 dsh 的 SQLite 约定 |

权限设计（最容易翻车的部分）。两类知识库的权限模型已确定，且**必须分开设计**：

| | 在线业务知识库 | 桌面本地知识库 |
|---|---|---|
| 使用者 | 多人共享，来自业务系统集成 | 仅本人维护与使用，不共享 |
| 权限判定方 | 业务系统（权威） | 操作系统用户即边界 |
| chunk 级 ACL | 必需 | 不需要 |
| 检索期过滤 | 必需（按 principal 硬过滤） | 不需要 |
| 鉴权方式 | 用户令牌透传（on-behalf-of） | 无 |
| 落盘加密 | 由业务系统侧负责 | 按密级自动开启（见 4.3） |

**在线库：硬过滤而非后过滤。** 索引时给每个 chunk 打 ACL 标签（来源系统、全宗、密级、开放状态、部门、角色）；向量检索必须 pre-filter，否则 topK 被无权内容占满，等于降低了有权内容的召回率。

**在线库：委托鉴权，不用服务账号。** 使用用户令牌透传（on-behalf-of）；用服务账号拉全量等于系统性越权。

**两类库共用：密级不降级。** 本次检索命中的最高密级决定会话密级标记（`ctx.classification`），密级会话自动禁用外发类工具。本地库不因为「是我自己的」而豁免这条。

**两类库共用：可撤回。** 文档撤密、销毁或权限变更时索引要能定向删除，chunk 必须保留反查键。

**两类库共用：可事后复查。** 每条引用记录 docId 与当时的权限判定快照到 session log，出事后能回答「他当时到底有没有权限看这条」。

**混合检索时标注来源。** 本地库与在线库结果融合后，每条引用必须携带来源标识；把本地来源的内容回写业务系统前需显式确认，避免未经审核的个人材料进入正式档案。

### 4.3 桌面端本地知识库可行性评估

结论：**可行，且建议做**，但可行性的边界是「页数 × 是否需要 OCR」，不是「档案件数」。单人自用这一前提去掉了最昂贵的部分（chunk 级 ACL、检索期鉴权、多用户并发），剩下的是纯工程量问题。

#### 规模与成本估算

下表按单人桌面（8 核 CPU、无独立显卡）估算，用于判断量级而非承诺性能；chunk 按每页约 1.2 个、每 chunk 约 500 字计。

| 规模 | chunk 数 | 向量体积（float32 / int8 / binary） | 首次建索引（仅嵌入） | 单次检索延迟 | 判断 |
|---|---|---|---|---|---|
| 个人工作集 1 万页 | 约 1.2 万 | 25 MB / 6 MB / 0.8 MB | 1 到 2 分钟 | 10 ms 级 | 毫无压力 |
| 部门级 10 万页 | 约 12 万 | 246 MB / 61 MB / 8 MB | 10 到 20 分钟 | 50 到 150 ms | 目标区间 |
| 大型 80 万页 | 约 100 万 | 2 GB / 512 MB / 64 MB | 2 到 3 小时 | 0.5 到 1.5 s | 需量化粗排 |
| 全馆级 500 万页以上 | 600 万以上 | 12 GB 以上 | 10 小时以上 | 不可接受 | 改用在线库或独立向量服务 |

嵌入模型取 bge-small-zh 级别（约 24M 参数、512 维），int8 量化后在 onnxruntime CPU 上运行，离线随桌面端分发。

#### 三个决定可行性的技术事实

**sqlite-vec 是穷举检索，没有 ANN 索引。** 它的 `vec0` 虚表支持 float、int8、binary 三种向量，支持元数据列与分区键（partition key）在查询期做预过滤，官方推荐的扩展手段是二值量化粗排加原始浮点重排，而不是近似索引。这正好解释了上表的分档：百万 chunk 以内穷举是划算的，再往上必须换 provider。它是纯 C、无外部依赖、可随桌面端分发的单文件方案，与「内网离线」要求高度契合。

**OCR 才是真瓶颈，不是嵌入。** 本地 CPU OCR 约每页 1 到 3 秒，10 万页需要 28 到 83 小时，比嵌入慢两个数量级。因此**不做全量本地 OCR**：入库时只提取已有文本层，无文本层的件登记为「待 OCR」，在用户首次打开或显式勾选时才 OCR 并补索引。业务系统已 OCR 过的文本应优先复用，不重复识别。

**FTS5 的中文分词必须显式选型。** 内置 `unicode61` 分词器对中文按字切分，BM25 质量很差。用 `trigram` 分词器（SQLite 3.34 以上）或自带分词词典，这个决定要在 L0 阶段就定下来，后期更换需要重建全量索引。

#### 单人前提消除的成本，与新引入的风险

消除的：chunk 级 ACL 标签、检索期 principal 过滤、委托鉴权、多用户并发与隔离、索引的多租户分区。这些是在线库的成本，本地库一概不需要。

保留的：落盘加密（做成 `Config` 开关，命中涉密内容时自动开启，密钥存系统 keychain）、密级传播、审计留痕。

**新引入的核心风险：本地库会成为业务系统权限的旁路。** 用户把当前有权查看的档案导出到本地入库后，调岗、撤权或档案改变密级时，本地副本依然可被检索——这是单人本地库最需要产品层堵住的漏洞，而不是技术问题。四条对策：

- 业务系统导出物入库时保留来源标签、原 docId 与原密级，不允许剥离；
- `authz-bizsys` 周期性校验本地库中业务来源条目的有效性，权限撤销后自动隐藏或删除；
- 提供企业策略开关 `localKb.allowBizsysDerived: false`，直接禁止业务系统内容进入本地库；
- 本地库的入库与检索同样写审计日志，这是出事后唯一能复原的证据。

#### 推荐实现路线

关键判断：**本地知识库的产品风险在「引用是否可信、密级是否正确传播、权限旁路是否堵住」，这些与向量无关。** 因此先用全文检索把产品机制跑通，向量只是召回率的增量，这样能把 M4 的风险前移消化。

| 阶段 | 内容 | 上限 | 挂靠里程碑 |
|---|---|---|---|
| L0 | 仅文本类文档 + FTS5（trigram 分词），不上向量；跑通入库、引用锚点、密级传播、权限旁路对策、审计 | 不限（全文检索） | 随 M1 |
| L1 | 接入 `embedding-local-onnx` + sqlite-vec float32；FTS5 与向量双路召回 RRF 融合 | 约 20 万 chunk | M4 前半 |
| L2 | 二值量化粗排 + 浮点重排；按需 OCR 补索引；权限回收校验落地 | 约 100 万 chunk | M4 后半 |
| L3 | 超规模时才做：换独立向量服务 provider（Milvus 或 ES-kNN） | 按需 | 客户实际触顶后 |

L0 到 L2 全部在 `retrieval-local-sqlite` 一个包内演进，对 `ctx.retrieval` 接缝与 `kb_search` 工具零改动；L3 只是新增一个 provider。这是把接缝设计对的直接收益。

### 4.4 模型与 Skills：业务系统与本地并存

**模型。** `llm-bizsys-gateway` 注册到现有 `ctx.llm`（OpenAI 兼容或自研协议，携带 SSO token 与配额）。本地配置沿用现成的 `ui-settings-models` 与 `ctx.credentials`。新增 `model-policy` 横切插件监听 `agent/request`，按会话密级与模型部署位置做选路矩阵：涉密数据只能走内网模型，否则直接拒绝并给出原因。

**Skills。** `ctx.skills` 已是 provider 注册表，新增 `skill-bizsys` provider 拉取组织级技能包（版本、签名、适用角色），缓存本地保证离线可用；冲突解析复用现成的 winning-skill 机制，策略上组织强制技能优先于用户自定义。同理下发 `preset-bizsys`（角色预设）。所有下发物经 `package-signature` 校验，拒绝未签名内容。

### 4.5 专家与连接器

**专家即 subagent provider。** dsh 架构文档明确 subagent provider 可以是「另一个产品里的一次委托回合」，这正是为远端专家准备的。`expert-bizsys` 注册为 provider，把回合委托给业务系统侧的领域智能体，流式回传，子会话事件照常落本地 session log，保证可追溯性不因远端执行而断链。前端 `ui-expert` 提供 @专家 提及与专家卡片。

**连接器分两档。** 业务系统能提供 MCP server 的，直接用现成的 `mcp-client`，零开发；只有 HTTP 或 WebService 的老系统，走 `ctx.connectors` 描述式连接器：一份 OpenAPI 或描述文件加凭据引用与参数映射，运行时动态生成工具注册到 `ctx.tools`，连接器定义本身可由业务系统下发，换客户不改代码。每个连接器工具必须声明副作用等级（只读、写入、不可逆），自动接入 `ctx.approval`。

### 4.6 工作台

`ctx.workbenchApps` 注册工作台应用：图标、标题、入口表单 schema、启动时用哪个 preset 创建什么会话与目标。批量能力复用 `ctx.jobs` 与 `ctx.workflowEngine`（PTC 引擎已存在）：一批档案乘一条流程等于 N 个子会话，工作台汇总进度与失败重试。业务系统待办通过 `webhook` 触发会话，dsh 已有可信规则与工作区会话机制。

### 4.7 皮肤

现状不足：`ui-theme` 只有 light / dark / system 三值，第三方主题只能注册 alias token 覆盖层且不进内置 settings schema。

设计：新增 `ctx.skins` 皮肤目录接缝，`SkinDefinition` 包含 id、名称、预览图、token 覆盖层、字体、纹理资源与圆角规则；`ui-skin-settings` 提供选择界面；默认值由 `archive-base` bundle 配置为朱红。

两条约束：皮肤只能改 token 与资源，不能改布局逻辑，否则维护成本失控；纹理与字体必须打包进插件，禁止外链 CDN（内网离线）。

`[上游]` 建议向 dsh 提 PR，把第三方皮肤从 in-process 扩展提升为一等皮肤目录，否则产品侧要复制一份 `ui-theme` 的持久化逻辑。另需注意 dsh 有 `verify-client-ui-i18n` 门禁，所有前端文案必须走 locale 字典。

-----

<a id="五扩展能力候选"></a>
## 五、扩展能力候选

### 5.1 强相关，建议进入 v1 范围

**审计投影 `audit`。** 把 append-only session log 投影成审计视图（谁、何时、用什么模型、访问了哪些档案、输出到哪），一键导出审计报告。dsh 的日志与投影接缝天然适配，这是相对竞品的硬优势。

**哈希链留痕 `audit-hashchain`。** 会话日志逐条哈希链加可选 CA 时间戳。这与四性检测中的「真实性」互相呼应：我们自己的产出也要经得起四性检测。

**四眼原则 `dual-control`。** 敏感筛查结论与开放鉴定意见强制二人复核，复用 `ctx.approval`。

**密级水印 `classification`。** 会话密级标记，导出与复制加水印（可见与隐写），密级会话禁止外发。

**国产化适配。** 麒麟与统信操作系统、龙芯与鲲鹏芯片，Electron 打包与签名，内网模型部署。这是档案行业准入门槛，必须在 M0 决策，不能后补。

### 5.2 次优先

格式转换与长期保存（PDF/A、OFD 迁移策略）。

重复件与相似件检测（simhash 与向量）。

数字化质检（清晰度、偏斜、页码连续性），可直接做成 `inspection-usability` 的子检测项。

档号与分类号自动推荐。

知识图谱（人物、机构、事件抽取）作为第三个 retrieval provider。

插件灰度与回滚，`plugin-manager` 已有装卸能力，补一层策略。

### 5.3 必须定义的度量

著录采纳率、机器值纠错率、单件节省工时、检测漏判率与误判率、查档一次命中率。缺少这几个数，产品无法向客户证明价值。

-----

<a id="六风险与落地节奏"></a>
## 六、风险与落地节奏

| 风险 | 对策 |
|---|---|
| 幻觉在档案领域不可接受 | 结论一律由工具产出并强制引用；高风险动作强制人工确认 |
| 各省四性检测细则不同 | 每个检测项一个插件加 YAML 配置规程，不硬编码 |
| 业务系统异构 | 描述式连接器与 MCP 优先，避免为每家客户改代码 |
| 内网离线 | 模型、OCR、嵌入模型、字体、向量库全部可离线分发 |
| 上游 dsh 演进 | 不 fork，只叠 bundle；需要新接缝时向上游提 PR |

### 6.1 里程碑与验收

每个里程碑以「可验收信号」结束，而不是以「代码写完」结束。本地知识库的 L0 到 L2 分阶段挂靠在 M1 与 M4 上（见 4.3）。

| 里程碑 | 交付物 | 验收信号 |
|---|---|---|
| **M0 内核** | profile 与 bundle 骨架、朱红皮肤、SSO 与组织同步、业务系统连接器、`ctx.retrieval` 接缝（只接在线业务知识库，不自建向量） | 用企业账号登录后能检索到在线知识库内容，且换一个低权限账号检索结果显著变少 |
| **M1 单点工具** | OCR、自动著录、四性检测（3 到 5 个高频检测项）、本地知识库 L0（FTS5 全文） | 一件真实档案跑完「OCR 到著录到检测」全流程，产出带引用的报告；本地库入库的涉密件能正确给会话打密级并禁用外发工具 |
| **M2 规模化** | 工作台、批处理（`ctx.jobs` 与 `ctx.workflowEngine`）、回写业务系统 | 一个批次 100 件档案无人值守跑完，失败项可单独重试，结果成功回写 |
| **M3 合规** | 敏感信息筛查、审计投影、哈希链、双人复核 | 能导出一份完整审计报告回答「谁在何时看了哪些档案」；高风险结论未经二人确认无法提交 |
| **M4 深化** | 本地知识库 L1 与 L2（向量与量化）、专家与技能下发、皮肤可插拔 | 10 万页本地库检索延迟稳定在 200 ms 内；撤权后本地库中对应业务来源条目自动失效 |

### 6.2 优先级建议

M0 与 M1 决定产品能不能进场，M3 决定产品能不能通过验收，M2 决定单客户能否放量。**M4 的本地向量库不要提前**：L0 的全文检索已能覆盖单人自用的多数查找场景，把向量放到机制验证之后，可以避免在产品形态未定时就背上嵌入模型分发、索引重建与规模调优的包袱。

-----

<a id="七待决策项"></a>
## 七、待决策项

### 7.1 已决策

**知识库采用双层模型。** 桌面本地知识库仅本人维护与使用、不共享；在线业务知识库从业务系统集成、支持多人。两者在 `ctx.retrieval` 同一接缝下作为两个 provider 并存，权限模型分开设计（见 4.2），本地库的可行性与实现路线见 4.3。

### 7.2 待决策

**首批目标客户是省级档案馆还是企业档案室。** 若是前者，国产化适配与地方检测细则的优先级要提到 M0。

**本地知识库的落盘加密是默认全开还是按密级触发。** 全开对用户无感但影响建索引吞吐；按密级触发需要先有可靠的密级识别，而密级识别本身依赖敏感筛查（M3），存在先后依赖。

**业务系统导出内容是否允许进入本地库。** 允许则本地库价值更高但需要 4.3 的权限回收机制；禁止则实现简单但本地库只能装用户自己产生的材料，使用率存疑。建议做成企业可配策略而非产品硬性选择。
