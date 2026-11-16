# 知识库 / 语义库架构 · 全球最新做法调研笔记

> 记录时间：2026-11-16 15:43（台北时间，UTC+8）· 来源：用户语音通话 + 当日网络调研（只读，未改任何规格文件）
> 本笔记只新增，不修改 `consolidated-spec-v2.md`、`odessia-spec.html` 或其他已有文件。资料日期、星标以 2026-11-16 查询为准；作者自报的指标未在厦门语料上复现。

## 0. mao789 的要求

- 以下先进做法要**评估并融入 xiamen-odessia 项目的语义库设计**（对应 `consolidated-spec-v2.md` 4.4 记忆层、第 7 章离线部分）。
- 总体结论：2025–2026 年主流仍是「混合检索 + 重排」，图谱和多模态是补强，不是替换。现有 4.4（五库共享 HNSW + BM25 + 结构化过滤 + RRF）与第 7 章（人工审核、只追加版本）方向正确。

## 1. 向量 + 关键词混合检索【可融入：直接进统一检索插件（4.4）】

- 做法：稠密（HNSW）与稀疏（BM25）并行召回，**融合前分别过滤**；RRF 融合（经典 k=60；注意 Qdrant 默认 k=2）；融合后对 top 50–100 做中文 cross-encoder 重排；评测看 Recall@K 和 nDCG@10。可与 DBSF 融合对比。
- Contextual Retrieval（Anthropic，2024-09）：入库前给每个切块补上下文（哪栋楼、年代、来源、所属库），作者报告失败检索约减少 49%，加重排约 67%。https://www.anthropic.com/engineering/contextual-retrieval
- BGE-M3（开源 MIT）：一个模型同时出稠密、稀疏、多向量，中文友好。https://github.com/FlagOpen/FlagEmbedding
- 原生混合检索引擎（以后若换）：Qdrant、Milvus（开源 Apache-2.0，优先）；OpenSearch、Vespa（Apache-2.0）；Weaviate（核心 BSD-3）；Elasticsearch（AGPL / SSPL / ELv2）；ParadeDB（AGPL）。SPLADE 原仓库为非商用许可。

## 2. GraphRAG 图检索【可融入：历史建筑档案、文化典故等有关联关系的内容】

- 建议：主检索不变，另建**领域小图**做关系扩展（建筑 ↔ 照片 ↔ 3D 锚点 ↔ 周边小吃 / 打卡点 ↔ 讲解典故）。
- Microsoft GraphRAG（开源 MIT）：官方称基本进入维护模式、索引成本高，不进主链路。https://github.com/microsoft/graphrag
- LightRAG（开源 MIT）：双层检索，支持增量插入和删除。https://github.com/HKUDS/LightRAG
- Neo4j GraphRAG for Python（开源 Apache-2.0）：先混合召回再图扩展。https://neo4j.com/docs/neo4j-graphrag-python/current/user_guide_rag.html
- KAG（蚂蚁 / OpenSPG，开源 Apache-2.0）：schema 约束的中文专业库，维护状态待核实。https://github.com/OpenSPG/KAG
- Graphiti（Zep，开源 Apache-2.0）：时序知识图谱，适合店铺开闭、菜单变更等会变的信息。https://github.com/getzep/graphiti
- 学术参考：中国建筑遗产多模态知识图谱（2026-10，CC BY-NC-ND）。https://www.nature.com/articles/s40494-026-03038-w

## 3. 多模态 RAG【可融入：图片、3D 建筑模型等多模态数据（4.5 视觉层、7.1 照片摄取）】

- Qwen3-VL-Embedding / Reranker（开源 Apache-2.0）：文本、图片、视频统一向量空间，中文友好；可与现有 CLIP 对比评估。https://github.com/QwenLM/Qwen3-VL-Embedding
- ColPali / ColQwen（MIT）：适合扫描旧档、地图等版式页面，存储成本高。https://github.com/illuin-tech/colpali
- VisRAG（开源 Apache-2.0）https://github.com/OpenBMB/VisRAG ；RAG-Anything（开源 MIT，已并入 LightRAG）https://github.com/HKUDS/RAG-Anything
- jina-embeddings-v4：Qwen Research License，商用待法务核实。
- 3D：未找到成熟的开源 3D 检索框架。建议约 50 栋轻量锚点先作为建筑实体的结构化属性（模型 URI、视角、编号），用过滤 / 图召回，讲解时再加载。

## 4. 2025–2026 年新开源框架进展【可融入：评测与流程借鉴】

- 未发现在公开基准上系统胜过「通用混合检索 + 重排」的旅游专用框架。
- RAGFlow（开源 Apache-2.0）：借鉴切块审阅、启停与引用流程，不整套替换现有 LangGraph + 五库。https://github.com/infiniflow/ragflow
- FlashRAG（开源 MIT）：离线对比检索 / 重排方法，用于第 8 章评测。https://github.com/RUC-NLPIR/FlashRAG
- Dify / FastGPT：Apache-2.0 + 附加条款，不建议作为核心依赖。

## 5. 知识库版本管理与人工审核【可融入：与第 7 章离线管道对齐】

- 现有第 7 章已具备：人工审核队列、24 小时 SLA、两级审核、日志可回滚、只追加的版本号。
- 需对齐 / 补充：
  - 草稿索引与已发布索引分开；审核通过才切换上线；线上只读已发布索引。
  - 按内容哈希只重建变更部分（增量切块 / 嵌入）。
  - 每条记录的版本元数据：记录 ID、版本号、审核人、入库时间、内容哈希。
  - 回滚用集合别名或 LanceDB 版本（开源 Apache-2.0）。https://docs.lancedb.com/oss/v0-41-0/tutorials/agents/time-travel-rag
  - 数据版本优先 DVC（开源 Apache-2.0）https://github.com/treeverse/dvc ；lakeFS 自 v1.87.0 改为 BSL，对外服务的生产链慎用。https://lakefs.io/blog/lakefs-business-source-license/
  - 审核界面可试 Label Studio（开源 Apache-2.0）。https://github.com/HumanSignal/label-studio
- 原则不变：运行时不自动修改知识库；图谱 / 增量更新都发生在人工审核通过之后。

## 6. 下一步（待决策）

- 以上各项何时、以何种形式并入 `consolidated-spec-v2.md` 4.4 / 4.5 / 第 7 章 / 第 8 章：待用户授权。
