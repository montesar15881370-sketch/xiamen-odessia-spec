# 行程规划智能体架构调研与对比报告（开源项目 + 四家 Harness + A2A / MCP）

> 生成时间：2026-11-16 14:40（台北时间，UTC+8）· 来源：用户语音通话委托的深度研究 · 状态：调研报告，**只新增本文件，不修改仓库中任何已有文件**。
> 对照对象：`consolidated-spec-v2.md`（17 节）与 `hotel-frontdesk-note.md`（酒店两套流程并存）。
> 写作约定：GitHub 星标数、分叉数均为 2026-11-16 当天通过 GitHub 公开 API 实时读取的数值，会随时间变化；文档没写明的一律标「待核实」；「建议 / 推断」是本文提出的意见，不是来源原文。
> 术语：**Harness**（智能体运行框架，指包在模型外面、负责循环、工具、上下文、权限、记录的那一层运行时）；**MCP**（Model Context Protocol，模型上下文协议，连接智能体与工具 / 数据）；**A2A**（Agent2Agent，智能体间通信协议，连接智能体与智能体）；**POI**（兴趣点）；**RAG**（检索增强生成）；**SSE**（服务端推送事件）；**OTel**（OpenTelemetry，可观测性标准）。

---

## 目录

1. 结论速览
2. 我们的方向与评估标准
3. 开源行程规划 / 旅游智能体项目（含「智旅云图」）
4. DeepSeek 的 Harness 架构
5. OpenAI 的 Harness 架构
6. Anthropic Claude 的 Harness 架构
7. xAI Grok 的 Bot 架构（只用公开资料）
8. A2A 与 MCP：两个协议在我们方向上的位置
9. 总对比表
10. 结合 `consolidated-spec-v2.md`：可以优化的地方
11. 结合 `consolidated-spec-v2.md`：可以并进来的东西（带优先级）
12. 矛盾与待决策
13. 待核实清单
14. 参考来源

---

## 1. 结论速览

1. **「智旅云图」已找到**：GitHub 仓库 `tutu-zzz/zhilv-yuntu`，FastAPI + LangChain + ChromaDB + 高德 Web 服务，**371 星、66 分叉**（2026-11-16 读取）。用户记的 371 星与实时数字一致。它直接用 HTTP 调高德 Web 服务 API，**没有走高德 MCP**。
2. **最值得借鉴的开源做法有三个**，都和我们第 5 节「防幻觉」直接对口：
   - 智旅云图的「**模型只输出候选 POI ID，服务层校验后回填真实名称、地址、坐标**」；
   - `Kyle-liu888/langgraph-trip-planner` 的「**生成 → 校验 → 带错误重试 → 多候选择优**」LangGraph 流水线；
   - `xiaojiune/TravelPal` 的「**LLM 只做理解，排程交给约束求解器**」（营业时间窗、停留时长、驾车耗时）。
3. **四家 Harness 的共同骨架是同一个循环**：模型 → 工具调用 → 结果回填 → 再调模型，直到不再调用工具。差别在外围：DeepSeek Harness 把一切做成可替换插件；OpenAI Agents SDK 原语最少（智能体、交接、护栏、追踪）；Claude 侧重上下文工程、Skills 渐进加载与子智能体；Grok Bot 是产品级「持久云电脑 + 连接器 + 技能 + 例程 + 审批」。
4. **和我们最契合的是「LangGraph 写死流程 + 插件化工具 + MCP 接入」**，这正是 v2 第 2 节已有的方向（「LangGraph 是骨架，dsh『一切皆插件』是血肉」）。Anthropic《Building effective agents》也主张：流程可预测时优先用写死的工作流，而不是让模型自由决策，这支持我们 3 节「路由规则写死」的做法。
5. **多模型兼容可以落地**：DeepSeek API 同时兼容 OpenAI Chat Completions、OpenAI Responses 和 Anthropic Messages 三种格式；xAI API 兼容 OpenAI Responses。建议在 4.1 加一层「模型适配器」，但要注意 DeepSeek 思考模式带工具时**必须回传 `reasoning_content`，否则返回 400 错误**。
6. **P0 建议（立刻可做）**：① 4.3 插件规范补「读写属性 / 是否需人工确认 / 来源标签」字段并按子图做工具白名单；② 3.1 规划子图把硬信息校验节点改成「ID 约束生成 + 校验重试」；③ 澄清「不做订票」与酒店实时锁房、支付 MCP 的关系（见第 12 节）。
7. **需要提醒的时效信息**：OpenAI 官方在 AgentKit 页面上注明，Agent Builder 与 Evals 产品将于 **2026-11-30 起下线**，推荐改用 Agents SDK。我们不应依赖这两个产品。

---

## 2. 我们的方向与评估标准

用户描述的目标方向：**Harness 架构、插件化、MCP / A2A 协议、不做订票、兼容多模型**。对应到 v2 规格：

| 方向 | v2 现状 | 评估时看什么 |
|---|---|---|
| Harness 架构 | 第 2 节：路由器—工作节点，每个工作节点是一个 LangGraph 实例；主循环子图「模型 → 工具 → 回填」 | 循环是否可控、可中断、可恢复；上下文怎么管 |
| 插件化 | 4.3：每个工具一个文件，声明名称、JSON 模式、超时、降级返回值、监控指标 | 插件能否热插拔、能否按场景限制可见工具 |
| MCP / A2A | 4.2：高德 MCP、微信 MCP、支付宝 MCP；A2A 尚未出现 | 是否原生支持 MCP；能否对外暴露为 MCP 服务 / A2A 智能体 |
| 不做订票 | 1.4：不做机票酒店一句话预订、不提供机票 / 动车工具；酒店走意向单（1.3） | 架构是否方便把「写操作」与「读操作」隔离、加人工闸门 |
| 多模型 | 4.1：Qwen3.8 Omni Flash Realtime 主模型，MOSS、Nex N2.6、MOSS-VL-Realtime 分工；切换备选要重过合规 | 是否绑定单一厂商；接口格式是否通用 |

---

## 3. 开源行程规划 / 旅游智能体项目

### 3.1 智旅云图 `tutu-zzz/zhilv-yuntu`（用户点名项目）

- **仓库**：https://github.com/tutu-zzz/zhilv-yuntu · **371 星 / 66 分叉** · 创建于 2026-04-11 · 最近推送 2026-09-10（UTC 时间戳换算为台北时间约 2026-09-10 16:05）· 主要语言 Python。
- **技术栈**（README 原文）：后端 FastAPI + Pydantic + SQLAlchemy；LangChain；ChromaDB 向量库；Redis 缓存；HTTPX + 高德地图 Web 服务 + 高德 JavaScript API；前端 Vue 3 + Vite；SQLite。
- **分层**（README 表格）：前端 → 接口层（trip / export / weather 路由）→ 服务层（城市解析、动态候选、行程编排、地图补全、天气、缓存、导出、存储）→ Agent 层（本地 RAG Planner、动态 POI ID Planner、LLM 查询改写）→ RAG 层（向量入库、检索、Cross-encoder 重排）→ 数据层（本地 Markdown 攻略）。
- **数据流**：前端收集输入 → 后端判断目的地覆盖等级 → 已沉淀城市（北京、大理、成都、西安、**厦门**、三亚）走本地 RAG；未沉淀城市采集高德景点 / 餐饮 / 住宿 POI，再做「ID 受约束」的规划 → 地图与天气补全 → 前端展示、保存、导出。
- **亮点**：
  - **真实性约束**：动态 Planner 只输出候选 POI ID，服务层校验后回填真实名称、地址、坐标、图片；模型返回越界 ID 时，只从本次真实候选池降级。
  - **失败降级不造实体**：候选不足时明确留空，不再生成「推荐景点 N」这类模板实体。
  - **RAG 工程化指标**：README 记录的 Top1 命中率从 80% 提到 93.3%、MRR（平均倒数排名）0.889 → 0.967，重排缓存把平均时延从 728 毫秒降到 425 毫秒；按目的地隔离防「跨城市污染」。这些是项目自报数字，未独立复现（待核实）。
  - **Token 分项统计**：查询改写、查询向量、重排、Planner 分项记账。
- **不足**：单城市规划；不是 Harness 架构，而是「FastAPI 服务里嵌 LangChain 调用」的固定管线；工具直接调 HTTP API，不走 MCP；没有多智能体、没有人工闸门（因为也不下单）。
- **对我们的适配度**：**高（借做法，不借框架）**。它的厦门攻略库、ID 约束生成、跨城市污染评估可直接参考；架构本身比我们简单。

### 3.2 TripStar 旅途星辰 `1sdv/TripStar`

- **仓库**：https://github.com/1sdv/TripStar · **2555 星** · 许可证 GPL-2.0。
- **架构**：基于 HelloAgents 框架的多智能体协作（天气预报员、酒店推荐专家等），FastAPI 后端 + Vue 3 前端；README 写明经「Google / AMap MCP」调用天气和 POI 搜索，Google 地图优先、失败回退高德。
- **亮点**：用户偏好记忆（带权重、遗忘机制、Top-K 召回）；多城市行程；行程转知识图谱可视化；「景点预约提醒」。
- **不足 / 风险**：攻略和图片来自**抓取小红书 SSR 页面**，存在平台条款与版权风险，和我们 4.2.2「不抓取携程、美团等平台评论」的原则冲突；GPL-2.0 许可证对闭源商用有传染性约束（待法务确认）。
- **对我们的适配度**：中。偏好记忆的「权重 + 遗忘」可参考 4.4；数据来源做法不可照搬。

### 3.3 HelloAgents 第十三章「智能旅行助手」`datawhalechina/hello-agents`

- **仓库**：https://github.com/datawhalechina/hello-agents（教程整体 87079 星）· 示例代码：`code/chapter13/helloagents-trip-planner`。
- **架构**：HelloAgents 的 SimpleAgent；**通过 MCP 协议接入 amap-mcp-server（高德地图）**；后端 FastAPI；章节正文描述 4 个专门 Agent（景点搜索、天气查询、酒店推荐、行程规划）依次调用，每个 Agent 通过 MCP 调外部 API；支持 OpenAI、DeepSeek 等多种模型提供商。
- **对我们的适配度**：中。是「高德 MCP + 多模型」的最小可运行参考，适合新人上手，不适合直接当生产骨架。

### 3.4 LangGraph Trip Planner `Kyle-liu888/langgraph-trip-planner`

- **仓库**：https://github.com/Kyle-liu888/langgraph-trip-planner · **107 星** · 描述「Model-agnostic」（不绑定模型）。
- **架构**（README 原文流程）：FastAPI `POST /api/trips` 立即返回行程 ID → SSE 推送进度并支持断线重放 → 后台 RunManager + PostgreSQL → LangGraph StateGraph：
  `collect_context`（高德景点 / 餐饮 / 酒店 / 天气）→ `build_prompt`（压缩上下文）→ `generate_candidate` → `validate_candidate`（模式 + 日期 + POI + 业务规则）→ `retry`（带校验错误重新生成）→ `select_best_candidate`。
- **亮点**：PostgreSQL Checkpointer 支持中断后恢复；README 明确写「门票显示为预算估算，不是实时票价……没有自动预约、实时余票」，边界写得清楚。
- **对我们的适配度**：**高**。几乎就是我们 3.1「硬信息校验节点」的现成参考实现，且同为 LangGraph。

### 3.5 TravelPal `xiaojiune/TravelPal`

- **仓库**：https://github.com/xiaojiune/TravelPal · **4 星**（体量很小，但思路独特）。
- **架构**：三层——决策层 LLM（LangGraph 编排，理解自然语言、调用 POI 与驾车耗时工具）；**求解层**用压缩退火（CA）/ 变邻域搜索（VNS）元启发式算法，严格约束营业时间窗、停留时长、驾车耗时，README 称以 TSPTW（带时间窗的旅行商问题）基准算例验证；工程层 FastAPI + SSE、Celery + Redis、PostgreSQL，并**对外提供 MCP Server**。
- **对我们的适配度**：中高。它点出了纯 LLM 排程的通病（绕路、营业时间不符），建议我们在 3.1 把「排时间」交给确定性算法（见第 11 节）。

### 3.6 基于 A2A + MCP 的旅行助手 `phoenix-zhou/multi-agent`

- **仓库**：https://github.com/phoenix-zhou/multi-agent · **490 星**。
- **架构**：总控中心做意图识别与路由 → 通过 **A2A 协议**分发给天气查询、票务查询、票务预定等专业 Agent（每个 Agent 带 AgentCard 名片和技能声明，各占一个端口）→ 数据访问以 **MCP 工具**形式暴露，与 Agent 解耦。
- **对我们的适配度**：中。是「A2A 管智能体之间、MCP 管工具」分工的清晰示例；但它包含**票务预定**，与我们「不做订票」相反，只借架构。

### 3.7 Azure AI Travel Agents `Azure-Samples/azure-ai-travel-agents`

- **仓库**：https://github.com/Azure-Samples/azure-ai-travel-agents · **483 星**。
- **架构**：微软官方参考应用，用 LangChain.js、LlamaIndex.TS、Microsoft Agent Framework 三种编排方式；多个 **MCP 服务器**（Python、Node.js、Java、.NET 实现）提供工具；智能体分为需求理解、目的地推荐、行程规划；全部容器化部署到 Azure Container Apps。
- **对我们的适配度**：中。证明「编排框架可换、工具层统一走 MCP」可行，正对应我们的多模型 + 插件化目标；它面向旅行社员工而非游客。

### 3.8 TravelPlanner 基准 `OSU-NLP-Group/TravelPlanner`

- **仓库**：https://github.com/OSU-NLP-Group/TravelPlanner · **570 星** · 论文 arXiv:2402.01622（ICML 2024 Spotlight）。
- **内容**：旅行规划评测基准，提供沙盒环境、近 400 万条数据记录的工具、1225 条规划意图与参考方案。论文摘要原文：当时 **GPT-4 的成功率只有 0.6%**；智能体难以保持任务、选对工具、同时追踪多个约束。
- **对我们的适配度**：**高（评测方法）**。它把「常识约束」与「硬约束」分开打分的思路可以直接用于我们第 8 节评测集。

### 3.9 其他候选（只列不展开）

| 仓库 | 星标 | 一句话 |
|---|---|---|
| `LeeFly-cn/TripStar-Java` | 65 | TripStar 的 Spring Boot + Spring AI Alibaba 版本 |
| `shaheennabi/Production-Ready-TripPlanner-Multi-AI-Agents-Project` | 80 | 多智能体行程规划（英文社区） |
| `lm041520/LangGraph-trip-planner` | 70 | LangGraph 行程规划（无描述，待核实） |
| `Bincode1/TravelPlanner` | 2 | LangGraph + FastMCP + 高德，并行召回景点 / 酒店 / 天气后评审、成稿 |
| `cablate/mcp-google-map` | 499 | Google 地图 MCP（含旅行规划 Agent Skills），海外场景参考 |

### 3.10 开源项目小结

- 国内项目普遍是 **FastAPI + LangChain / LangGraph + 高德 + Vue** 这套组合，和我们技术方向一致。
- 真正的差异化在「**真实性约束**」上：ID 约束生成、校验重试、确定性求解。这正是我们第 5、6 节要求的。
- 没有一个开源项目有我们的「地头库存 + 931 栋档案 + 人工确认闭环」，护城河判断（1.2）仍成立。

---

## 4. DeepSeek 的 Harness 架构

### 4.1 公开内容

- **DeepSeek Harness（`dsh`）**：DeepSeek 官方开源的智能体运行框架，仓库 https://github.com/deepseek-ai/deepseek-harness ，MIT 许可证，2026-08-13 创建；GitHub API 读到 281875 星（数字异常大，原样记录）。README 明确标注「**开发者预览版，会有破坏兼容性的变更**」。
- **核心理念「一切皆插件」**：架构文档写明，dsh 基于 Cordis 框架，插件向共享上下文贡献服务、类型化事件和可撤销的副作用；**模型适配器、工具注册表、会话日志、智能体循环本身都是插件**，都能通过配置替换，「没有需要打补丁的特权内核」。
- **组合方式**：Profile（命名组合，如 `web`、`headless`、`sdk`、`sdk-minimal`、`acp`）由有序的 Bundle 叠加而成，用户可以用 patch 文件替换任意一行配置。
- **循环**：一个 step = 一次模型请求 + 它调用的工具；一个 turn = 零到多个 step。工具执行经过 `tools/pre-execute → tools/execute → tools/post-execute` 三段可拦截事件。会话日志是**只追加**的事件流，模型看到的上下文从日志投影出来；分叉、恢复、转录、遥测都从日志派生。
- **包清单**（仓库 `packages/` 目录，2026-11-16 读取）：`mcp/mcp-client`、`mcp/mcp-manager`；`skill/*`；`subagent/*`（含调用 Claude Code、Codex、ACP 的子智能体提供者）；`compaction/*`（上下文压缩、工具结果裁剪）；`sandbox/*`；`interaction/user-approval`；`plan/plan-mode`；`team/agent-team`；`telemetry/otel`；`llm/llm-pi-ai`（多厂商模型适配，具体覆盖范围待核实）。
- **minimal 模式**：DeepSeek 在模型更新日志里说，其代码智能体评测用的是「DeepSeek Harness minimal mode」。
- **模型侧（DeepSeek API）**：
  - 从 V3.2 开始支持「**思考中调用工具**」（Thinking in Tool-Use）；带 `tools` 参数时，之前所有轮次的 `reasoning_content` 必须完整回传，**否则 API 返回 400**。
  - API 兼容 **OpenAI Chat Completions、OpenAI Responses、Anthropic Messages** 三种格式；更新日志写明 V4 系列「原生支持 Responses API 并适配 Codex」。
  - 2026-11-13 更新：`deepseek-flash` 支持服务端内置工具 `deepseek_web_search`、`deepseek_read_url`、文件输入、安全检查，工具调用与上下文拼装在服务端完成；面向公众的应用**必须开启安全检查**，开发者仍需负责内容审核、大模型备案和 AI 生成内容标识。
  - 只有 Anthropic 与 Responses 格式支持在对话中间插入非模型生成的工具调用；Chat Completions 不支持。

### 4.2 优点

- 插件化最彻底，和我们 4.3「循环子图可替换」理念完全一致；v2 第 2 节其实已经引用了 dsh 的「一切皆插件」。
- 原生 MCP、Skills、子智能体、压缩、审批、OTel 都有对应插件。
- 模型 API 三格式兼容，是多模型方案里「最好换」的一家。
- 服务端内置搜索 / 读网页 / 安全检查，适合境内面向公众的合规要求。

### 4.3 缺点

- 开发者预览，接口会破坏性变更，不适合当生产底座。
- 技术栈是 Node.js / TypeScript（另有 Python SDK 封装），和我们 Python + LangGraph 不同。
- 定位偏「代码 / 通用工作智能体」，不针对实时语音场景（我们主模型是实时语音）。

### 4.4 对我们的适配度

**中高：借设计思想，不直接依赖。** 可借的点：只追加会话日志、工具执行三段拦截、Profile / Bundle 配置化组合。DeepSeek 模型可作为文本规划的备选模型（需合规评估，见第 12 节）。

---

## 5. OpenAI 的 Harness 架构

### 5.1 公开内容

- **Agents SDK**（https://openai.github.io/openai-agents-python/ ，`openai/openai-agents-python` 30701 星）：官方称原语「非常少」——
  - **Agents**：带指令和工具的 LLM，内置循环直到任务完成；
  - **Agents as tools / Handoffs（交接）**：智能体之间委派任务；
  - **Guardrails（护栏）**：输入 / 输出校验，可与智能体执行**并行**运行，不通过就快速失败；
  - **Tracing（追踪）**：内置可视化、调试、监控，可接 OpenAI 的评测与微调工具。
  - 另有：函数工具（自动生成模式、Pydantic 校验）、**MCP 服务器工具调用**、Sessions（会话记忆）、人在回路、Sandbox agents（隔离工作区）、Realtime agents（实时语音智能体）、Voice pipeline（语音转文字 → 智能体 → 文字转语音）；文档有「非 OpenAI 提供商」的模型接入页。
- **Responses API**：Agents SDK 默认底层；支持远程 MCP 工具；`/responses/compact` 端点做上下文压缩（Codex 文章原文）。
- **Codex harness**：《Unrolling the Codex agent loop》（2026-01-23）把 Codex 的核心循环称为 harness：模型要么给出最终回答，要么请求工具调用；harness 执行工具、把输出追加回提示、再查询模型，直到不再调用工具；超过 `auto_compact_limit` 时自动调用压缩端点。
- **Harness engineering**（2026-02-11）：OpenAI 内部团队用 Codex 写了约 100 万行代码、零手写代码的实验；结论是工程师的主要工作变成「设计环境、表达意图、建立反馈回路」，用 `AGENTS.md` 等仓库内文档约束智能体。
- **AgentKit**（2025-10-06）：Agent Builder（可视化画布）、Connector Registry（含第三方 MCP）、ChatKit（嵌入式聊天界面）、Guardrails 开源库、Evals 新能力。**页面顶部注明：Agent Builder 与 Evals 自 2026-11-30 起下线**，代码工作流推荐 Agents SDK。

### 5.2 优点

- 原语少、上手快；交接 + 护栏 + 追踪可以对应私人定制子代理里的转人工（不是独立章）、防幻觉（第 5 节）、监控（4.6）。
- 实时语音智能体有一等支持（中断检测、护栏）。
- 护栏「并行运行、快速失败」的设计适合低时延场景。

### 5.3 缺点

- 默认绑定 OpenAI 模型与 Responses API；接其他厂商要走适配层。
- 托管产品更替快（Agent Builder / Evals 一年内上线又下线），平台依赖风险高。
- 境内面向公众使用 OpenAI 模型需单独评估合规（4.1、第 11 节已有「切换需重过合规」）。

### 5.4 对我们的适配度

**中：借原语，不换框架。** 交接（handoff）带「输入过滤」的设计可作参考；合并规格里不再有交接包专节。护栏可作为第 5 节的实现方式；我们保留 LangGraph。

---

## 6. Anthropic Claude 的 Harness 架构

### 6.1 公开内容

- **《Building effective agents》**：区分「工作流」（预先写好的代码路径编排 LLM 与工具）与「智能体」（LLM 自主决定流程）；总结五种模式——提示链、路由、并行、编排者—工作者、评估者—优化者；主张**能用简单方案就别上复杂智能体**。
- **Claude Agent SDK**（文档 https://docs.claude.com/en/docs/agent-sdk/overview ，`anthropics/claude-agent-sdk-python` 8556 星）：把 Claude Code 的 harness 开放成 SDK。官方博客把循环概括为「**收集上下文 → 执行动作 → 校验工作 → 重复**」。
- **子智能体**（Claude Code 文档）：独立上下文窗口、独立工具权限的专用助手。
- **Agent Skills**：技能是带 `SKILL.md` 的文件夹；启动时只把每个技能的名称和描述放进系统提示，用到时才加载正文——官方称之为「**渐进式披露**」的第一层。
- **上下文工程**（《Effective context engineering for AI agents》）：压缩（compaction）、结构化笔记、子智能体架构、即时检索。
- **代码执行 + MCP**：让智能体用代码按需加载 MCP 工具定义，官方示例把 token 从 15 万降到 2000，节省 98.7%。
- **长时运行 harness**：初始化智能体先建环境（`init.sh`、`claude-progress.txt` 进度日志等），后续每个会话只做增量并留下清晰交接产物。
- **多智能体研究系统**：多智能体（Opus 4 主导 + Sonnet 4 子智能体）在内部研究评测上比单智能体 Opus 4 高 90.2%，但 token 消耗约为普通对话的 15 倍——只适合价值足够高的任务。
- **MCP 本身**由 Anthropic 发起，现行规范版本 2026-07-28（https://modelcontextprotocol.io/specification/2026-07-28 ）。

### 6.2 优点

- 方法论最完整，「什么时候不要用智能体」讲得最清楚。
- Skills 的渐进加载、按需加载工具，直接解决「插件越多、提示越长、越贵越慢」的问题。
- MCP 生态的源头，兼容性最好。

### 6.3 缺点

- Agent SDK 主要面向 Claude 模型；境内调用与合规不可行或需另行评估（待核实）。
- 偏「编码 / 长任务」场景，实时语音不是重点。
- 多智能体成本高，不适合我们 MVP 月成本 < 500 元的目标（10.2）。

### 6.4 对我们的适配度

**高（方法论）、低（直接依赖）。** 第 2、3 节「路由写死」符合其「路由」模式；3.1 规划子图可用「评估者—优化者」模式实现校验重试；4.4 记忆层可引入压缩与结构化笔记；4.2 插件多起来后用 Skills 式渐进加载。

---

## 7. xAI Grok 的 Bot 架构（只用公开资料）

> 说明：下面只引用 xAI 官方文档（docs.x.ai，页面目前署名为 SpaceXSI）和官方新闻页。官方未公开的内部实现，本文不推测。

### 7.1 xAI API 层（开发者可用）

- **两类工具**（Tools Overview）：
  - **内置工具**（服务端执行）：Web Search、X Search、Code Interpreter、Image Generation、Collections Search；
  - **函数调用**（客户端执行）：开发者自定义函数。
- **混合执行**：服务端工具由 xAI 自动执行；遇到客户端工具时暂停并把调用返回给开发者，开发者执行后回填，直到没有客户端工具调用为止。
- **Remote MCP Tools**：在 `tools` 里指定 `server_url`、`server_label`，xAI 代为连接 MCP 服务器；只支持 Streaming HTTP 和 SSE 传输；`allowed_tools` 可只开放部分工具，官方列出的好处是减少上下文开销、降低风险（例如只开只读工具）。OpenAI Responses 的 `require_approval`、`connector_id` 参数目前不支持。
- **接口兼容**：支持 xAI 原生 SDK、OpenAI 兼容的 Responses API、Vercel AI SDK；答案自动附引用来源。
- **网页搜索参数**：`allowed_domains` / `excluded_domains`（各最多 5 个）。

### 7.2 Grok Bot 产品层

- **定位**（官方概述）：主动型 AI 队友，能访问一台云端真实电脑。
- **运行环境**（团队与企业文档）：Grok Bot 是计算机使用型智能体，运行在 Cursor 的云上；每个用户一台专属 Firecracker 微虚拟机，同一用户的所有 Bot 共用这台电脑（浏览器登录、文件、命令行凭据共享），每个 Bot 有自己的屏幕；桌面与手机应用只是聊天、审阅、审批的瘦客户端。
- **连接器（Apps）**：通过 MCP 接入外部服务；OAuth 令牌留在连接器后端，Bot 调工具但拿不到令牌；有结构化连接器时优先用，没有时才用浏览器。
- **技能与例程**：技能是可复用的任务说明（何时用、需要的输入与权限、步骤、如何校验、返回什么、什么需要审批）；例程让某个 Bot 按时间或事件（Slack、GitHub、邮件、Webhook 等）运行。官方「为信任而设计」清单：先自动化准备、再自动化执行；先起草 / 推荐；发送、购买、删除、发布、改生产系统都要审批；要有无数据 / 数据过期策略；重试尽量幂等。
- **审批**：独立的审查模型 Auto Review 在执行命令、使用连接器、操作电脑前检查；需要时向用户弹出审批卡片。
- **多 Bot 协作**：Bot 之间可以交接工作；Grok Bot 1.0 新增团队共享 Bot、主动提出任务、接入 Slack / Microsoft Teams / 视频会议。
- **模型**：团队文档写明「Grok Bot 管理模型选择」，具体用哪个模型未公开。

### 7.3 优点

- 产品级的「持久环境 + 连接器 + 技能 + 例程 + 审批」组合最完整，审批与权限设计成熟。
- API 层服务端工具 + Remote MCP + `allowed_tools`，接 MCP 最省事。

### 7.4 缺点

- Grok Bot 是面向知识工作者的通用助手产品，不是可嵌入我们小程序的框架。
- 内部架构细节未公开；Grok 模型在境内面向公众使用的合规路径未知（待核实）。

### 7.5 对我们的适配度

**中：借治理设计。** 「技能 = 何时用 / 输入 / 步骤 / 校验 / 输出 / 审批」的六要素模板，可以直接用作我们 4.3 插件规范和 7.1 离线管道的模板；`allowed_tools` 式按场景白名单、审批卡片式人工确认，可以落到 4.2 / 4.6。

---

## 8. A2A 与 MCP：两个协议在我们方向上的位置

- **MCP（模型上下文协议）**：智能体 ↔ 工具 / 数据。我们已在用（高德 MCP）。高德官方 MCP Server（https://lbs.amap.com/api/mcp-server/summary ，页面最后更新 2026-03-17）提供：地理编码、逆地理编码、IP 定位、天气、骑行 / 步行 / 驾车 / 公交路径规划、距离测量、关键词搜索、周边搜索、POI 详情，以及 2025 年 5 月升级加入的「**生成专属地图**」（把行程点位导入高德 App）、导航唤端、打车唤端。官方介绍提到「一键导航、打车、订票」体验，是通过唤起高德 App 完成。
- **A2A（智能体间协议）**：智能体 ↔ 智能体。Google 于 2025 年 4 月发起，现由 `a2aproject` 维护（https://github.com/a2aproject/A2A ，26551 星），官网称各语言 SDK 都实现同一份 **A2A v1.0 规范**；核心概念是 AgentCard（智能体名片：能力、技能、端点）、任务委派、不暴露内部记忆与工具（「不透明」）。官网原话：MCP 连接工具与数据，A2A 连接其他智能体，两者一起用。
- **对我们的意义**：
  - **内部**：我们的子图是同一进程内的 LangGraph 节点，不需要 A2A；用 A2A 只会增加时延（14.2 要求端到端 P95 < 2 秒）。
  - **对外**：第 12 节「库存即服务」——把厦门地头库存和 931 栋档案开放给任何模型、眼镜、应用——正是 MCP（给别人的智能体当工具）和 A2A（作为一个「厦门本地智能体」被别人的智能体委派任务）的典型场景；17 节的眼镜厂商技能平台也可能需要这种接入（待核实各平台是否支持 A2A / MCP）。

---

## 9. 总对比表

### 9.1 四家 Harness / Bot 对比

| 维度 | DeepSeek Harness + API | OpenAI Agents SDK + Responses | Claude Agent SDK + 方法论 | xAI API + Grok Bot |
|---|---|---|---|---|
| 核心循环 | step / turn 事件驱动，循环本身是插件 | Runner 内置循环，至无工具调用为止 | 收集上下文 → 执行 → 校验 → 重复 | 服务端工具自动执行 + 客户端工具暂停回填 |
| 插件化程度 | 最高（一切皆插件，配置替换） | 中（函数工具、MCP、交接） | 中高（Skills、子智能体、钩子） | API 层中；产品层技能 / 连接器 |
| MCP | 有 mcp-client / mcp-manager | 内置 MCP 工具调用；Responses 远程 MCP | MCP 发起方 | Remote MCP（HTTP / SSE），`allowed_tools` |
| A2A | 未见官方说明（待核实）；有 ACP 子智能体 | 未见官方说明（待核实） | 未见官方说明（待核实） | 未见官方说明（待核实） |
| 多模型 | API 兼容三种格式；harness 有多厂商适配 | 文档支持非 OpenAI 提供商 | 主要面向 Claude | 兼容 OpenAI Responses |
| 实时语音 | 有独立 ASR / TTS API；harness 不以此为重点 | Realtime agents 一等支持 | 非重点 | API 支持语音对语音（Remote MCP 可用） |
| 人工闸门 | user-approval 插件、沙箱策略 | 人在回路、护栏 | 权限、钩子 | Auto Review + 审批卡片 |
| 成熟度 | 开发者预览，会破坏性变更 | 生产可用；托管产品更替快 | 生产可用 | 产品 1.0；API 稳定 |
| 境内合规 | 境内厂商，路径最清晰 | 需单独评估 | 需单独评估 | 需单独评估 |
| 对我们的定位 | 借插件化设计；DeepSeek 作文本备选模型 | 借交接 / 护栏 / 追踪原语 | 借方法论（工作流优先、上下文工程、Skills） | 借治理设计（技能模板、白名单、审批） |

### 9.2 开源旅行项目对比

| 项目 | 星标 | 编排 | 地图接入 | 真实性约束 | 对我们最有用的一点 |
|---|---|---|---|---|---|
| 智旅云图 zhilv-yuntu | 371 | FastAPI 管线 + LangChain | 高德 Web 服务（HTTP） | ID 约束生成 + 服务层回填 | 3.1 / 第 5 节防编造 |
| TripStar | 2555 | HelloAgents 多智能体 | Google / 高德 MCP | 弱（依赖小红书抓取） | 4.4 偏好记忆（权重 + 遗忘） |
| hello-agents 第 13 章 | 87079（整库） | HelloAgents SimpleAgent | 高德 MCP | 弱 | 高德 MCP 最小示例 |
| langgraph-trip-planner | 107 | LangGraph StateGraph | 高德 | 校验 + 重试 + 择优 | 3.1 硬信息校验节点 |
| TravelPal | 4 | LangGraph + 元启发式求解 | 高德真实路网 | 时间窗硬约束求解 | 3.1 排程交给算法 |
| phoenix-zhou/multi-agent | 490 | 总控 + A2A | MCP 工具层 | 一般 | A2A / MCP 分工示例 |
| azure-ai-travel-agents | 483 | 三种编排框架可换 | 多语言 MCP 服务器 | 一般 | 编排可换、工具统一 MCP |
| TravelPlanner 基准 | 570 | 评测基准 | 沙盒数据 | 常识 + 硬约束评分 | 第 8 节评测方法 |

---

## 10. 结合 `consolidated-spec-v2.md`：可以优化的地方

| # | 我们的章节 | 现状 | 优化建议 | 来源 | 优先级 |
|---|---|---|---|---|---|
| O-1 | 4.3 插件规范 | 名称、JSON 模式、超时、降级返回值、监控指标 | 增加字段：**读 / 写属性**、**是否需人工确认**、**来源标签类型**（自建 / 官方 / 高德）、**适用子图**、版本号。工具层与 MCP 工具描述对齐，便于将来对外暴露 | MCP 规范；xAI `allowed_tools`；Grok Bot 技能六要素 | P0 |
| O-2 | 2、3 节 主循环 / 子图 | 主循环里模型看到全部六个插件 | **按子图做工具白名单**：讲解子图只给只读检索工具；支付 MCP 不进规划子图的模型可见列表，只由人工确认流程调用 | xAI `allowed_tools`；Claude 子智能体独立工具权限 | P0 |
| O-3 | 3.1 硬信息校验节点 | 「输出前强制经过」，没写校验失败后怎么办 | 改成「**ID 约束生成 → 校验（模式 / 日期 / POI / 营业时间 / 业务规则）→ 带错误重试（上限 N 次，待测）→ 多候选择优**」；模型只输出自建库 / 高德候选 ID，名称、地址、价格由服务层回填 | 智旅云图；langgraph-trip-planner；Claude「评估者—优化者」模式 | P0 |
| O-4 | 3.1 行程生成 | 「模板 + 槽位约束 + 实时库存」 | 时间安排（营业时间窗、停留时长、轮渡班次、步行 / 驾车耗时）交给**确定性求解器**，LLM 只负责理解偏好和写讲解文字 | TravelPal；TravelPlanner 论文（GPT-4 成功率 0.6%） | P1 |
| O-5 | 4.1 模型层 | 写死 Qwen 主模型 + 若干专用模型 | 加一层**模型适配器**：统一封装 OpenAI Chat Completions / Responses / Anthropic Messages 三种格式；适配器负责保留 DeepSeek 等厂商的 `reasoning_content`；主模型不变 | DeepSeek API 文档；OpenAI Agents SDK 模型页；xAI 兼容 Responses | P1 |
| O-6 | 4.4 记忆层 | Redis TTL 30 分钟 + 五个语义库 | 补「**会话压缩 + 结构化笔记**」：会话超长时压缩，关键槽位（预算、人数、硬约束）写入结构化笔记，跨会话保留须经用户同意；偏好记忆加权重与遗忘 | Claude 上下文工程；Codex 压缩；TripStar 偏好记忆 | P1 |
| O-7 | 第 5 节 防幻觉 | 拒答、硬事实只来自工具、强制引用、一致性评测 | 把输入 / 输出检查实现为**与主流程并行的护栏**，不通过就快速失败（不增加首包时延）；面向公众时开启模型厂商的安全检查 | OpenAI Guardrails；DeepSeek 安全检查参数 | P1 |
| O-8 | 第 8 节 评测集 | 回归 / 合成 / 对抗 / 一致性 | 增加「**约束通过率**」评测：常识约束（不重复景点、时间不冲突、轮渡可达）与硬约束（预算、人数、偏好槽位）分开打分；加「工具调用轨迹」评测 | TravelPlanner 基准；OpenAI trace grading 思路 | P1 |
| O-9 | 第 9 节 失败模式 | 每个组件四种返回 | 新增两行：「模型适配器」（如思考模式缺 `reasoning_content` 返回 400 → 自动补全或降级到非思考模式）；「外部 MCP 服务器不可用」（高德 MCP 整体不可达 → 自建库 + 缓存路径） | DeepSeek 文档 | P1 |
| O-10 | 4.6 监控层 | LangSmith → LangFuse，OTel 旁路 | 保持不变；明确**不依赖 OpenAI Agent Builder / Evals**（2026-11-30 下线）；追踪片段按 OTel GenAI 语义约定命名（建议，具体约定版本待核实） | OpenAI AgentKit 页面公告 | P2 |
| O-11 | 7.1 知识运营离线管道 | 定时上网搜索新资料 → 人工审核 | 按「技能六要素」写成可审计的离线任务说明；加「无数据 / 数据过期策略」与幂等重试 | Grok Bot 技能与例程文档 | P2 |
| O-12 | 私人定制子代理 | 不再有独立章，也没有五步交接 | 若以后要转人工，只传本次任务载荷里已授权的字段，不把整段上下文原样转出 | OpenAI Agents SDK Handoffs | P2 |

---

## 11. 结合 `consolidated-spec-v2.md`：可以并进来的东西

| # | 并入内容 | 来源 | 建议放在 | 优先级 | 备注 |
|---|---|---|---|---|---|
| A-1 | 「ID 约束生成 + 服务层回填」规则 | 智旅云图 | 第 5 节新增第 5 条；3.1 硬信息校验节点 | P0 | 与「硬事实只许来自工具」一致，是它的具体实现 |
| A-2 | 「生成 → 校验 → 重试 → 择优」LangGraph 子流程 | langgraph-trip-planner | 3.1；4.3 | P0 | 同为 LangGraph，可直接参考节点划分 |
| A-3 | 插件规范新增字段 + 子图工具白名单 | MCP 规范、xAI `allowed_tools` | 4.3；4.2「规则」 | P0 | 见 O-1、O-2 |
| A-4 | 高德 MCP「生成专属地图」 | 高德 MCP Server 官方文档 | 4.5 视觉层「行中」；3.1 输出 | P1 | 行程一键导入高德 App；只用「生成专属地图 / 导航」唤端，不用「订票」相关唤端（见第 12 节） |
| A-5 | 约束求解排程 | TravelPal | 3.1 行程生成；4.2 新增「排程」内部函数 | P1 | 鼓浪屿轮渡班次、营业时间作为硬约束 |
| A-6 | 模型适配器 + 备选模型清单（DeepSeek 作文本规划备选） | DeepSeek API、OpenAI Agents SDK | 4.1；第 11 节合规 | P1 | 主模型 Qwen 不变；任何切换仍要重过合规 |
| A-7 | 会话压缩与结构化笔记 | Claude 上下文工程、Codex | 4.4 | P1 | Redis TTL 30 分钟保持不变 |
| A-8 | 并行护栏 | OpenAI Guardrails | 第 5 节；4.3 | P1 | 不进关键路径，不影响首包 < 800 毫秒 |
| A-9 | 约束通过率评测 | TravelPlanner 基准 | 8.1 新增一类；14.2 新增指标（目标值待决策） | P1 | |
| A-10 | Skills 式渐进加载 | Claude Agent Skills、DeepSeek Harness skill 插件、Grok Bot 技能 | 4.2 / 4.3（插件数量超过约 10 个后再做） | P2 | 降低提示长度与成本 |
| A-11 | 只追加事件日志 + 工具执行三段拦截 | DeepSeek Harness 架构文档 | 4.3；4.6；7.4 版本控制 | P2 | 让回放、审计、「5 分钟追溯」（13.2）更容易 |
| A-12 | 对外开放 MCP 服务器 + A2A AgentCard | MCP 规范、A2A v1.0、phoenix-zhou/multi-agent、TravelPal | 第 12 节库存即服务；第 16 节眼镜入口 | P2 | 只开放只读查询（库存、档案、讲解）；写操作不开放 |
| A-13 | 例程治理清单（审批边界、无数据策略、幂等） | Grok Bot 文档 | 7.1、7.2、7.3 | P2 | |
| A-14 | 用户偏好记忆（权重 + 遗忘 + Top-K） | TripStar | 4.4 规划维度「用户历史偏好」 | P2 | 需征得同意（第 11 节） |

**不建议并入的**：
- 多智能体并行研究架构（Claude 研究系统、TripStar 多 Agent）用在实时主链路：token 成本约为普通对话的 15 倍，和 MVP 月成本 < 500 元、P95 < 2 秒冲突。可留给离线写作 / 知识运营。
- 内部子图之间改用 A2A 通信：增加网络跳数和时延，没有收益。
- 抓取小红书 / 携程 / 美团内容：违反 4.2.2 的数据原则。
- 直接以 DeepSeek Harness 作为生产底座：开发者预览，接口会破坏性变更。

---

## 12. 矛盾与待决策

1. **「不做订票」的范围需要明确定义（待决策）**
   - v2 1.4 已写「不做全球一句话订机票酒店」「不提供机票 / 动车工具」，与「不做订票」一致。
   - 但下面几处属于「预订 / 交易」：
     - `hotel-frontdesk-note.md` 第 5 节：直连 PMS 的酒店走**实时锁房**（智能体下单成功即锁定房间）；
     - v2 4.2：**微信 MCP + 支付宝 MCP** 支付插件；
     - v2 第 12 节收入模型含「**预订佣金**」；
     - 高德 MCP 的唤端能力里有「订票」体验。
   - 酒店房间不是「票」，但「实时锁房」本质是智能体代用户下单。建议明确为：「**不做交通票、门票等票务预订；酒店按 `hotel-frontdesk-note.md` 两套流程处理，涉及付款 / 下单 / 改单的动作一律人工确认或由用户在支付通道内自行确认**」。由用户决定。
2. **v2 正文仍与酒店两套流程决策冲突**：1.3「不做实时扣库存」、1.4「不对接酒店 PMS」、4.2「不对接 PMS」尚未修改（`hotel-frontdesk-note.md` 第 5 节已记录，待用户授权后并入）。
3. **「兼容多模型」与「切换要重过合规」的张力**：4.1、第 11 节写明切换备选模型要重新过合规。多模型兼容在技术上可做（A-6），但每增加一个面向公众的模型，都可能需要单独的备案 / 登记；境外模型（OpenAI、Claude、Grok）在境内面向公众使用的可行性：待核实。建议「技术上可插拔，上线清单只放已过合规的模型」。
4. **v2 第 2 节「dsh」的指代**：v2 写「LangGraph 是骨架，dsh『一切皆插件』是血肉」。本次调研确认 `dsh` 是 DeepSeek Harness 的命令名，其口号就是「Everything is a Plugin」。v2 原文是否就是指它：推断是，待用户确认。若是，需要注意它目前是开发者预览版，建议规格里写成「借鉴其插件化思想」而不是「依赖」。
5. **实时语音主模型与 Harness 的配合**：四家 Harness 大多以文本循环为主；我们主模型是实时语音 + 函数调用一体（Qwen3.8 Omni Flash Realtime）。实时会话中工具调用的超时与降级（3 秒 + 重试 1 次）如何与「校验重试」（O-3）共存、不突破首包 < 800 毫秒：待测。建议校验重试只放在规划子图「出最终行程」这一步，语音先回一句过渡语。

---

## 13. 待核实清单

- DeepSeek Harness 的 GitHub 星标 281875 为 API 原值，数字异常大，未独立核实其真实性。
- DeepSeek Harness `llm-pi-ai` 等多厂商适配插件实际支持哪些厂商：待核实。
- 四家 Harness 是否原生支持 A2A：官方文档中未见说明，待核实。
- 智旅云图 README 自报的 RAG 指标（Top1 93.3%、MRR 0.967 等）：未复现。
- TripStar 的 GPL-2.0 许可证对我们参考或复用代码的影响：待法务确认。
- Rokid / 雷鸟 / 千问眼镜开放平台是否支持 MCP / A2A 接入：待核实。
- OpenAI、Claude、Grok 模型在境内面向公众服务的合规可行性：待核实。
- OTel GenAI 语义约定的具体版本与字段：待核实。
- Grok Bot 实际使用的模型：官方未公开。

---

## 14. 参考来源

**开源项目**
- 智旅云图：https://github.com/tutu-zzz/zhilv-yuntu
- 智旅云图学习指南（掘金）：https://juejin.cn/post/7640643117474381865
- TripStar：https://github.com/1sdv/TripStar
- hello-agents：https://github.com/datawhalechina/hello-agents
- hello-agents 第十三章示例代码：https://github.com/datawhalechina/hello-agents/tree/main/code/chapter13/helloagents-trip-planner
- langgraph-trip-planner：https://github.com/Kyle-liu888/langgraph-trip-planner
- TravelPal：https://github.com/xiaojiune/TravelPal
- Bincode1/TravelPlanner：https://github.com/Bincode1/TravelPlanner
- phoenix-zhou/multi-agent（A2A + MCP）：https://github.com/phoenix-zhou/multi-agent
- Azure AI Travel Agents：https://github.com/Azure-Samples/azure-ai-travel-agents
- TravelPlanner 基准：https://github.com/OSU-NLP-Group/TravelPlanner
- TravelPlanner 论文：https://arxiv.org/abs/2402.01622
- TravelPlanner 项目页：https://osu-nlp-group.github.io/TravelPlanner/
- TripStar-Java：https://github.com/LeeFly-cn/TripStar-Java
- mcp-google-map：https://github.com/cablate/mcp-google-map

**DeepSeek**
- DeepSeek Harness 仓库：https://github.com/deepseek-ai/deepseek-harness
- DeepSeek Harness 架构文档：https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/architecture.md
- DeepSeek Harness 文档站：https://deepseek-harness.github.io/deepseek-harness/
- 思考模式：https://api-docs.deepseek.com/guides/thinking_mode.md
- 工具调用：https://api-docs.deepseek.com/guides/tool_calls.md
- Anthropic 格式兼容：https://api-docs.deepseek.com/guides/anthropic_api
- 更新日志：https://api-docs.deepseek.com/updates/
- 应用级 API 发布（2026-11-13）：https://api-docs.deepseek.com/news/news261113/
- DeepSeek-V3.2 发布：https://www.deepseek.com/en/news/deepseek-v3-2/

**OpenAI**
- Agents SDK 文档：https://openai.github.io/openai-agents-python/
- Handoffs：https://openai.github.io/openai-agents-python/handoffs/
- Guardrails：https://openai.github.io/openai-agents-python/guardrails/
- Tracing：https://openai.github.io/openai-agents-python/tracing/
- MCP：https://openai.github.io/openai-agents-python/mcp/
- Agents SDK 仓库：https://github.com/openai/openai-agents-python
- 远程 MCP（Responses API）：https://platform.openai.com/docs/guides/tools-remote-mcp
- Unrolling the Codex agent loop：https://openai.com/index/unrolling-the-codex-agent-loop/
- Harness engineering：https://openai.com/index/harness-engineering/
- Introducing AgentKit（含下线公告）：https://openai.com/index/introducing-agentkit/
- Codex 仓库：https://github.com/openai/codex

**Anthropic**
- Building effective agents：https://www.anthropic.com/engineering/building-effective-agents
- Effective context engineering for AI agents：https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
- Building agents with the Claude Agent SDK：https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk
- Effective harnesses for long-running agents：https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents
- Equipping agents for the real world with Agent Skills：https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills
- Code execution with MCP：https://www.anthropic.com/engineering/code-execution-with-mcp
- How we built our multi-agent research system：https://www.anthropic.com/engineering/multi-agent-research-system
- Claude Agent SDK 文档：https://docs.claude.com/en/docs/agent-sdk/overview
- Claude Code 子智能体：https://code.claude.com/docs/en/sub-agents
- Claude Agent SDK（Python）仓库：https://github.com/anthropics/claude-agent-sdk-python

**xAI**
- Tools Overview：https://docs.x.ai/developers/tools/overview
- Advanced Usage（服务端 / 客户端工具混用）：https://docs.x.ai/developers/tools/advanced-usage
- Remote MCP Tools：https://docs.x.ai/developers/tools/remote-mcp
- Web Search：https://docs.x.ai/developers/tools/web-search
- Function Calling：https://docs.x.ai/developers/tools/function-calling
- Streaming & Synchronous Requests：https://docs.x.ai/developers/tools/streaming-and-sync
- Release Notes：https://docs.x.ai/developers/release-notes
- Grok Bot 概述：https://docs.x.ai/grok-bot/overview
- Grok Bot 电脑与应用：https://docs.x.ai/grok-bot/computer-and-apps
- Grok Bot 团队与企业：https://docs.x.ai/grok-bot/teams-and-enterprises
- Grok Bot 技能与例程：https://docs.x.ai/grok-bot/skills-routines-and-automations
- Grok Bot 审批、安全与隐私：https://docs.x.ai/grok-bot/approvals-security-and-privacy
- Introducing Grok Bot：https://x.ai/news/introducing-grok-bot
- Introducing Grok Bot 1.0：https://x.ai/news/grok-bot-1-0

**协议与地图**
- A2A 官网：https://a2a-protocol.org/latest/
- A2A 仓库：https://github.com/a2aproject/A2A
- Google A2A 发布博客：https://developers.googleblog.com/en/a2a-a-new-era-of-agent-interoperability/
- MCP 规范（2026-07-28）：https://modelcontextprotocol.io/specification/2026-07-28
- MCP 规范仓库：https://github.com/modelcontextprotocol/modelcontextprotocol
- 高德 MCP Server：https://lbs.amap.com/api/mcp-server/summary

---

*厦门版 Odessia · 智能体架构调研报告 · 2026-11-16 · 只新增本文件，未修改仓库中任何已有文件*
