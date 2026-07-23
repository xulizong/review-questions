# data_agent

`data_agent` 是默认主 MAS 的第一层入口，定义在 `applications/data_agent/data_oxy_space.py`。`main.py` 创建 `MAS.create(oxy_space=data_oxy_space)` 后，如果 `/chat` payload 没有显式指定 `callee`，MAS 会进入注册顺序中第一个 `is_master=True` 的 Agent；当前这个 Agent 就是 `data_agent`。它不依赖 LLM prompt 做判断，而是由 `data_agent_workflow` 根据 `shared_data` 分流。

## 子 Agent 与职责

| 子 Agent | 位置 | 调用条件 | 功能 |
| --- | --- | --- | --- |
| `sql_proxy_agent` | `applications/sql_proxy_space/sql_proxy_oxy_space.py` | `shared_data.mode == ChatModelType.QUICK_DATA` | 快速查数代理，把请求提交给外部 SQLAgent，并消费 SSE trace 返回结果。 |
| `bi_proxy_agent` | `applications/bi_proxy_agent/bi_proxy_oxy_space.py` | `str(shared_data.type) == "3"` | BI 看板代理，先生成/确认大纲，再生成 dashboard。 |
| `master_agent` | `applications/master_agent/run_master_0130.py` | 其他普通数据分析请求 | 数据分析主控入口，负责规划、执行、取数、总结和输出。 |

## 执行流

```text
/chat
  ↓
services.chat_service.chat_with_agent
  ↓
MAS(data_oxy_space).chat_with_agent
  ↓
data_agent_workflow
  ├─ QUICK_DATA → sql_proxy_agent
  ├─ type == "3" → bi_proxy_agent
  └─ default → master_agent
```

`data_agent` 只做入口分流，不直接持有业务工具，也没有专属 prompt。它的输出直接返回所调用子 Agent 的 `rsp.output`。如果新增一个顶层业务模式，优先在这里增加明确的分流条件，并同步更新本文档和主索引。

## Tool 和 Prompt 对应

| 对象 | 对应关系 |
| --- | --- |
| Tool | `data_agent` 自身无业务工具；它通过 `req.call(callee=...)` 调用子 Agent。 |
| Prompt | 无直接 prompt；行为由 `data_agent_workflow` 代码控制。 |
| LLM | 注册了 `default_llm`，但当前分流逻辑不使用模型推理。 |

