# LightRAG 知识图谱构建流程深度解析

## 整体架构

LightRAG 的知识图谱构建采用 **文档分块 -> LLM抽取 -> 合并写入 -> 持久化** 的四阶段流水线，核心代码分布在：

- `lightrag/lightrag.py` — 主入口与编排
- `lightrag/operate.py` — 抽取、合并、查询核心逻辑
- `lightrag/prompt.py` — LLM Prompt 设计
- `lightrag/kg/` — 图存储实现（NetworkX、Neo4j、PostgreSQL 等）

---

## 第一阶段：文档分块 (Chunking)

文档通过 `ainsert` (`lightrag.py:1370-1383`) 进入系统：

```python
tasks = [
    self.chunks_vdb.upsert(inserting_chunks),          # 向量存储
    self._process_extract_entities(inserting_chunks),  # 实体抽取
    self.full_docs.upsert(new_docs),                   # 全文存储
    self.text_chunks.upsert(inserting_chunks),         # KV存储
]
await asyncio.gather(*tasks)
```

- `chunker` 模块按 token 大小分块，每个 chunk 生成唯一 ID (`chunk-xxx`)
- 分块内容同时写入 `text_chunks` (KV存储) 和 `chunks_vdb` (向量存储)

---

## 第二阶段：实体与关系抽取 (Extraction)

核心函数 `extract_entities` (`operate.py:3232`) 对每个 chunk 调用 LLM 进行抽取。

### Prompt 设计

`prompt.py` 定义了两种抽取模式的 Prompt：

**文本模式** (`prompt.py:34-113`):

- 实体行格式: `entity<|#|>名称<|#|>类型<|#|>描述`
- 关系行格式: `relation<|#|>源实体<|#|>目标实体<|#|>关键词<|#|>描述`
- 使用 `<|#|>` 作为字段分隔符，`<|COMPLETE|>` 作为结束标记

**JSON模式** (`prompt.py:238-306`, 启用 `entity_extraction_use_json=True`):

- 输出 `{entities: [...], relationships: [...]}` 结构
- 提供更精确的结构化抽取，适合支持 JSON mode 的 LLM

Prompt 中包含：

- 实体类型指导（Person, Organization, Location, Concept, Method 等 11 类）
- 3 个精心设计的示例（叙事文本、学术论文、技术报告）
- 数量限制 (`max_entity_records`, `max_total_records`)
- 语言和命名约束

### Gleaning 二次抽取

(`operate.py:3398-3499`) 当 `entity_extract_max_gleaning > 0` 时：

1. 将初次抽取的 user prompt + LLM response 作为 history
2. 发送 `entity_continue_extraction_user_prompt`，让 LLM 补充遗漏的实体/关系
3. 合并时对比描述长度，保留更完整的版本
4. 有 token 预算保护：若 gleaning 的总输入超过 `MAX_EXTRACT_INPUT_TOKENS`，自动跳过

### 解析 LLM 输出

- `_process_extraction_result` (`operate.py:1197-1325`) 解析文本格式：按行分割、修复分隔符错误、逐行尝试解析为实体或关系
- `_process_json_extraction_result` (`operate.py:633-817`) 解析 JSON 格式：使用 `json_repair` 处理畸形 JSON，遍历 entities/relationships 数组
- `_handle_single_entity_extraction` (`operate.py:421-505`) 验证实体名称、类型、描述
- `_handle_single_relationship_extraction` (`operate.py:508-592`) 验证源/目标实体、关键词、描述

解析后输出：

- `maybe_nodes`: `{entity_name: [entity_data_list]}` 实体字典
- `maybe_edges`: `{(src, tgt): [edge_data_list]}` 关系字典

### 并发控制

通过 `asyncio.Semaphore(llm_model_max_async)` 控制并发 chunk 处理数量，使用 `asyncio.wait(FIRST_EXCEPTION)` 实现快速失败。

---

## 第三阶段：合并写入图谱 (Merge & Upsert)

`merge_nodes_and_edges` (`operate.py:2826`) 是核心合并函数，采用 **两阶段合并**：

### Phase 1 — 实体合并

`_merge_nodes_then_upsert` (`operate.py:1912-2238`)：

1. **查询已有实体**: `knowledge_graph_inst.get_node(entity_name)` 获取现有数据
2. **合并 source_ids**: 将新 chunk 来源 ID 与已有来源合并，存入 `entity_chunks_storage`
3. **应用来源限制**: `apply_source_ids_limit` 按 KEEP（保留最早）或 FIFO（保留最新）策略截断
4. **合并描述列表**: 已有描述 + 新描述，去重后按时间戳排序
5. **Map-Reduce 摘要**: `_handle_entity_relation_summary` (`operate.py:198-333`) 处理长描述：
   - 若总 token < `summary_context_size` 且条目数 < `force_llm_summary_on_merge` → 直接拼接
   - 否则分块摘要，每块调 LLM 生成摘要，递归合并直到满足限制
6. **确定实体类型**: 取 `Counter` 中出现最多的类型
7. **写入双存储**:
   - `knowledge_graph_inst.upsert_node()` → 图存储（节点含 entity_type, description, source_id, file_path）
   - `entities_vdb.upsert()` → 向量存储（嵌入内容: `entity_name + description`）

### Phase 2 — 关系合并

`_merge_edges_then_upsert` (`operate.py:2241-2581`)：

1. **查询已有边**: `knowledge_graph_inst.has_edge(src, tgt)` + `get_edge()`
2. **合并 source_ids**: 同实体合并逻辑，存入 `relation_chunks_storage`
3. **合并权重**: 新权重 + 已有权重求和
4. **合并关键词**: 解析逗号分隔关键词，去重后排序拼接
5. **合并描述**: 同样使用 Map-Reduce 摘要
6. **自动创建缺失节点**: 若关系端点实体不存在，创建占位节点 (`entity_type=UNKNOWN`)
7. **写入双存储**:
   - `knowledge_graph_inst.upsert_edge()` → 图存储
   - `relationships_vdb.upsert()` → 向量存储（嵌入内容: `keywords + src + tgt + description`）

关系视为 **无向边**，键使用 `sorted((src, tgt))` 保证方向无关性。

### Phase 3 — 索引更新

将文档涉及的实体名和关系对记录到 `full_entities_storage` / `full_relations_storage`，用于后续文档删除时的反向定位。

### 并发安全

使用 `get_storage_keyed_lock` 对每个实体名/关系对加锁，防止同一实体的并发写入冲突。Semaphore 控制并发度 (`llm_model_max_async * 2`)。

---

## 第四阶段：持久化

`_insert_done` (`lightrag.py:1410-1431`)：

```python
tasks = [storage_inst.index_done_callback() for storage_inst in [...]]
await asyncio.gather(*tasks)
```

对所有存储（full_docs, text_chunks, entities_vdb, relationships_vdb, chunks_vdb, chunk_entity_relation_graph, llm_response_cache 等）调用 `index_done_callback()`，将内存数据持久化到磁盘/数据库。

---

## 关键设计特点

| 特点                  | 代码位置               | 说明                                                         |
| --------------------- | ---------------------- | ------------------------------------------------------------ |
| **增量合并**          | `operate.py:1912,2241` | 新文档插入时与已有实体/关系合并，而非重建整个图谱            |
| **Map-Reduce 摘要**   | `operate.py:198-333`   | 描述过多时自动分块摘要再合并，控制 token 预算，避免信息丢失  |
| **无向图**            | `operate.py:2886-2888` | 关系键用 `sorted((src,tgt))`，视为无向边                     |
| **双存储架构**        | 全流程                 | 图结构存 GraphStorage（拓扑查询），语义检索存 VectorStorage（相似度搜索） |
| **并发安全**          | `operate.py:2923-3035` | `get_storage_keyed_lock` 对实体/关系粒度加锁                 |
| **Gleaning 二次抽取** | `operate.py:3398-3499` | 首轮抽取后再调 LLM 补充遗漏信息                              |
| **来源追踪与限制**    | `operate.py:1993-2017` | source_ids 限流策略 (KEEP/FIFO)，支持增量删除时反向定位      |
| **自动补全节点**      | `operate.py:2548-2566` | 关系端点实体不存在时自动创建占位节点                         |
| **LLM 缓存**          | `operate.py:3364-3374` | 抽取结果缓存到 `llm_response_cache`，避免重复调用            |

---

## 存储层架构

LightRAG 使用 4 类可插拔存储后端：

| 存储类型               | 用途                      | 默认实现             | 其他实现                        |
| ---------------------- | ------------------------- | -------------------- | ------------------------------- |
| **KV_STORAGE**         | LLM缓存、文本块、文档信息 | JsonKVStorage        | PostgreSQL, Redis, MongoDB      |
| **VECTOR_STORAGE**     | 实体/关系/块嵌入          | NanoVectorDBStorage  | Qdrant, Milvus, Faiss, PGVector |
| **GRAPH_STORAGE**      | 实体-关系图结构           | NetworkXStorage      | Neo4j, Memgraph, PostgreSQL     |
| **DOC_STATUS_STORAGE** | 文档处理状态              | JsonDocStatusStorage | PostgreSQL                      |

后端注册表在 `kg/__init__.py`，工厂方法 `kg/factory.py::get_storage_class()` 从配置解析后端类。

---

## 查询阶段简述

知识图谱建成后，查询通过 `kg_query` (`operate.py:3659`) 使用 5 种模式：

- **local**: 从实体向量检索相关实体 → 扩展到其邻居关系和文本块
- **global**: 从高层关键词检索关系 → 获取社区级摘要
- **hybrid**: 结合 local + global
- **naive**: 直接向量搜索，不使用图谱
- **mix**: KG检索 + 向量检索，推荐配合 reranker 使用

查询流程：关键词抽取 → 向量检索实体/关系 → 图扩展 → 构建上下文 → LLM生成回答