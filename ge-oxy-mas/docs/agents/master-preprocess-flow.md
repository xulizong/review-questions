# master_agent 预处理与分流

本文只解释 `master_agent` 常规主链路进入 planner 之前的三段逻辑：`add_data_desc`、`pre_process_clarifier` 和 `get_input_mode`。它们的作用不是执行分析任务，而是把用户输入、数据源元信息、知识召回和入口模式整理成后续 `data_planner` 可以直接使用的 `shared_data`。下面的 curl 按当前项目默认 `ENV=dev` 配置书写；如果运行时显式设置了 `ENV=pro/pre/local`，需要以对应 `configs/application-*.yml` 中的 URL 为准。

可以用这个例子理解主链路的输入：用户希望基于一个在线数据集做报告分析，`shared_data.type=2` 表示报告模式，`report_pref.book_id` 会被带到数据集元信息和语义层召回接口里，`tool_list` 中包含 `web_search_tool` 只影响 planner 提示，不会在预处理阶段立刻联网搜索。

```json
{
  "query": [
    {
      "type": "text",
      "content": "分析近30天销售额下降原因"
    }
  ],
  "shared_data": {
    "type": 2,
    "report_pref": {
      "outline": "auto",
      "outline_launchable": "",
      "book_id": 2842
    },
    "user_data": [
      {
        "type": "dataset",
        "content": "品策数据洞察SPU",
        "dataset_id": "1073097"
      }
    ],
    "system_knowledge": {
      "status": "disable"
    },
    "tool_list": ["web_search_tool"]
  },
  "group_data": {
    "user": {
      "pin": "<ERP>"
    }
  }
}
```

## `add_data_desc`

`add_data_desc` 定义在 `applications/master_agent/tools/file_tools.py`，它遍历 `shared_data.user_data`。当数据源类型是 `dataset` 时，它读取 `dataset_id`、登录人 `pin` 和 `report_pref.book_id`，调用 `trans_dataset_info_to_shared_data`，最终落到 `extends/datasource/dataset/dataset_client.py` 的 `get_dataset_info`。这个外部请求的请求体是 JSON 数组，不是对象，三个位置分别是登录人、数据集 ID 和 book ID。

```bash
curl --location 'http://debug.jd.com/fw/127.0.0.1/8881/com.jd.bpp.agents.query.data.analysis.dataset.api.DataSetRpcService/data-analysis-alias-pre/getBookDatasetDetail/jsf/20000' \
  --header 'Content-Type: application/json' \
  --data '["<ERP>", 1073097, 2842]'
```

如果当前请求没有 `report_pref.book_id`，代码会把第三个参数传成 `null`，对应的 Postman 请求可以这样查。

```bash
curl --location 'http://debug.jd.com/fw/127.0.0.1/8881/com.jd.bpp.agents.query.data.analysis.dataset.api.DataSetRpcService/data-analysis-alias-pre/getBookDatasetDetail/jsf/20000' \
  --header 'Content-Type: application/json' \
  --data '["<ERP>", 1073097, null]'
```

接口返回后，代码读取 `body.caption` 作为表名，`body.description` 作为数据集描述，`body.columnList` 作为字段列表，并把字段转成 `short_desc.detail.colDesc` 里的 Markdown 表格。`colDesc` 不是接口返回字段，而是代码根据 `columnList[*].columnCn` 和 `columnList[*].fieldType` 现场拼出来的展示字段。`filter_custom=True` 时只是本地过滤 `columnList` 中 `type == "CUSTOM"` 的字段，不会改变上面这个 HTTP 请求体，所以 Postman 查询时不需要加 `filter_custom` 参数。

以 `品策数据洞察SPU` 的返回为例，curl 返回中真正被使用的部分可以简化看成下面这样。这里的 `columnEn`、`customSql`、`dimCode`、`synonyms`、`columnTypeDesc` 等字段不会进入 `colDesc`，默认也不会直接展示到主 prompt 里。

```json
{
  "body": {
    "caption": "品策数据洞察SPU",
    "description": "JD和站外渠道SPU粒度销量数据，按月度更新。用于趋势洞察等场景，应用于黄金眼C面及采销工作台",
    "columnList": [
      {"columnCn": "SPU编码", "fieldType": "string", "type": "ORIGINAL"},
      {"columnCn": "SPU名称", "fieldType": "string", "type": "ORIGINAL"},
      {"columnCn": "销售额", "fieldType": "float", "type": "ORIGINAL"},
      {"columnCn": "销量", "fieldType": "float", "type": "ORIGINAL"},
      {"columnCn": "日期", "fieldType": "date", "type": "ORIGINAL"},
      {"columnCn": "京东市占", "fieldType": "float", "type": "CUSTOM"}
    ]
  }
}
```

`add_data_desc` 会把它整理成下面的 `short_desc.detail` / `long_desc.detail`。真实字段会完整列出；这里用省略号表示中间字段。你给的这份返回一共有 45 个字段，其中 5 个是 `CUSTOM` 字段：`京东市占`、`抖音市占`、`淘宝市占`、`天猫市占`、`阿里系市占`。

```json
{
  "table_name": "品策数据洞察SPU",
  "table_desc": "JD和站外渠道SPU粒度销量数据，按月度更新。用于趋势洞察等场景，应用于黄金眼C面及采销工作台",
  "colDesc": "| column name | value type |\n| --- | --- |\n| SPU编码 | string |\n| SPU名称 | string |\n| 销售额 | float |\n| 销量 | float |\n| 日期 | date |\n| 京东市占 | float |",
  "dataQuality": {
    "sampleData": []
  }
}
```

进入 planner prompt 时，它会变成 `all_data_summary` 这一段。这个展示面向计划生成，所以只强调“数据源名称、数据源描述、字段名和字段类型”。

```text
第1个数据源: 品策数据洞察SPU,类型为在线数据集,描述: JD和站外渠道SPU粒度销量数据，按月度更新。用于趋势洞察等场景，应用于黄金眼C面及采销工作台,包含的列信息:
| column name | value type |
| --- | --- |
| SPU编码 | string |
| SPU名称 | string |
| 销售额 | float |
| 销量 | float |
| 日期 | date |
| 京东市占 | float |
...
```

进入 `Text取数Agent` prompt 时，同一份字段表会被包进 `dataset_markdown`。这个展示面向工具选择和取数执行，所以额外带上 `dataset_id`。

```text
# 当前使用的在线数据源概述（仅限 text2data_tool 使用）
## 在线数据源列表: ['品策数据洞察SPU']
## dataset_id列表: ['1073097']
### 在线数据源详情:
- 数据源 1: **品策数据洞察SPU**

**数据类型**: dataset

**数据名称**: 品策数据洞察SPU

**dataset_id**: 1073097

**数据总结**: JD和站外渠道SPU粒度销量数据，按月度更新。用于趋势洞察等场景，应用于黄金眼C面及采销工作台

**数据字段信息**:

| column name | value type |
| --- | --- |
| SPU编码 | string |
| SPU名称 | string |
| 销售额 | float |
| 销量 | float |
| 日期 | date |
| 京东市占 | float |
...
```

当数据源类型是 `file` 时，`add_data_desc` 会先判断 workspace 下的本地文件是否存在。如果本地文件不存在且 `user_data` 中带了 `url`，代码会通过 `utils/download_util.async_download_file` 对这个文件 URL 发起 GET 下载；下载完成后，`trans_file_info_to_shared_data` 只用 pandas 在本地读取文件前几行和字段类型，不再调用外部服务。

```bash
curl --location '<file_url>' \
  --output '<file_name>'
```

如果本地已有部分文件，代码会带 `Range` 头做断点续传；Postman 里可以手动加同样的 header 复现。

```bash
curl --location '<file_url>' \
  --header 'Range: bytes=<already_downloaded_size>-' \
  --output '<file_name>'
```

普通 `/chat` 请求在进入 MAS 前，`webs/chat_router.py` 已经会把 `user_data.type == "file"` 的 URL 下载到 `get_workspace_dir(pin, group_id, trace_id)` 返回的目录，并把 `user_data.content` 改成本地文件名。`add_data_desc` 里的下载逻辑主要是兜底：例如重启、恢复执行或 workspace 文件丢失时，如果本地文件不存在，就再用 `user_data.url` 重新下载。

文件元信息读取由 `extends/datasource/file/file_client.py` 的 `get_file_info` 完成。它根据扩展名选择解析方式：`.csv` 会按 `utf-8`、`gbk`、`gb2312`、`gb18030`、`latin1` 顺序尝试读取前 10 行；`.xlsx` 用 `openpyxl`，`.xls` 用 `xlrd` 读取前 10 行；其它扩展名会被 `utils/pandas_util.py` 判为“不支持的文件类型”。读取成功后，它只抽取字段名、pandas 推断类型、最多 3 行样例和 CSV 编码，再由 `trans_file_info_to_shared_data` 组装成 `short_desc` / `long_desc`，其中 `table_desc` 固定是“用户上传的数据文件”。

如果文件内容不符合预期，当前代码不是统一走一个显式的“文件校验失败”状态，而是分阶段暴露问题。第一类是文件不可下载：普通 `/chat` 入口没有在下载处做本地 try/catch，下载异常会直接阻断请求；`add_data_desc` 的兜底下载会捕获异常并跳过当前 file，但如果请求里原本没有 `short_desc` / `long_desc`，后面的 `pre_process_clarifier` 仍可能因为取不到字段描述而失败。第二类是文件格式或内容无法被 pandas 解析：`get_file_info` 会捕获 `FileNotFoundError`、不支持扩展名、编码不支持、Excel 损坏等异常并返回 `None`，但 `trans_file_info_to_shared_data` 目前没有对 `None` 判空，会继续访问 `base_res['columns']`，因此可能抛出 `TypeError` 并阻断 `master_graph_flow` 中的预处理链路。第三类是文件能读成表，但业务内容不满足分析预期，例如列名不对、日期列格式混乱、金额列混有逗号/百分号/空值、默认 sheet 不是目标 sheet。这类不会在 `add_data_desc` 阶段失败，因为预处理只看前 10 行 schema；真正执行时由 `Text取数Agent` 调用 `py_coder_tool` 用 `pd.read_csv` / `pd.read_excel` 读取文件并处理，代码执行器会把 `KeyError`、`ValueError`、空结果等放进 `stderr`，让 ReAct 根据错误修正代码并重试。连续失败达到工具描述中的限制后，应在最终结论中说明无法完成计算的原因。

对于上面的 dataset 示例输入，`add_data_desc` 最终会把原来的数据集条目补成带 `short_desc`、`long_desc` 的结构，同时写入 `shared_data.dataset_id_name_map = {"1073097": "品策数据洞察SPU"}`，供后续语义层知识召回把数据集 ID 映射回可读名称。对于 file 输入，不会写入 `dataset_id_name_map`，后续主链路会把它作为“本地文件”放进 `all_data_summary` 和 `file_markdown`，并在纯本地文件场景优先让 `Text取数Agent` 使用 `py_coder_tool` 读取和加工。

## `pre_process_clarifier`

`pre_process_clarifier` 定义在 `applications/data_process/util/pre_process_clarifier.py`。它先把所有数据源的 `short_desc.detail.colDesc` 拼成 `shared_data.all_data_summary`，再根据 `tool_list` 判断是否写入 `planner_web_search_info`。随后它通过 Oxy 内部调用 `callee="search_knowledge"` 检索默认知识，并调用 `master_recall_few_shot` 召回 planner few-shot 示例。因此它本身不是 HTTP client，但它间接触发的外部请求集中在 `applications/master_agent/tools/new_master_tool.py` 和 `applications/master_agent/vearch_db/recall_memory.py`。

召回few-shot的原因是因为，给planner的prompt是静态的，和用户当前的query可能并没有太大关系。但是给他动态召回few-shot能让她更好的根据当前的问题去设计方案。

如果 `shared_data.system_knowledge.status == "enable"`，`search_knowledge` 会先查系统知识库。代码中的真实鉴权值不应写入文档或 Postman collection，下面用占位符表示。

```bash
curl --location 'http://ge-data-mas.jd.local/knowledge/retrieve' \
  --header 'Content-Type: application/json' \
  --header 'Authorization: Bearer <KNOWLEDGE_TOKEN>' \
  --header 'app_id: <KNOWLEDGE_APP_ID>' \
  --data '{
    "query": "分析近30天销售额下降原因",
    "rec_type": "outline",
    "rec_column": "outline",
    "retrieval_num": 2,
    "threshold": 0.4,
    "user_knowledge_id": []
  }'
```

接口返回的 `data[*].analysis_scene` 和 `data[*].outline` 会被格式化后写入 `default_knowledge`。以“分析近30天销售额下降原因”的实际召回为例，进入 planner 的系统知识内容如下：

```text
系统知识库：
第1篇<start>
标题：分析思路-交易分析；分析报告大纲如下：
第一章 交易分析框架与核心要求
1.1 分析目标与流程
先总结交易核心指标的整体升降趋势，并对下降指标进行维度拆解归因。
...
第1篇<end>

第2篇<start>
标题：分析思路-店铺转型、选品营销；分析报告大纲如下：
...
第2篇<end>
```

如果 `shared_data.user_knowledge` 里带了知识文件，`search_knowledge` 会把每个条目的 `knowledge_id` 取出来查用户知识库。接口还是同一个，只是召回类型从 `outline` 变为 `chunk`，召回字段从 `outline` 变为 `content`。

```bash
curl --location 'http://ge-data-mas.jd.local/knowledge/retrieve' \
  --header 'Content-Type: application/json' \
  --header 'Authorization: Bearer <KNOWLEDGE_TOKEN>' \
  --header 'app_id: <KNOWLEDGE_APP_ID>' \
  --data '{
    "query": "分析近30天销售额下降原因",
    "rec_type": "chunk",
    "rec_column": "content",
    "retrieval_num": 2,
    "threshold": 0.4,
    "user_knowledge_id": ["<KNOWLEDGE_ID>"]
  }'
```

用户知识接口返回的 `data[*].content` 会按片段格式化。以一个已上传的分析方法论知识文件为例，进入 planner 的用户知识内容如下：

```text
用户知识库：
第1条<start>
本方法论的业务逻辑遵循一个自上而下的目标分解路径，通过“经营目标 → 杠杆拆解 → 核心驱动 → 执行动作”构成整个方法论的骨架……
第1篇<end>

第2条<start>
优化现金流的核心抓手是提升库存周转率，并从体验、效率、成本三个方向拆解具体运营动作……
第2篇<end>
```

只要 `user_data` 里有 dataset，`search_knowledge` 还会查语义层知识。短 query 走 `config.dataset.semantics_server_url`，也就是 `callBackDatasetFull`，这个接口会带字段、枚举值和业务语义知识；请求中的 `tokenReportContext` 来自当前 OxyRequest 的 trace、node、callee category 和 group 上下文。代码会按数据集逐个请求，所以每次请求的 `datasetIdList` 只有一个 ID。

```bash
curl --location 'http://debug.jd.com/fw/127.0.0.1/8881/com.jd.bpp.agents.query.data.analysis.dataset.api.DataSetRagRpcService/data-analysis-alias-pre/callBackDatasetFull/jsf/180000' \
  --header 'Content-Type: application/json' \
  --data '{
    "datasetIdList": [1073097],
    "erp": "<ERP>",
    "query": "分析近30天销售额下降原因",
    "bookId": 2842,
    "tokenReportContext": {
      "traceId": "<TRACE_ID>",
      "nodeId": "<NODE_ID>",
      "nodeType": "<CALLEE_CATEGORY>",
      "groupId": "<GROUP_ID>"
    }
  }'
```

`callBackDatasetFull` 的返回不会直接执行取数，它只被整理成 prompt 里的“语义层知识库”。当前项目主要使用 `body.knowledges` 和 `body.callBackDatasetDTOS[].columns`：`knowledges` 非空时展示业务词解释；`columns` 只保留 `columnName`、`fieldType` 和 `fieldEnum`。`callStatsDTO`、`score`、`customSql`、`semanticSource`、`standardWord`、`originColumnName` 等不会进入 prompt。

以 `品策数据洞察SPU` 的返回为例，接口返回可以简化看成：

```json
{
  "body": {
    "callBackDatasetDTOS": [
      {
        "datasetSchema": {
          "datasetCaption": "品策数据洞察SPU"
        },
        "columns": [
          {"columnName": "SPU编码", "fieldType": "string", "fieldEnum": []},
          {"columnName": "销售额", "fieldType": "float", "fieldEnum": []},
          {"columnName": "日期", "fieldType": "date", "fieldEnum": []},
          {"columnName": "京东市占", "fieldType": "float", "fieldEnum": []}
        ]
      }
    ],
    "callStatsDTO": {
      "columnRecallWords": ["销售额", "下降", "近30天"]
    },
    "knowledges": []
  }
}
```

语义层知识最终进入 prompt 的形式如下。这个示例的 `knowledges` 为空，所以没有具体的业务知识内容；`fieldEnum` 也为空，所以“字段枚举值列表”为空。

```text
语义层知识库：
对于数据集：【品策数据洞察SPU】

### 与当前query可能相关的字段知识

| 字段名称 | 字段类型 | 字段枚举值列表 |
|------|------|------|
| SPU编码 | string |  |
| 销售额 | float |  |
| 日期 | date |  |
| 京东市占 | float |  |
...
```

当传入 `search_knowledge` 的 query 长度超过 `get_dataset_knowledge_query_len_threshold()` 返回的阈值时，代码改走 `config.dataset.sample_semantics_server_url`，也就是 `callBackKnowledge`。它仍然带同样的上下文，但返回内容更偏语义知识，不带完整字段知识。

```bash
curl --location 'http://debug.jd.com/fw/127.0.0.1/8881/com.jd.bpp.agents.query.data.analysis.dataset.api.DataSetRagRpcService/data-analysis-alias-pre/callBackKnowledge/jsf/180000' \
  --header 'Content-Type: application/json' \
  --data '{
    "datasetIdList": [1073097],
    "erp": "<ERP>",
    "query": "分析近30天销售额下降原因",
    "bookId": 2842,
    "tokenReportContext": {
      "traceId": "<TRACE_ID>",
      "nodeId": "<NODE_ID>",
      "nodeType": "<CALLEE_CATEGORY>",
      "groupId": "<GROUP_ID>"
    }
  }'
```

`master_recall_few_shot` 是另一条独立召回链路。它先把 query 发给 embedding 服务，代码要求 `Accept-Encoding: identity`，响应里 `outputs[0].data` 是 base64 编码的向量内容，程序会解码、归一化后再查 Vearch。

```bash
curl --location 'http://custom-big-data-emb.jd.local/v2/models/embedding/infer' \
  --header 'Accept-Encoding: identity' \
  --header 'Content-Type: application/json' \
  --data '{
    "model_name": "embedding",
    "inputs": [
      {
        "name": "text",
        "shape": [1],
        "datatype": "BYTES",
        "data": ["分析近30天销售额下降原因"]
      }
    ],
    "outputs": [
      {
        "name": "last_hidden_state_clip"
      }
    ]
  }'
```

Vearch 查询是第二步，请求体里的 `feature` 必须替换成上一步解码并归一化后的浮点数组。代码里 Vearch 使用 Basic Auth 生成 header，文档里不记录用户名和密码；如果你在 Postman 里查询，需要从有权限的配置渠道拿到鉴权信息后自行添加。

```bash
curl --location 'http://oxygent-aigc-kw.vearch.jd.local/document/search' \
  --header 'Content-Type: application/json' \
  --header 'Authorization: Basic <VEARCH_BASIC_AUTH>' \
  --data '{
    "space_name": "guojian31_ge_mas_master_few_shot_v0204",
    "db_name": "ge_mas_master",
    "vectors": [
      {
        "field": "field_vector",
        "feature": [<embedding_float_array>],
        "min_score": 0.8
      }
    ],
    "is_brute_search": 1,
    "limit": 2
  }'
```

对于示例输入，如果系统知识关闭且没有用户知识文件，`pre_process_clarifier` 仍然会查数据集语义层知识和 few-shot。它最终写入 `shared_data.default_knowledge`、`shared_data.recall_few_shot`、`shared_data.all_data_summary`、`shared_data.day_info` 和 `shared_data.day_info_simple`，这些字段会进入 planner prompt。

## `get_input_mode`

`get_input_mode` 定义在 `applications/master_agent/tools/master_tool.py`，它没有外部 HTTP 请求，也没有对应 curl。这个方法只读 `shared_data.report_pref.outline_launchable`、`shared_data.type` 和 `shared_data.hitl_subtype` 做本地分流。优先级最高的是 `outline_launchable`：只要它能被 `json.loads` 解析为合法 JSON，就直接返回 `direct_execute_with_global_layout`，不会再看 `type`。

在示例输入中，`outline_launchable` 是空字符串，无法解析成 JSON，所以继续看 `shared_data.type=2`，最终返回 `report`。如果 `type=1` 则返回 `chat`；如果 `type=4` 则进入 HITL 子模式，`hitl_subtype=1` 返回 `plan_placeholder`，`hitl_subtype=2` 返回 `direct_execute`，`hitl_subtype=3` 返回 `intention_enhance_placeholder`。这些分支决定 `master_agent` 后续走常规 planner、人工确认、直接执行还是增强占位符链路。

`report_pref.outline` 的 `auto/self` 不参与 `get_input_mode` 的入口分流，它在 planner 生成计划之后生效：`auto` 展示计划后直接执行，`self` 展示计划后等待 `/chat/append` 回传用户修改。完整设计见 [master-agent.md](./master-agent.md) 的“计划确认与自动执行”。
