# Augment Agent 日报增强 Agent

Augment Agent 是 `AGENT=augment-agent` 时的 master agent，代码入口是 `modules/agents/augment_agents/main.py`。它面向采销日报和增强报表场景，负责意图识别、任务创建、基础数据加载、字段新增、文件导入、报表展示和 EasyBI 组件生成。

## 1. Agent 组成

| 名称 | 类型 | 功能 | Prompt / 工具 |
| -- | -- | -- | -- |
| `augment_agent` | `WorkflowAgent` | master，日报增强总控 | `augment_agent_func_workflow` |
| `intent_agent` | `ReActAgent` | 识别 start、load_base_data、add_file、add_fields、display 等场景 | `prepare_agent.py` 内联 prompt |
| `generate_title_agent` | `ReActAgent` | 根据用户输入生成日报标题 JSON | `prepare_agent.py` 内联 prompt |
| `query_ds_agent` | `SSEOxyGent` | 远程数据源查询 | `config.agents.augment_agent.query_ds_agent_url` |
| `add_fields_agent` | `WorkflowAgent` | 新增字段，创建视图或写步骤 | `select`、`insert`、`execute` |
| `generate_field_agent` | `ReActAgent` | 生成字段表达式或字段规则 | `prompt_add_fields()` 注入 `${prompt}` |
| `add_file_agent` | `WorkflowAgent` | 解析上传文件并写入数据 | `select`、`insert`、`execute` |
| `display_agent` | `WorkflowAgent` | 查询并展示报表数据 | `select` |
| `group_select_agent` | `WorkflowAgent` | 分组查询辅助 | `select` |
| `easybi_agent` | `WorkflowAgent` | 生成 EasyBI 报表和组件步骤 | `select`、`execute`，子 Agent `component_agent` |
| `component_agent` | `ReActAgent` | 生成 EasyBI 组件配置 | `prompt_component()` 注入 `${prompt}` |
| `clickhouse_tools` | `SSEMCPClient` | ClickHouse 查询和写入工具 | `select`、`insert`、`execute` |

## 2. 主流程

`augment_agent_func_workflow(req)` 先把 `shared_data.app` 写入 `arguments.source`，因为历史逻辑大量使用 source 字段。随后它校验 source、pin、func_scene，并按 `augment_open_percent()` 和 `augment_sys_white_list()` 做灰度/白名单控制。如果 `func_scene` 缺失或等于 `intent`，workflow 调用 `intent_agent` 对用户问题分类。

当前主 workflow 直接支持的分支包括 `reconfigure`、`start`、`report_load_data`、`report_generate`、`report_easybi_steps`、`report_regenerate`、`report_list` 和 `report_detail`。部分旧意图如 `add_file`、`add_fields`、`display` 对应的子流程仍在文件中存在，但主日报分支更偏 `report_*` 场景。

```text
augment_agent
    ↓
source / pin / 白名单校验
    ↓
func_scene 缺失则 intent_agent 分类
    ↓
按场景调用日报任务、数据加载、EasyBI、字段或文件能力
```

## 3. 日报任务场景

`report_start` 会读取当前 ERP 和 source 的最新任务。如果没有任务，调用 `create_task_info()` 创建状态为 `99` 的任务并返回 task_id；如果已有任务，则返回当前 task_id 和状态。

`report_generate` 是完整生成路径。它根据 `isUpdate` 决定复用最新任务还是创建新任务，然后调用 `func_scene_report_load_data()` 加载基础数据。基础数据完成后发送一条 `data` 消息，再调用 `easybi_agent` 生成报表步骤和组件。如果 EasyBI 返回错误，workflow 会转换成业务错误码；成功则返回“本次报告任务已完成”。

`report_easybi_steps`、`report_list` 和 `report_detail` 主要委托 `easybi_agent`。`report_regenerate` 会读取最新任务和用户传入的 steps，再调用 `easybi_agent` 重新生成或调整报告。多个场景使用 `redis_lock`，锁 key 由 source 和 pin 组成，避免同一用户同一来源并发生成。

## 4. 字段、文件与 EasyBI 子流程

`add_fields_agent` 用于在已有 ClickHouse 表基础上新增字段或创建视图。它调用 `generate_field_agent` 生成字段表达式，prompt 由 `prompt_add_fields()` 注入，默认可回退到 `get_prompt_edit_field_default()`。执行层依赖 ClickHouse MCP 的 `select`、`insert` 和 `execute`。

`add_file_agent` 处理上传文件场景，负责解析 Excel 或文件头、读取批量数据并写入 ClickHouse。`easybi_agent` 用于生成 EasyBI 组件，内部可调用 `component_agent`。`component_agent` 的 prompt 由 `prompt_component()` 构造，通常结合 `get_prompt_easybi_component()` 和当前表元数据。

## 5. 示例

用户说“新增日报”。如果没有传 `func_scene`，`intent_agent` 可能输出 `start`。进入 start 分支后，Augment Agent 查询当前用户是否已有日报任务；没有则创建 task_id，并返回任务状态。随后用户触发“生成日报”时，`report_generate` 会先加载基础数据，再调用 EasyBI Agent 生成报表结构。

用户说“给日报新增一个毛利率字段”。场景可能进入字段新增能力。`add_fields_agent` 会准备当前表信息和用户规则，调用 `generate_field_agent` 生成字段逻辑，再通过 ClickHouse 工具创建视图或写入步骤。

用户上传 Excel 并要求加入日报。`add_file_agent` 会解析文件，识别表头和数据批次，然后通过 ClickHouse MCP 写表，供后续 EasyBI 或展示流程使用。
