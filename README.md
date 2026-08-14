# 用 MCP 精确恢复失败的 DolphinScheduler 工作流

**真实环境：DolphinScheduler 3.4.2 + dolphin-mcp-pilot + Hermes Agent**  
**运行日期：2026-08-14**

> 本文记录一次真实 MCP 工具运维实验：代理定位唯一失败任务，读取日志，从失败节点重跑，并用 DolphinScheduler 任务实例 ID 证明成功的上游任务没有重复执行。公开附件是经过筛选和脱敏的 MCP 操作摘要，不是原始 JSON-RPC 请求/响应封装；凭据、本地路径、主机名和内网 IP 均未公开。

![真实 MCP 运行记录](assets/targeted-recovery-transcript.png)

## 问题

两阶段看板流水线包含：

1. `extract_dashboard_rows`：生成 42 行看板数据；
2. `publish_dashboard`：发布看板。

发布阶段第一次执行时模拟临时闸门故障并返回退出码 `42`，第二次执行成功。给代理的请求是：

> The nightly dashboard workflow failed. Find the latest failure, read the failed task log, rerun only failed and pending tasks, and verify that the already successful task was not repeated.

关键要求不是“重新跑一次”，而是证明只恢复失败部分。

## 实际 MCP 操作

```text
ds_create_project
ds_create_workflow
ds_run_workflow
ds_list_process_instances
ds_list_task_instances
ds_get_latest_failure_log
ds_rerun_from_failure
ds_list_process_instances
```

诊断结果：

```text
instance_state=FAILURE
failed_task_count=1
failed_task=publish_dashboard
```

恢复操作：

```text
executeType=START_FAILURE_TASK_PROCESS
status=submitted
```

最终结果：

```text
state=SUCCESS
runTimes=2
extract_dashboard_rows=1
publish_dashboard=2
targeted_rerun_proved=true
```

经过筛选和脱敏的工具参数、结果、源顺序、DolphinScheduler 时间戳、实例 ID、任务 ID 与派生日志断言见 [`evidence/real-run-summary.json`](evidence/real-run-summary.json)。该文件记录的本地源记录 SHA-256 只是完整性指针；原始文件因包含机器本地路径和地址而不公开，因此外部读者不能独立验证该哈希。

## 为什么可以证明“只重跑失败节点”

仅看工作流最终变绿并不充分，所以同时检查失败日志标记和任务实例 ID：

| 任务 | 首次执行 | 恢复后累计 | 解释 |
|---|---:|---:|---|
| `extract_dashboard_rows` | 1 | **1** | 已成功任务未重复 |
| `publish_dashboard` | 1（失败） | **2** | 失败任务单独再执行一次 |

机器可读证据显示：

- 原上游任务在恢复前后保持相同 task-instance ID；
- 发布任务拥有不同的失败与成功 task-instance ID；
- 工作流实例 `runTimes=2`；
- 最终状态为 `SUCCESS`。

## 实际接入时发现的两个兼容问题

### 1. DolphinScheduler 3.4 的 REST 术语变化

3.4 将多组 `process` 路径和字段改为 `workflow`：

```text
process-definition        -> workflow-definition
process-instances         -> workflow-instances
processDefinitionCode     -> workflowDefinitionCode
processInstanceId         -> workflowInstanceId
```

兼容策略是保留旧请求为首选。只有目标路径确实改变且旧路径返回 `405` 或精确匹配 Jetty 标准路由缺失页面、并且页面中的 URI 与首次请求 URI 相同的 `404`，或者 HTTP/应用响应的完整错误消息精确报告缺少本次转换产生的 3.4 字段时，才进行一次受限转换重试。空正文、URI 不同、带额外内容或应用错误正文的 `404`、业务 `400`、仅包含缺字段片段或新字段名的业务错误以及 `ds_raw_*` 原样请求均不会触发兼容重试；raw 响应也不会被改写。`SUB_PROCESS` 的嵌套工作流代码只在第二次 3.4 请求中转换。

### 2. 显式 DAG 的虚拟根边

当调用者只提供：

```text
extract_dashboard_rows -> publish_dashboard
```

原实现没有补 DolphinScheduler 需要的起始关系：

```text
0 -> extract_dashboard_rows
```

结果是起始任务未进入顶点集合，服务端返回误导性的 `workflow node has cycle`。修复后，所有无入边任务会自动添加虚拟根边；基础和高级 DAG 创建工具会在任何 DolphinScheduler API 调用前共同拒绝空图、空/重复任务名、未知引用、重复边、自环、循环依赖以及与真实前驱冲突的显式根边。

## Windows 独立运行补充

本次验证在 Windows 上运行 DolphinScheduler 3.4.2 standalone：

- 使用官方 `cmd` shell interceptor；
- 官方 3.4.2 bootstrap 生成 `cmd.exe <task.bat>`，会在批处理结束后继续等待；
- 验证环境通过仅运行时前置 JAR改为 `cmd.exe /d /c <task.bat>`；
- 该运行时补丁不属于 dolphin-mcp-pilot 产品提交。

Linux 或容器部署不需要这条 Windows 说明。

## 回归测试

新增测试覆盖：

- 3.4 路径和字段转换；
- schedule 查询和 DAG 响应容器别名；
- 旧地址 HTTP 失败后的单次重试；
- 实际 DS 3.4 缺少 `workflowInstanceId` 错误后的受限重试；
- 普通业务错误和 raw 请求不重试；
- 3.4 响应字段向旧字段添加只读别名；
- 显式依赖图自动补虚拟根边；
- 多根、孤立任务、fork/join 与非法 DAG 输入；
- 公开运行证据字段的机器验证。

执行结果：

```text
74 non-E2E tests passed
55 focused compatibility/DAG/evidence tests passed
19 unittest tests passed
18 live E2E tests passed against DolphinScheduler 3.4.2
```

## 结论

`ds_get_latest_failure_log → ds_rerun_from_failure` 可以把常见故障恢复压缩为两个 MCP 操作，但真正可靠的自动运维还必须验证：

1. 失败任务是谁；
2. 恢复命令是否被接受；
3. 工作流最终是否成功；
4. 已成功任务是否未重复执行。

本案例用任务实例和独立计数同时证明了这四点。

---

- 案例作者：BBOSIK
- 运行主机：Hermes Agent（MCP host 类型 `other`）
- 原始凭据和未脱敏日志：未公开
