# AGENT_SHARED — 团队 Agent 共享记忆

> 用途：替代不存在的 `tag=cluster` 共享记忆。关键结论写这里，提交后队友 `pull` 即同步。
> 纪律：只写**跨会话有价值**的结论；密钥/Token/密码一律不写；写完立即提交。

## 一、使用规则

1. 每条记录必须带日期与记录人（或 Agent 标识）。
2. 只记结论与决策，不记过程日志（过程放 commit message / Issue）。
3. 修改前先 `git pull`，避免冲突。
4. 敏感信息（API Key / token / 密码 / 内网地址凭据）**禁止写入本文件**，只写环境变量名。

## 二、团队共识与决策

| 日期 | 记录人 | 结论 |
|---|---|---|
| 2026-11-20 | Agent | 多 Agent 协同在 Trae 环境下 = Git 仓库 + 本文件 + Issues，不使用 RPC 通道 |

## 三、环境与工具事实（踩坑沉淀）

| 日期 | 记录人 | 事实 |
|---|---|---|
| 2026-11-20 | Agent | Trae 项目记忆按 workspace 路径隔离，切换 workspace 即切换记忆空间 |
| 2026-11-20 | Agent | npm 版 `@modelcontextprotocol/server-git`、`@docker/mcp-server` 已下架，git 用 `uvx mcp-server-git`，docker 用 `docker mcp gateway run` |
| 2026-11-20 | Agent | 由 Trae Shell 启动的常驻进程会在命令结束时被回收，需用 WMI `Win32_Process.Create` 脱离进程树启动 |

## 四、待办与分工

| 任务 | 负责人 | 状态 | 关联 Issue |
|---|---|---|---|
| 示例：补齐 MCP 配置文档 | 待认领 | 待开始 | #1 |
