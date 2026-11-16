# V3 修订待办

更新时间：2026-11-16 16:00 台北时间（V3-1 记于 15:46，V3-2 记于 15:47，V3-3 记于 15:52，V3-4 记于 15:55，V3-5 记于 15:56、更新于 15:57、15:58、15:59、16:00，V3-6 记于 15:58）

> 只记录待办，不修改 `consolidated-spec-v2.md`、`odessia-spec.html` 或其他已有文件。待办在生成 V3 时统一处理。

## V3-1：10.3 成本与时延从主线相关表述中拿掉，明确归入子线（写作智能体）

- **类型**：待办（新增）
- **提出**：用户通话，按 mao789 的口径（2026-11-16 15:46 台北时间）
- **现状（v2 @ `bd24673`，1.5 节）**：
  - 「主线」一条下面列有两条子项，其中第二条原文为：「10.3 成本与时延出自写作智能体，改标为子线（见 D-11）。」
  - 这句话虽然写的是「改标为子线」，但它仍挂在「主线」一条之下，和「主线运行依赖的公共设施」那条子项并列，读起来像仍属主线相关表述。
  - 同一条「主线」的修订注里还保留原文「并把 10.3 成本与时延标为主线」（这是对旧定义的引用）。
- **V3 要做的**：
  - 把「10.3 成本与时延」这条从「主线」下面移走，放到「子线」下的「写作智能体（3.3，全文见 `writing-agent-spec.md`）」一条里，明确归入写作智能体这条子线。
  - 写明口径：成本与时延属于行程规划结束后的离线环节（mao789）。
  - 检查 10.3 标题标签（现为【子线 · 写作智能体】）、目录、D-6、D-11 及网页版 `odessia-spec.html` 对应处是否一致。
- **状态**：待处理

## V3-2：主线、子线章节各配一张架构图

- **类型**：待办（新增）
- **提出**：用户通话（2026-11-16 15:47 台北时间）
- **V3 要做的**：在 1.5 / 1.6（或第 2 节）主线、子线说明处各放一张架构图，草稿如下，V3 落笔时可直接使用。
- **口径说明**：
  - 按 v2 @ `bd24673` 的现行定义，主线只指规划子图本身；路由器、工作节点、主循环、编排 / 记忆 / 视觉 / 监控层、防幻觉机制是主线运行依赖的**公共设施**（1.5）。图里把它们画在主线周围，标为公共设施。
  - 讲解子图在 v2 里是**分支线**，不是子线；子线图里单独标为分支线。
  - 子线与主线环节的对应关系是本待办的整理（推断），V3 落笔时再核对。
- **状态**：待处理

### 图 A：主线架构图（草稿）

```mermaid
flowchart TB
    U["用户输入<br/>小程序 / 网页 / 眼镜"] --> R["路由器<br/>（公共设施，第 2 节）"]
    R --> W["工作节点（窗口）<br/>（公共设施，第 2 节）"]
    W --> L["主循环<br/>（公共设施，第 2 节）"]
    L --> P

    subgraph P["主线：规划子图（3.1）= 行程规划本身"]
        direction LR
        P1["① 收集需求与偏好"] --> P2["② 检索"] --> P3["③ 生成 2–3 套行程"] --> P4["④ 硬信息校验"] --> P5["⑤ 输出"]
        P4 -. 校验不过 .-> P3
    end

    subgraph INFRA["公共设施（主线运行依赖，不单独标注）"]
        direction TB
        O["编排层 4.3<br/>LangGraph；3 秒超时、重试 1 次、降级打标"]
        M["记忆层 4.4<br/>Redis（TTL 30 分钟）+ 五个语义库<br/>HNSW + BM25 + RRF"]
        V["视觉层 4.5<br/>结构化卡片 / 路线提示 / 轻量锚点"]
        MON["监控层 4.6<br/>OTel 旁路，只记录不阻塞"]
        AH["防幻觉机制 第 5 节<br/>双源校验 + 来源标签"]
    end

    subgraph TOP["顶层（主线与子线共用底座）"]
        direction LR
        MOD["4.1 模型层<br/>Qwen3.8 Omni Flash Realtime / MOSS / Nex N2.6"]
        TOOL["4.2 工具层<br/>插件化工具、高德 MCP 等"]
    end

    O -. 调度 .-> R
    O -. 调度 .-> W
    M -. 会话状态 .-> L
    M -. 语义检索 .-> P2
    AH -. 校验规则 .-> P4
    P5 -. 渲染 .-> V
    MON -. 追踪 .-> L
    MOD -. 推理 / 函数调用 .-> L
    TOOL -. 工具调用 .-> P2
```

### 图 B：子线架构图（草稿）

```mermaid
flowchart LR
    subgraph MAIN["主线：规划子图（3.1）"]
        direction TB
        S1["① 收集需求与偏好"]
        S2["② 检索"]
        S3["③ 生成 2–3 套行程"]
        S4["④ 硬信息校验"]
        S5["⑤ 输出"]
        S1 --> S2 --> S3 --> S4 --> S5
    end

    BR["讲解子图 3.2【分支线】<br/>旁路触发，看到什么讲什么"]

    subgraph DEST["目的地服务类子线（可插拔，1.6 / 4.2.3）"]
        REG["注册<br/>用户注册 / 登录（细节待定义）"]
        HOTEL["酒店（双模式）<br/>PMS直连：下单即锁房，前台看板<br/>意向单：意向单 + 人工确认"]
        PAY["支付<br/>微信 / 支付宝 MCP，目的地本地支付"]
        TICKET["门票<br/>景区门票，直接入园"]
        NAV["自驾导航<br/>高德 MCP 导航"]
        PC["私人定制<br/>人工顾问交接（第 16 节）<br/>企业微信群仅用于此"]
    end

    subgraph OFF["离线类子线（不进主链路）"]
        WR["写作智能体（3.3）<br/>含 10.3 成本与时延"]
        FB["反馈收集智能体（7.2）"]
        KO["知识运营离线管道（7.1）<br/>人工审核后入库"]
    end

    GL["智能眼镜对接（第 17 节）<br/>语音 + 摄像头入口"]

    REG -- "进入前：身份与偏好" --> S1
    PC -- "触发 T1–T3 时：人工接手需求" --> S1
    S3 -- "行程中的住宿：查询 / 下单" --> HOTEL
    S3 -- "行程中的景点：购票" --> TICKET
    HOTEL -- "付款环节" --> PAY
    TICKET -- "付款环节（推断）" --> PAY
    S5 -- "行中：路线一键导航" --> NAV
    S5 -- "行中：到点讲解" --> BR
    GL -- "输入：语音 / 画面" --> S1
    GL -- "画面识别触发" --> BR
    S5 -- "行程结束后：游记 / 内容" --> WR
    S5 -- "行程结束后：收集反馈" --> FB
    FB -- "变更草稿" --> KO
    KO -- "审核后更新语义库" --> S2
```

### 子线与主线环节对应表（草稿）

| 子线 | 对应主线环节 | 作用 |
|---|---|---|
| 注册 | ① 之前 | 用户身份、历史偏好进入需求收集 |
| 私人定制 | ①（交接触发时） | 复杂或特殊需求转人工顾问，结果回写订单系统，规划子图读最新确认版 |
| 酒店（双模式） | ③ 生成 → 下单 | 住宿推荐后下单；按酒店配置走实时锁房或意向单 + 人工确认 |
| 门票 | ③ 生成 → 下单 | 行程中景区的门票，直接入园 |
| 支付 | 酒店 / 门票下单后 | 微信 / 支付宝 MCP，目的地本地支付，用户本人确认 |
| 自驾导航 | ⑤ 输出后（行中） | 把行程路线交给高德 MCP 导航 |
| 讲解子图【分支线】 | ⑤ 输出后（行中） | 旁路触发的景点讲解 |
| 智能眼镜对接 | ① 输入 / 讲解触发 | 第三个入口：语音输入、摄像头画面触发讲解 |
| 写作智能体 | ⑤ 输出后（行程结束） | 离线生成内容；10.3 成本与时延属于这一环（见 V3-1） |
| 反馈收集 | ⑤ 输出后（行程结束） | 收集纠错与评价，生成变更草稿 |
| 知识运营离线管道 | 反哺 ② 检索 | 人工审核后更新语义库 |

## V3-3：五个语义库各配一张参考图

- **类型**：待办（新增）
- **提出**：用户通话（2026-11-16 15:52 台北时间）
- **V3 要做的**：在 4.4 记忆层「五个独立语义库」处，给每个库配一张参考图。图的画法参照[智旅云图 README 的「系统数据流」图](https://github.com/tutu-zzz/zhilv-yuntu/blob/main/README.md)：按模块分块，实线表示主流程，虚线表示返回路径，颜色区分层级。草稿如下，V3 落笔时可直接使用。
- **每张图统一四段**：
  1. **数据来源**：线上爬取、线下采集、人工录入。
  2. **入库**：清洗与结构化，人工审核，切片，然后同时做向量化和 BM25 索引。
  3. **在线检索**：查询改写，查缓存，向量召回与 BM25 召回并行，结构化过滤，RRF 融合，重排，最后把结果交给大模型。
  4. **服务的主线环节**：规划子图或讲解子图。
- **和 v2 的对照**（v2 @ `bd24673` 原有口径）：
  - 4.4 写的是：五个库共享 HNSW 向量基础设施，混合检索用 HNSW 向量 + BM25 + 结构化过滤，用 RRF 融合；规划子图以结构化 + BM25 为主，讲解子图走纯语义。图里在线段画的是通用链路，V3 落笔时要在 4.4 写明两个子图的侧重不同。
  - 「查询改写」和「重排」两步 v2 没写，是参照智旅云图加的，V3 要决定是否正式写入。
  - 图 1 里的「3D 建筑模型」是这次通话的要求。v2 写的是「不做全岛 3D 预建模，只对约 50 栋做轻量空间锚点」（4.5、第 17 节）。图里暂按 50 栋轻量锚点画，3D 模型做到什么范围，V3 前需要确认。
  - 图 2 的「自建库优先，外部联网兜底」出自 v2 的餐厅查询双库。
  - 入库挂在知识运营离线管道（7.1）下，图里的「人工审核」也出自 7.1。
  - 各库的具体线上来源（哪些网站、哪些名录）v2 没写，图中列的是示意，标有「推断」的连线同样是推断。
- **状态**：待处理

### 图 1：931 栋历史建筑档案库（多模态）

```mermaid
flowchart TD
    subgraph Sources["数据来源"]
        aS1["线上爬取<br/>政府公开名录 / 文保资料 / 公开文献"]
        aS2["线下采集<br/>实地拍照、测绘、口述访谈"]
        aS3["人工录入<br/>档案字段、年代、风格、产权、开放状态"]
        aS4["3D 建筑模型<br/>约 50 栋做轻量空间锚点"]
    end
    aS1 & aS2 & aS3 --> aCL
    aS4 --> aCL
    subgraph Ingest["入库（离线，知识运营管道 7.1）"]
        aCL["清洗 / 去重 / 字段结构化"]
        aHR{"人工审核"}
        aCH["切片"]
        aEMB["向量化"]
        aIDX["BM25 索引"]
        aCL --> aHR -->|通过| aCH
        aCH --> aEMB
        aCH --> aIDX
    end
    aCH --> aIMG["图片向量化（CLIP）"]
    subgraph Store["931 栋历史建筑档案库（共享 HNSW 基础设施上的独立库，4.4）"]
        aVDB[("向量索引 HNSW")]
        aBDB[("BM25 倒排索引")]
        aMETA[("结构化字段")]
    end
    aIMG --> aVDB
    aEMB --> aVDB
    aIDX --> aBDB
    aCL --> aMETA
    aVDB --> aV
    aBDB --> aB
    aMETA --> aF
    subgraph Online["在线检索"]
        aQ("用户问题 / 眼镜画面 + GPS")
        aQR{"查询改写"}
        aC{"Redis 缓存命中?<br/>TTL 30 分钟"}
        aV["HNSW 向量召回"]
        aB["BM25 关键词召回"]
        aF["结构化过滤<br/>位置、年代、风格"]
        aRRF["RRF 融合"]
        aRR{"重排"}
        aSC["写回缓存"]
        aOUT("top-k 片段 + 来源标签<br/>交给大模型")
        aQ --> aQR --> aC
        aC -->|命中| aOUT
        aC -->|未命中| aV & aB
        aV --> aF
        aB --> aF
        aF --> aRRF --> aRR --> aSC --> aOUT
    end
    subgraph Serve["服务的主线环节"]
        aT1["讲解子图 3.2（分支线）<br/>主要服务方，纯语义"]
        aT2["规划子图 ② 检索<br/>建筑类景点候选（推断）"]
        aT3["视觉层 4.5<br/>GPS 粗定位 → CLIP 精定位"]
    end
    aOUT --> aT1
    aOUT -.-> aT2
    aT3 -.->|画面 / 位置| aQ
    aOUT -.->|结果返回| aT1
    classDef src fill:#eef2ff,stroke:#818cf8,color:#111;
    classDef ingest fill:#fefce8,stroke:#facc15,color:#111;
    classDef store fill:#fff7ed,stroke:#fb923c,color:#111;
    classDef online fill:#f0fdfa,stroke:#2dd4bf,color:#111;
    classDef rerank fill:#fff1f2,stroke:#fb7185,color:#111;
    classDef serve fill:#f0fdf4,stroke:#4ade80,color:#111;
    class aS1,aS2,aS3,aS4 src;
    class aCL,aHR,aCH,aEMB,aIDX ingest;
    class aVDB,aBDB,aMETA store;
    class aQ,aQR,aC,aV,aB,aF,aRRF,aSC online;
    class aRR rerank;
    class aOUT,aT1,aT2,aT3 serve;
    class aIMG ingest;
```

### 图 2：小吃探店 / 烟火气库

```mermaid
flowchart TD
    subgraph Sources["数据来源"]
        bS1["线上爬取<br/>公开点评、攻略帖、社交平台"]
        bS2["线下采集（核心）<br/>实地踩点：美团、大众点评上没有的线下好店"]
        bS3["人工录入<br/>店名、位置、营业时间、招牌、人均、踩点日期"]
    end
    bS1 & bS2 & bS3 --> bCL
    subgraph Ingest["入库（离线，知识运营管道 7.1）"]
        bCL["清洗 / 去重 / 字段结构化"]
        bHR{"人工审核"}
        bCH["切片"]
        bEMB["向量化"]
        bIDX["BM25 索引"]
        bCL --> bHR -->|通过| bCH
        bCH --> bEMB
        bCH --> bIDX
    end
    subgraph Store["小吃探店 / 烟火气库（共享 HNSW 基础设施上的独立库，4.4）"]
        bVDB[("向量索引 HNSW")]
        bBDB[("BM25 倒排索引")]
        bMETA[("结构化字段")]
    end
    bEMB --> bVDB
    bIDX --> bBDB
    bCL --> bMETA
    bVDB --> bV
    bBDB --> bB
    bMETA --> bF
    subgraph Online["在线检索"]
        bQ("用户需求<br/>位置、口味、预算、时段")
        bQR{"查询改写"}
        bC{"Redis 缓存命中?<br/>TTL 30 分钟"}
        bV["HNSW 向量召回"]
        bB["BM25 关键词召回"]
        bF["结构化过滤<br/>营业中、距离、人均"]
        bRRF["RRF 融合"]
        bRR{"重排"}
        bSC["写回缓存"]
        bOUT("top-k 片段 + 来源标签<br/>交给大模型")
        bQ --> bQR --> bC
        bC -->|命中| bOUT
        bC -->|未命中| bV & bB
        bV --> bF
        bB --> bF
        bF --> bRRF --> bRR --> bSC --> bOUT
    end
    subgraph Serve["服务的主线环节"]
        bT1["规划子图 ② 检索<br/>餐厅查询：自建库优先"]
        bT2["外部联网兜底<br/>自建库查不到时"]
        bT3["规划子图 ④ 硬信息校验<br/>营业时间、地址双源校验"]
    end
    bOUT --> bT1
    bT1 -.->|无结果| bT2
    bT1 --> bT3
    bT3 -.->|结果返回| bQ
    classDef src fill:#eef2ff,stroke:#818cf8,color:#111;
    classDef ingest fill:#fefce8,stroke:#facc15,color:#111;
    classDef store fill:#fff7ed,stroke:#fb923c,color:#111;
    classDef online fill:#f0fdfa,stroke:#2dd4bf,color:#111;
    classDef rerank fill:#fff1f2,stroke:#fb7185,color:#111;
    classDef serve fill:#f0fdf4,stroke:#4ade80,color:#111;
    class bS1,bS2,bS3 src;
    class bCL,bHR,bCH,bEMB,bIDX ingest;
    class bVDB,bBDB,bMETA store;
    class bQ,bQR,bC,bV,bB,bF,bRRF,bSC online;
    class bRR rerank;
    class bOUT,bT1,bT2,bT3 serve;
```

### 图 3：文化典故 / 讲解库

```mermaid
flowchart TD
    subgraph Sources["数据来源"]
        cS1["线上爬取<br/>地方志、公开文献、文保资料"]
        cS2["线下采集<br/>导游口述、老居民访谈"]
        cS3["人工录入<br/>典故正文、出处、关联建筑 / 景点"]
    end
    cS1 & cS2 & cS3 --> cCL
    subgraph Ingest["入库（离线，知识运营管道 7.1）"]
        cCL["清洗 / 去重 / 字段结构化"]
        cHR{"人工审核"}
        cCH["切片"]
        cEMB["向量化"]
        cIDX["BM25 索引"]
        cCL --> cHR -->|通过| cCH
        cCH --> cEMB
        cCH --> cIDX
    end
    subgraph Store["文化典故 / 讲解库（共享 HNSW 基础设施上的独立库，4.4）"]
        cVDB[("向量索引 HNSW")]
        cBDB[("BM25 倒排索引")]
        cMETA[("结构化字段")]
    end
    cEMB --> cVDB
    cIDX --> cBDB
    cCL --> cMETA
    cVDB --> cV
    cBDB --> cB
    cMETA --> cF
    subgraph Online["在线检索"]
        cQ("用户提问 / 讲解触发点<br/>（当前位置或识别出的建筑）")
        cQR{"查询改写"}
        cC{"Redis 缓存命中?<br/>TTL 30 分钟"}
        cV["HNSW 向量召回"]
        cB["BM25 关键词召回"]
        cF["结构化过滤<br/>关联建筑 / 景点 ID"]
        cRRF["RRF 融合"]
        cRR{"重排"}
        cSC["写回缓存"]
        cOUT("top-k 片段 + 来源标签<br/>交给大模型")
        cQ --> cQR --> cC
        cC -->|命中| cOUT
        cC -->|未命中| cV & cB
        cV --> cF
        cB --> cF
        cF --> cRRF --> cRR --> cSC --> cOUT
    end
    subgraph Serve["服务的主线环节"]
        cT1["讲解子图 3.2（分支线）<br/>纯语义，看到什么讲什么"]
        cT2["防幻觉 第 5 节<br/>标出处，无出处不讲"]
    end
    cOUT --> cT2 --> cT1
    cT1 -.->|结果返回| cQ
    classDef src fill:#eef2ff,stroke:#818cf8,color:#111;
    classDef ingest fill:#fefce8,stroke:#facc15,color:#111;
    classDef store fill:#fff7ed,stroke:#fb923c,color:#111;
    classDef online fill:#f0fdfa,stroke:#2dd4bf,color:#111;
    classDef rerank fill:#fff1f2,stroke:#fb7185,color:#111;
    classDef serve fill:#f0fdf4,stroke:#4ade80,color:#111;
    class cS1,cS2,cS3 src;
    class cCL,cHR,cCH,cEMB,cIDX ingest;
    class cVDB,cBDB,cMETA store;
    class cQ,cQR,cC,cV,cB,cF,cRRF,cSC online;
    class cRR rerank;
    class cOUT,cT1,cT2 serve;
```

### 图 4：打卡点库

```mermaid
flowchart TD
    subgraph Sources["数据来源"]
        dS1["线上爬取<br/>社交平台热门打卡点、攻略帖"]
        dS2["线下采集<br/>实地拍机位、最佳时段、人流"]
        dS3["人工录入<br/>坐标、开放时间、拍摄建议"]
    end
    dS1 & dS2 & dS3 --> dCL
    subgraph Ingest["入库（离线，知识运营管道 7.1）"]
        dCL["清洗 / 去重 / 字段结构化"]
        dHR{"人工审核"}
        dCH["切片"]
        dEMB["向量化"]
        dIDX["BM25 索引"]
        dCL --> dHR -->|通过| dCH
        dCH --> dEMB
        dCH --> dIDX
    end
    subgraph Store["打卡点库（共享 HNSW 基础设施上的独立库，4.4）"]
        dVDB[("向量索引 HNSW")]
        dBDB[("BM25 倒排索引")]
        dMETA[("结构化字段")]
    end
    dEMB --> dVDB
    dIDX --> dBDB
    dCL --> dMETA
    dVDB --> dV
    dBDB --> dB
    dMETA --> dF
    subgraph Online["在线检索"]
        dQ("用户偏好 / 当前位置")
        dQR{"查询改写"}
        dC{"Redis 缓存命中?<br/>TTL 30 分钟"}
        dV["HNSW 向量召回"]
        dB["BM25 关键词召回"]
        dF["结构化过滤<br/>坐标范围、开放时间"]
        dRRF["RRF 融合"]
        dRR{"重排"}
        dSC["写回缓存"]
        dOUT("top-k 片段 + 来源标签<br/>交给大模型")
        dQ --> dQR --> dC
        dC -->|命中| dOUT
        dC -->|未命中| dV & dB
        dV --> dF
        dB --> dF
        dF --> dRRF --> dRR --> dSC --> dOUT
    end
    subgraph Serve["服务的主线环节"]
        dT1["规划子图 ② 检索<br/>打卡点进入候选"]
        dT2["规划子图 ③ 生成行程<br/>按时段、顺路编排"]
        dT3["视觉层 4.5<br/>打卡点作为独立图层叠加在路网上"]
        dT4["讲解子图 3.2<br/>眼镜识别打卡点时触发"]
    end
    dOUT --> dT1 --> dT2
    dOUT --> dT3
    dOUT -.-> dT4
    dT2 -.->|结果返回| dQ
    classDef src fill:#eef2ff,stroke:#818cf8,color:#111;
    classDef ingest fill:#fefce8,stroke:#facc15,color:#111;
    classDef store fill:#fff7ed,stroke:#fb923c,color:#111;
    classDef online fill:#f0fdfa,stroke:#2dd4bf,color:#111;
    classDef rerank fill:#fff1f2,stroke:#fb7185,color:#111;
    classDef serve fill:#f0fdf4,stroke:#4ade80,color:#111;
    class dS1,dS2,dS3 src;
    class dCL,dHR,dCH,dEMB,dIDX ingest;
    class dVDB,dBDB,dMETA store;
    class dQ,dQR,dC,dV,dB,dF,dRRF,dSC online;
    class dRR rerank;
    class dOUT,dT1,dT2,dT3,dT4 serve;
```

### 图 5：古早味库

```mermaid
flowchart TD
    subgraph Sources["数据来源"]
        eS1["线上爬取<br/>老字号名录、公开报道"]
        eS2["线下采集<br/>实地踩点、店主口述、做法与传承"]
        eS3["人工录入<br/>店名、品类、年代、故事、营业信息"]
    end
    eS1 & eS2 & eS3 --> eCL
    subgraph Ingest["入库（离线，知识运营管道 7.1）"]
        eCL["清洗 / 去重 / 字段结构化"]
        eHR{"人工审核"}
        eCH["切片"]
        eEMB["向量化"]
        eIDX["BM25 索引"]
        eCL --> eHR -->|通过| eCH
        eCH --> eEMB
        eCH --> eIDX
    end
    subgraph Store["古早味库（共享 HNSW 基础设施上的独立库，4.4）"]
        eVDB[("向量索引 HNSW")]
        eBDB[("BM25 倒排索引")]
        eMETA[("结构化字段")]
    end
    eEMB --> eVDB
    eIDX --> eBDB
    eCL --> eMETA
    eVDB --> eV
    eBDB --> eB
    eMETA --> eF
    subgraph Online["在线检索"]
        eQ("用户需求<br/>想吃老味道 / 某道传统小吃")
        eQR{"查询改写"}
        eC{"Redis 缓存命中?<br/>TTL 30 分钟"}
        eV["HNSW 向量召回"]
        eB["BM25 关键词召回"]
        eF["结构化过滤<br/>营业中、距离、品类"]
        eRRF["RRF 融合"]
        eRR{"重排"}
        eSC["写回缓存"]
        eOUT("top-k 片段 + 来源标签<br/>交给大模型")
        eQ --> eQR --> eC
        eC -->|命中| eOUT
        eC -->|未命中| eV & eB
        eV --> eF
        eB --> eF
        eF --> eRRF --> eRR --> eSC --> eOUT
    end
    subgraph Serve["服务的主线环节"]
        eT1["规划子图 ② 检索<br/>老字号进入餐饮候选"]
        eT2["规划子图 ④ 硬信息校验<br/>营业信息双源校验"]
        eT3["讲解子图 3.2<br/>讲这道菜 / 这家店的来历（推断）"]
    end
    eOUT --> eT1 --> eT2
    eOUT -.-> eT3
    eT2 -.->|结果返回| eQ
    classDef src fill:#eef2ff,stroke:#818cf8,color:#111;
    classDef ingest fill:#fefce8,stroke:#facc15,color:#111;
    classDef store fill:#fff7ed,stroke:#fb923c,color:#111;
    classDef online fill:#f0fdfa,stroke:#2dd4bf,color:#111;
    classDef rerank fill:#fff1f2,stroke:#fb7185,color:#111;
    classDef serve fill:#f0fdf4,stroke:#4ade80,color:#111;
    class eS1,eS2,eS3 src;
    class eCL,eHR,eCH,eEMB,eIDX ingest;
    class eVDB,eBDB,eMETA store;
    class eQ,eQR,eC,eV,eB,eF,eRRF,eSC online;
    class eRR rerank;
    class eOUT,eT1,eT2,eT3 serve;
```

## V3-4：整体系统流程图

- **类型**：待办（新增）
- **提出**：用户通话（2026-11-16 15:55 台北时间）
- **V3 要做的**：在第 1 节总览或第 2 节架构处放一张整体系统流程图，和 V3-2 的两张分图、V3-3 的五张库图配套。画法参照[智旅云图 README 的「系统数据流」图](https://github.com/tutu-zzz/zhilv-yuntu/blob/main/README.md)：按模块分块，用颜色区分层级。草稿如下，V3 落笔时可直接使用。
- **图例**：

| 类别 | 线型 | 节点颜色 |
|---|---|---|
| 主线（规划子图 3.1） | 蓝色粗实线 | 蓝底 |
| 运行骨架（路由器、工作节点、主循环；公共设施） | 蓝色粗实线（主线经过） | 黄底 |
| 公共设施（4.3–4.6、第 5 节） | 灰色细虚线 | 灰底虚线框 |
| 子线 | 橙色实线 | 橙底 |
| 分支线（讲解子图 3.2） | 紫色长虚线 | 紫底虚线框 |

- **节点上标的交互方式**（〔〕内）：
  - **MCP 调用**：高德 MCP（硬信息校验、自驾导航），微信 / 支付宝 MCP（支付）。出自 v2 4.2、4.2.3。
  - **插件插拔**：统一检索插件、插件工具箱、酒店 / 门票 / 注册等子线插件。出自 v2 第 2 节「一切皆插件」和 4.2.3。
  - **人工确认卡点**：私人定制顾问接手、酒店意向单、支付由用户本人确认、写作发布前确认、知识入库两级审核。出自 v2 1.3、4.2.1、4.6、7.1–7.3、第 16 节。
  - **A2A 委托**（智能体之间委托任务）：路由器分派、讲解子图、写作智能体、反馈收集智能体。**v2 里没有出现「A2A」这个词**，这几处是按「智能体之间委托」标的推断，V3 要决定是否采用这个说法。
- **和 v2 的对照，以及需要确认的点**：
  - 门票接支付这条连线是推断：v2 4.2.3 写的是「支付作为酒店子线的支付环节」，没写门票走支付。
  - 酒店 `PMS直连` 实时锁房和 v2 3.1.1「AI 不付款、不下单、不改单」的关系，仍是之前留下的待确认项，图里把酒店同时标成插件插拔和人工确认卡点。
  - 讲解子图有两个进入点：主循环按意图分叉（第 2 节），以及行中旁路触发（3.2）。图里两个都画了。
  - 第 7 节说离线链路「只通过讲解库这一个交汇点连接」。图里知识运营更新的是记忆层整体，V3 落笔时如果要精确，应改成只连讲解库（另见 V3-3）。
  - 支付上那条监控虚线出自 4.6「敏感路径（支付、下单、前端展示）人工确认」。
- **状态**：待处理

### 整体系统流程图（草稿）

```mermaid
flowchart TD
    %% ------- 入口 -------
    subgraph Entry["用户入口"]
        MP["小程序"]
        WEB["网页"]
        GL["智能眼镜<br/>第 17 节【子线】<br/>〔插件插拔：眼镜 SDK 接入〕"]
    end

    %% ------- 主线运行骨架（公共设施） -------
    subgraph Core["运行骨架（第 2 节，公共设施）"]
        R["路由器<br/>〔A2A 委托：按意图分派〕"]
        W["工作节点（窗口）<br/>每窗口一个 LangGraph 实例"]
        L["主循环<br/>模型节点 → 工具执行 → 结果回填<br/>〔插件插拔：插件工具箱〕"]
    end

    %% ------- 主线 -------
    subgraph Main["主线：规划子图 3.1（行程规划本身）"]
        P1["① 收集需求与偏好"]
        P2["② 检索<br/>〔插件插拔：统一检索插件〕"]
        P3["③ 生成 2–3 套行程"]
        P4{"④ 硬信息校验<br/>〔MCP 调用：高德双源校验〕"}
        P5["⑤ 输出"]
    end

    %% ------- 公共设施 -------
    subgraph Infra["公共设施（虚线，不单独标主线 / 子线）"]
        ORC["编排层 4.3<br/>LangGraph；3 秒超时、重试 1 次、降级"]
        MEM[("记忆层 4.4<br/>Redis 会话（TTL 30 分钟）<br/>五个语义库：931 栋建筑 / 小吃烟火气 /<br/>文化典故 / 打卡点 / 古早味")]
        VIS["视觉层 4.5<br/>结构化卡片、路网、打卡点图层"]
        MON["监控层 4.6<br/>OTel 旁路，只记录不阻塞"]
        AH["防幻觉机制 第 5 节<br/>双源校验 + 来源标签"]
    end

    %% ------- 子线：行前 -------
    subgraph PreSub["子线：需求阶段"]
        REG["注册【子线】<br/>〔插件插拔〕细节待定义"]
        PC["私人定制【子线】<br/>第 16 节，T1–T3 触发<br/>〔人工确认卡点：顾问接手〕"]
    end

    %% ------- 子线：下单 -------
    subgraph OrderSub["子线：下单阶段（4.2.3 插件）"]
        HOTEL["酒店【子线】双模式<br/>PMS直连：实时锁房<br/>意向单：人工确认<br/>〔插件插拔 + 人工确认卡点〕"]
        TICKET["门票【子线】<br/>景区门票直接入园<br/>〔插件插拔〕"]
        PAY["支付【子线】<br/>〔MCP 调用：微信 / 支付宝 MCP〕<br/>〔人工确认卡点：用户本人确认〕"]
    end

    %% ------- 子线 / 分支线：行中 -------
    subgraph TripSub["行程中"]
        NAV["自驾导航【子线】<br/>〔MCP 调用：高德 MCP〕"]
        BR["讲解子图 3.2【分支线】<br/>旁路触发，看到什么讲什么<br/>〔A2A 委托〕"]
    end

    %% ------- 子线：行程结束后 / 离线 -------
    subgraph OffSub["行程结束后 / 离线（不进主链路）"]
        WR["写作智能体【子线】<br/>含 10.3 成本与时延<br/>〔A2A 委托 + 人工确认卡点：发布前确认〕"]
        FB["反馈收集智能体 7.2【子线】<br/>〔A2A 委托〕"]
        KO["知识运营离线管道 7.1【子线】<br/>〔人工确认卡点：两级审核后入库〕"]
    end

    %% ------- 连线 -------
    MP ==> R
    WEB ==> R
    GL ==> R
    R ==> W
    W ==> L
    L ==> P1
    P1 ==> P2
    P2 ==> P3
    P3 ==> P4
    P4 ==>|通过| P5
    P4 ==>|校验失败，回生成| P3
    ORC -.->|调度| R
    ORC -.->|调度| W
    MEM -.->|会话状态| L
    MEM -.->|五库检索| P2
    AH -.->|校验规则| P4
    P5 -.->|渲染| VIS
    MON -.->|追踪| L
    MON -.->|敏感路径| PAY
    REG -->|进入前：身份 / 偏好| P1
    PC <-->|T1–T3 触发：人工接手需求| P1
    P3 -->|下单| HOTEL
    P3 -->|下单| TICKET
    HOTEL -->|付款| PAY
    TICKET -->|付款（推断）| PAY
    P5 -->|行中：一键导航| NAV
    P5 -->|行程结束后| WR
    P5 -->|行程结束后| FB
    FB -->|变更草稿| KO
    KO -->|审核后反向更新语义库| MEM
    P5 -->|行中：到点讲解| BR
    L -->|意图分叉：讲解| BR
    GL -->|画面识别触发| BR
    MEM -->|讲解库纯语义| BR

    %% ------- 线型 / 颜色区分 -------
    linkStyle 0,1,2,3,4,5,6,7,8,9,10 stroke:#2563eb,stroke-width:3px;
    linkStyle 11,12,13,14,15,16,17,18 stroke:#9ca3af,stroke-width:1px,stroke-dasharray:4 4;
    linkStyle 19,20,21,22,23,24,25,26,27,28,29 stroke:#ea580c,stroke-width:1.5px;
    linkStyle 30,31,32,33 stroke:#9333ea,stroke-width:1.5px,stroke-dasharray:8 3;

    classDef entry fill:#eef2ff,stroke:#818cf8,color:#111;
    classDef core fill:#fefce8,stroke:#facc15,color:#111;
    classDef main fill:#dbeafe,stroke:#2563eb,stroke-width:2px,color:#111;
    classDef infra fill:#f3f4f6,stroke:#9ca3af,stroke-dasharray:4 4,color:#111;
    classDef sub fill:#fff7ed,stroke:#ea580c,color:#111;
    classDef branch fill:#faf5ff,stroke:#9333ea,stroke-dasharray:8 3,color:#111;

    class MP,WEB entry;
    class R,W,L core;
    class P1,P2,P3,P4,P5 main;
    class ORC,MEM,VIS,MON,AH infra;
    class GL,REG,PC,HOTEL,TICKET,PAY,NAV,WR,FB,KO sub;
    class BR branch;
```

## V3-5：7.2 反馈收集智能体，交叉验证按场景分级使用多模型（补充，不替换）

- **类型**：待办（新增），已决策
- **提出 / 决策**：用户通话转述，mao789 同意（2026-11-16 15:56 台北时间）
- **决策内容**：交叉验证这一步按场景分级：
  - **核心行程问答**：单模型 + 规则校验，目的是省 token 成本。
  - **争议大的内容**（典故、冷门店铺这类）：开双模型对比融合，候选模型如 DeepSeek、豆包、千问；融合后**人工终审**再入库。（15:59 口径补充：双模型答案有偏差时不自动融合，按下面「统一口径」转人工，提取核心要点后判定入库或删除。）
- **与 v2 的关系**：这是对 7.2 现有流程的**补充，不是替换**。v2 原文是「**四步流程**：结构化提取 → 交叉验证（官方 / 高德）→ 生成变更草稿 → 人工审核队列。」四步保留，「交叉验证（官方 / 高德）」这一步原样保留，多模型对比是按场景追加的一层。
- **V3 落笔时要做的**：
  - 在 7.2「四步流程」下补一条子项，写明上述分级规则；原有四步和「官方 / 高德」交叉验证不改。
  - 「人工终审」接到 7.3 现有的人工审核规则上（24 小时内处理，导游一级审事实、你二级审语气和优先级），不另起一套审核流程。
  - 同步更新第 10 节成本里与反馈智能体相关的模型调用估算。双模型只用于争议内容，调用量要单独列出。
- **需要在 V3 前定下来的**（以下 v2 和这次决策都没写）：
  - ~~「争议大」的判定标准：按库划分（例如文化典故库、小吃探店库），还是按单条内容的置信度。~~ ~~已决策（15:57）：争议大的内容直接转人工判定，不设自动判定标准。人工判定后如果需要双模型对比，流程照旧：双模型对比融合，再人工终审入库。~~ **已决策，按 15:59 完整口径更新（mao789）**：争议大的内容**一律转人工**，不设自动判定标准。人工只提取**核心要点**（不看完整原文），判断对错后决定**入库或删除**。口径见下面「统一口径」。
  - ~~双模型具体用哪两个：待定。~~ **已决策（2026-11-16 16:00 台北时间，mao789）**：选**最聪明的模型**。
    - **国内版**：DeepSeek 最新版或 Kimi。
    - **国外版**：GPT、Gemini。Grok 也可以，但目前不可用，先不列入。
    - 注：「决策内容」里 15:56 记的候选（DeepSeek、豆包、千问）以本条为准。国内版选「DeepSeek 最新版或 Kimi」，是两者都用来组成双模型，还是二选一再搭别的模型，口径未说明，V3 落笔前确认。具体版本号随落笔时的最新版定。分版本见 V3-6。
  - ~~融合规则：两个模型结论不一致时怎么处理。~~ ~~已决策（15:58）：两个模型结论不一致时，由人工直接挑选，不自动融合。~~ **已决策，按 15:59 完整口径更新（mao789）**：双模型答案**只要有偏差，一律转人工**，不自动融合。人工只提取**核心要点**（不看完整原文），判断对错后决定**入库或删除**。口径见下面「统一口径」。
  - **统一口径（2026-11-16 15:59 台北时间，mao789）**：双模型答案有偏差，或者内容争议大 → 人工介入 → 提取核心要点（不看完整原文）→ 判定对错 → 入库或删除。两种情况都不自动判定、不自动融合。
  - 「核心行程问答」用哪些规则做校验。
- **状态**：待处理（决策已定，等 V3 落笔；「争议大」判定项和「不一致时怎么处理」两项已决策，15:59 按 mao789 完整口径统一；双模型选型 16:00 已决策；仅核心问答的校验规则仍待定）

## V3-6：增加国内版、国外版两个版本

- **类型**：待办（新增）
- **提出**：用户通话转述，mao789 要求（2026-11-16 15:58 台北时间）
- **要求**：系统分国内版和国外版两个版本。
  - **国内版**：用国产模型，如 DeepSeek、豆包、千问。
  - **国外版**：用国际模型，如 GPT、Claude、Gemini。
- **两个版本可能不同的地方**（V3 落笔时要分别规划）：
  - **模型层（4.1）**：两版各自的模型选型。
  - **工具层（4.2 / 4.2.3）**：例如支付，国内版用支付宝 / 微信 MCP，国外版用 Stripe 等。地图、导航等其他 MCP 也要逐项核对。
  - **合规要求（第 11 节合规清单等）**：两版分别列。
- **和 v2 的对照，V3 前需要确认**：
  - v2 4.1 的主选模型（Qwen3.8 Omni Flash Realtime、MOSS、Nex N2.6）都是国产模型。GPT Realtime / Gemini Live 在 v2 里只是备选，原文为「只在客户明确要求或主模型挂掉时切换；**切换需要重新过合规**」。国外版改成以国际模型为主选后，这条备选定位要改写；国外版的转写、字段抽取、视觉匹配分别用哪个模型，v2 没写。
  - v2 1.6 把范围限定为「厦门目的地闭环」，4.2.3 写明「所有支付都是目的地本地支付」。需要确认国外版的定位：是「服务来厦门的海外游客」，目的地仍是厦门；还是「把系统用到海外目的地」。这会决定国外版是否仍走厦门本地的酒店、门票，以及支付通道怎么选。
  - 两个版本是同一套主线和子线、只在 4.1、4.2 换实现（v2 第 2 节「一切皆插件」的思路），还是两套独立部署。v2 没写。
  - 第 10 节成本、第 11 节合规清单（含 11 节「模型切换要重新过合规」一条）、第 5 节防幻觉里的双源校验（高德）可能都要按版本拆开写。
  - V3-5 里交叉验证的候选模型（DeepSeek、豆包、千问）属于国内版；国外版对应用哪些模型，要一并确定。（16:00 补充：V3-5 双模型选型已定，国内 DeepSeek 最新版或 Kimi，国外 GPT、Gemini，见 V3-5。）
- **状态**：待处理
