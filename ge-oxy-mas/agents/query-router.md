# /data/query_router

`/data/query_router` 不是 Oxy Agent，但它是当前项目中重要的路由服务入口，因此单独成文档。入口在 `webs/data_router.py`，实现从 `applications/query_routing/api_handler.py` 进入，核心路由逻辑在 `applications/query_routing/query_router_service.py` 和 `applications/query_routing/llm_router/*`。

## 执行流

```text
POST /data/query_router
  ↓
handle_query_router
  ↓
prepare_query_rewrite
  ↓
route_query
  ├─ 店铺诊断前缀规则
  ├─ 客服场景 LLM 分类
  └─ 通用 LLM / Book 路由
      └─ BookRoutingEngine: BM25 + embedding + reranker + RRF
```

`handle_query_router` 会先调用 `prepare_query_rewrite` 做多轮追问改写，然后把改写后的请求交给 `route_query`。`route_query` 先匹配硬编码店铺诊断规则，再判断是否客服场景；客服场景会使用 `desc_mas.default_llm` 调用客服路由 prompt。若没有命中特殊规则，就进入通用 LLM/book 路由，Book 候选会经过 BM25、向量、reranker 和 RRF 融合。

## Prompt 对应

| Prompt | 位置 | 用途 |
| --- | --- | --- |
| `CUSTOMER_ROUTER_SYSTEM_PROMPT` | `applications/query_routing/prompt.py` | 客服上下文下判断应走客服知识问答、客服数据分析还是通用数据分析。 |
| `INITIAL_SYSTEM_PROMPT` | 同上 | 通用路由初筛，判断知识、分析、智能体或直接回答。 |
| `FINAL_SYSTEM_PROMPT` | 同上 | 在候选 Book profile 和检索分数基础上做最终路由。 |
| `REWRITE_SYSTEM_PROMPT` | 同上 | 根据历史对话把省略追问改写成可独立理解的问题。 |

这些 prompt 可通过 `extends/ducc/laf_instance.py` 中的 `get_query_router_*_system_prompt` 运行时覆盖。由于它不是 Oxy Agent，文档中不要把这些 prompt 归到 `master_agent` 或 `desc_agent` 子树下；它只是复用了 `desc_mas.default_llm` 做模型调用。

当前 DUCC/LAF 快照中的 query router prompt 内容见 [../prompt文档.md](../prompt文档.md)；其中 rewrite prompt provider 当前未被 DUCC 覆盖，会回退本地默认值。
