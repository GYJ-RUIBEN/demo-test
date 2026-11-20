# Trae + WorkBuddy Agent 体系 · 最终执行清单（诚实可落地版）

> 本清单以「统一基准方案 V1.0」为底座，合并了多 Agent 协同、项目记忆切换、Git/Docker 扩展三项内容。
> 原则：**能落地的给精确命令，落不了地的给真实替代形态，不写空头支票。**

---

## 0. 先看这张总表（5 秒看清全局）

| 模块 | Trae 能否落地 | 真实落地形态 |
|---|---|---|
| MCP 底座（filesystem / sqlite / memory / fetch / run-python） | ✅ 能 | `mcp.json` 配置 + 逐包实测 |
| 四层记忆 + 项目记忆切换 | ✅ 能 | 复用 Trae 原生记忆文件，切 workspace 即切记忆 |
| Git 扩展 | ✅ 能 | Trae 内置 Git + 可选 `server-git` MCP |
| Docker 扩展 | ✅ 能（需本地 Docker） | `@docker/mcp-server` |
| 斜杠命令库 /xxx | ❌ 不能 | Trae 命令是内置 CLI / skill，**不能靠提示词注册** |
| Hook 事件总线 | ❌ 不能 | 无运行时承载 |
| 插件系统 | ❌ 不能 | 无运行时承载 |
| 定时自检 / 常驻监听 | ❌ 不能 | Trae 无后台常驻，降级为**手动** `/self-check` |
| 多 Agent RPC 协同 | ❌ 不能（RPC） | 真实形态 = 多人共享 Git 仓库 + 共享记忆文件 |

**结论先行**：本清单只把「✅ 能」的三块做成可执行步骤（Phase 1~3）；「❌ 不能」的部分在 Phase 4 给出**在 Trae 里真实可行的替代方案**，不再浪费时间去"部署"一个没有运行时承载的蓝图。

---

## Phase 0：环境预检（一次性，前置）

- [ ] 安装 **Node.js LTS**（`npx` 依赖，装好即带 npm）
- [ ] 安装 **Python 3.10+**
- [ ] 安装 **uv / uvx**（`mcp-run-python` 依赖）：`pip install uv` 或官网装 uv
- [ ] （可选）安装 **Deno**（`mcp-run-python` 部分场景需要）
- [ ] （仅 Docker 扩展需要）安装 **Docker Desktop** 并启动

验证（PowerShell 逐条跑，能打印版本号即通过）：

```powershell
node -v          # 期望 v20+
python --version # 期望 3.10+
uvx --version
docker --version # 可选
```

---

## Phase 1：MCP 底座（唯一真正"部署"的部分）

### 1.1 创建 MCP 配置

在项目根目录建 `.trae/mcp.json`（项目级），内容如下（RAG 后端先留空，二选一验证后再补）：

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "C:/Users/你的用户名/Documents/Trae-CN/default"]
    },
    "sqlite": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-sqlite", "./data/agent.db"]
    },
    "memory": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-memory"]
    },
    "fetch": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-fetch"]
    },
    "run-python": {
      "command": "uvx",
      "args": ["mcp-run-python", "stdio"]
    }
  }
}
```

> 注意：`filesystem` 的 `args` 最后一项必须是**你授权的真实工作目录绝对路径**，改成你自己的，不要照抄。

### 1.2 逐包验证（装一个、验一个，别一锅端）

| 包 | 用途 | 验证方法 |
|---|---|---|
| filesystem | 文件读写、目录列表 | 调 `list_directory` 看能否列出工作目录 |
| sqlite | 审计日志、任务、配置 | 调 `query` 执行 `SELECT 1` |
| memory | 短事实 KV 记忆 | 写一条 → 读回 → 删除 |
| fetch | 网页抓取 | 抓一个公开网页看返回正文 |
| run-python | Python 沙箱执行 | 跑 `print(1+1)` 看返回 2 |

**通过标准**（V1.0 第 7.5 节）：每个包必须「发现工具 + 至少一次真实调用成功」，才允许标记 READY。`npx` 启动成功 ≠ 工具可调用。

### 1.3 常见坑

- `npx` 首次拉包慢/失败 → 换网络或配 npm 镜像。
- `sqlite` 若报版本问题（V1.0 提到 `server-sqlite` 历史版本有坑）→ 换 PyPI 版 `mcp-server-sqlite` 或社区稳定版。
- `run-python` 报缺 Deno → 回退装 Deno，或改用本地受限 runner。

---

## Phase 2：四层记忆 + 项目记忆切换（复用 Trae 原生机制，不另造）

**关键判断**：Trae 本身已经有记忆系统，**不需要再靠 memory/sqlite-vec MCP 重造一遍**。直接映射即可，省掉一半工作量。

### 2.1 四层记忆 → Trae 原生文件映射

| 方案里的分区 | Trae 原生对应 | 位置 |
|---|---|---|
| system-config | 项目规则/约束 | `项目目录/.trae-cn/memory/.../project_memory.md` |
| user-long | 用户跨项目偏好 | `user_profile.md`（全局） |
| project-{id} | 每项目专属记忆 | 各项目目录下的 `project_memory.md` + `topics.md` |
| session-tmp | 会话临时记忆 | `session_memory_*.jsonl`（自动） |

### 2.2 项目记忆切换 = 切 workspace 目录

- [ ] 验证：打开项目 A 的 workspace，让 Agent 写一条"项目 A 专属信息"到 project_memory
- [ ] 切换到项目 B 的 workspace，确认读不到项目 A 的记忆（Trae 按项目目录自动隔离）
- [ ] 结论：**不需要 `/switch-project` 命令**，切换目录就是切换记忆空间，Trae 已原生支持

### 2.3 记忆安全红线（沿用 V1.0 R8）

- [ ] 明文密钥 / API Key / token **绝不写入**任何记忆文件
- [ ] 密钥只做环境变量引用

---

## Phase 3：Git 扩展 + Docker 扩展（按需）

### 3.1 Git 扩展

- [ ] **首选**：直接用 Trae 内置 Git 能力（提交、分支、PR），无需额外 MCP
- [ ] **可选增强**：需要仓库级操作时，追加 `server-git` MCP

```json
"git": {
  "command": "npx",
  "args": ["-y", "@modelcontextprotocol/server-git", "--repository", "./"]
}
```

> 注意：npm 版 `server-git` 曾下架，装不上就换 PyPI 版 `mcp-server-git`。

团队协作规范直接沿用你的《GitHub 竞赛团队协作使用手册》（分支规范 / PR 评审 / tag 发版）。

### 3.2 Docker 扩展

- [ ] 前置：Docker Desktop 已安装并运行
- [ ] 追加 `@docker/mcp-server`

```json
"docker": {
  "command": "npx",
  "args": ["-y", "@docker/mcp-server"]
}
```

> Docker / Terminal 权限高，**必须限制命令白名单**，不开放给所有 Agent 实例。

---

## Phase 4：多 Agent 协同（诚实替代方案）

**核心事实**：Trae 没有 Agent RPC 通道，`/agent-call`、`on_agent_request` Hook、`tag=cluster` 共享记忆——这些在 Trae 里**没有运行时承载**，写进清单也只是一段永远不执行的文字。

9 人竞赛团队的真实协同形态，是你已经做过的这套：

- [ ] **代码协同**：9 个 Trae 实例 = 9 个 workspace，共享同一个 GitHub 仓库，走分支 + PR 评审（见你的 GitHub 手册）
- [ ] **记忆协同（cluster 记忆的替代）**：约定一个共享记忆文件，通过 Git 提交同步
  - 方案：在仓库里放 `docs/AGENT_SHARED.md`，团队约定"跨会话的关键结论写这里，提交后队友 pull 即同步"
  - 效果 ≈ `tag=cluster`，只是用 Git 而非 RPC 做同步
- [ ] **任务协同**：用 GitHub Issues / 项目看板分配任务，替代 `/agent-call` 调度

**诚实结论**：多 Agent 协同在 Trae 里 = **「Git 仓库 + 共享文档 + Issues」**，而不是 RPC 调用。这套你已经具备，无需再"部署"。

---

## Phase 5：验收清单（只保留可落地项）

- [ ] MCP 底座 5 个包全部 discover + 真实调用通过
- [ ] 四层记忆映射生效，跨项目隔离验证通过
- [ ] Git 扩展（内置或 MCP）可正常提交/推送
- [ ] Docker 扩展（如启用）可列出容器
- [ ] 团队共享记忆文件 Git 同步跑通
- [ ] 密钥明文零写入（抽查记忆文件）

---

## 不做清单（明确降级，避免白忙）

| 原方案项 | 处置 |
|---|---|
| 斜杠命令库 /xxx | 用 Trae 内置命令 / skill 替代，不尝试注册 |
| Hook 事件总线 + 插件系统 | 放弃，无运行时 |
| 定时自检 600s | 降级为手动 `/self-check`（或按需让 Agent 跑一次自检） |
| 多 Agent RPC | 降级为 Git + 共享文档 + Issues |

---

## 执行顺序总结

```
Phase 0 环境 → Phase 1 MCP 底座（核心）→ Phase 2 记忆映射 → Phase 3 Git/Docker → Phase 4 团队协同 → Phase 5 验收
```

**最快可交付的成果** = Phase 0 + Phase 1 + Phase 2（这三步做完，你就有一个「文件/代码/抓取/存储/记忆」能力齐备、且真正可用的 Agent 工作环境，对应 V1.0 里唯一能在 Trae 落地的部分）。

---

# 执行记录（2026-11-20，本机实测）

## Phase 0 环境预检结果

| 组件 | 结果 | 说明 |
|---|---|---|
| Node.js | ✅ v18.17.1 | 可用但低于清单期望 v20+，filesystem 用固定 0.5.1 版兼容 |
| npm / npx | ✅ 9.6.7 | 可用 |
| Python | ✅ 3.13.9（Anaconda，D:\develop\anaconda） | 可用 |
| uv / uvx | ✅ 已装 0.13.9 | winget 安装（GitHub 下载较慢但成功） |
| Deno | ✅ 已装 2.9.6 | winget 下载 GitHub 失败 → 改走 `npm install -g deno`，并把真实 `deno.exe` 复制到 `C:\Users\gyj\AppData\Local\Deno`（已加入用户 PATH） |
| Docker | ⏭ 跳过 | 仅 Phase 3 需要，本次未装 |

## Phase 1 MCP 底座验证结果（逐包 discover + 真实调用）

| 包 | 结果 | 启动方式（已写入 `.trae/mcp.json`） | 实测内容 |
|---|---|---|---|
| filesystem | ✅ PASS | `npx -y @modelcontextprotocol/server-filesystem@0.5.1 <工作目录>` | `list_directory` 成功列出工作目录（9 个工具） |
| sqlite | ✅ PASS | `uvx --with "mcp[cli]==1.6.0" --with "pydantic<2.11" mcp-server-sqlite --db-path ...` | `read_query` 执行 `SELECT 1` 返回 `[{'ok': 1}]`（6 个工具） |
| memory | ✅ PASS | `npx -y @modelcontextprotocol/server-memory` | 创建实体 → `read_graph` 读回 → 删除，全通过（9 个工具） |
| fetch | ✅ PASS | `uvx mcp-server-fetch` | 抓取 https://example.com 返回正文 |
| run-python | ✅ PASS | `uvx mcp-run-python stdio` | `run_python_code` 执行 `print(1+1)` 返回 `2`（沙箱经 Deno+Pyodide） |

**结论：5/5 全部 READY**（每个包都完成了「工具发现 + 至少一次真实调用」）。

## 执行中踩坑与处置（对应清单 1.3 预案）

1. **npm 版 `server-sqlite`、`server-fetch` 均已下架（404）** → 按预案换 PyPI 版走 uvx。
2. **sqlite 新版 mcp SDK 不兼容**（`list_resources` 被移除、pydantic 2.11+ 移除 `eval_type_backport`）→ 用 `--with` 固定 `mcp[cli]==1.6.0` + `pydantic<2.11`。
3. **filesystem 最新版要求 Node 20+**（`util.getSystemErrorMessage`）→ 本机 Node 18，固定 `@0.5.1` 版；升级 Node 后可去掉版本号。
4. **run-python 缺 Deno** → 清单预判命中；winget 装 Deno 时 GitHub 下载两次失败（0x80072efd / 0x80072ee2），改 `npm install -g deno` 成功，并解决 `deno.cmd` 壳无法被 Python 子进程直启的问题（复制真实 `deno.exe` 到稳定目录入 PATH）。
5. **uvx 首次拉包慢** → 已设用户级环境变量 `UV_PYTHON_PREFERENCE=only-system`，让 uvx 复用本机 Python，避免再从 GitHub 下载托管解释器。

## 落地物

- MCP 配置：`C:\Users\gyj\Documents\Trae-CN\default\.trae\mcp.json`（5 个包，uvx 用绝对路径启动，不依赖 PATH 状态）
- sqlite 数据库：首次在 Trae 中调用时自动创建于 `data/agent.db`
- 验证脚本（可复跑）：`C:\Users\gyj\AppData\Local\Temp\agent-verify\verify-mcp.mjs`（用法 `node verify-mcp.mjs <包名>`）

## 待用户完成的收尾动作

1. **重启 Trae** 使 `.trae/mcp.json` 生效，并在 Trae 的 MCP 面板确认 5 个包显示为已连接。
2. （可选）**升级 Node 到 LTS**：管理员终端运行 `winget install OpenJS.NodeJS.LTS`；升级后把 mcp.json 里 filesystem 的 `@0.5.1` 去掉即可用最新版。
3. 确认后进入 Phase 2（四层记忆映射，复用 Trae 原生机制，无需新装）。

## Phase 2 四层记忆映射验证结果（2026-11-20，重启后实测）

| 清单项 | 结果 | 实证 |
|---|---|---|
| 2.1 四层映射 | ✅ 基本成立，1 处更正 | `project_memory.md`、`topics.md`、`session_memory_*.jsonl` 均已实测存在且由 Trae 自动写入；**`user_profile.md` 全盘不存在**（Trae 不预建，按需生成），映射表据此更正 |
| 2.1 可写验证 | ✅ PASS | 向当前项目 `project_memory.md` 追加「Phase 2 记忆映射验证」条目，落盘生效（文件可读回） |
| 2.2 项目记忆切换 | ✅ 成立，无需 `/switch-project` | `memory/projects` 下存在 **18 个项目目录**，目录名 = workspace 路径编码（如 `-c-Users-gyj-Documents-Trae-CN-default--p2-ff1f...`）；抽读 3 个项目对比，内容互不相通 |
| 2.3 安全红线 | ✅ PASS | 严格模式（sk-/api_key/password/secret/BEGIN 等）**零命中**；宽松模式仅命中 1 行 = 红线声明文字本身，**无真实明文密钥** |

### 关键发现（如实记录）

1. **记忆按 Trae workspace 路径隔离，而非项目代码路径**。实例：workspace `Trae-CN\temp-20261102-01` 里做的是"爆款雷达"项目（代码在 `E:\爆款雷达\baokuan`），其记忆落在 `-c-...-temp-20261102-01--p2-...` 目录下。→ **切 workspace 才切记忆**；两个不同 workspace 打开同一份代码，记忆互不相通。
2. **user-long 层（`user_profile.md`）当前未创建**：Trae 不预建该文件，首次需要时按需生成或由用户手动创建。清单 2.1 映射表原写"user_profile.md（全局）"成立，但应标注"待创建/按需生成"。
3. 目录结构实证：项目目录下按日期分文件夹（20261119/20261120），`topics.md` 记主题摘要，`session_memory_{session_id}.jsonl` 自动追加会话记录；`project_memory.md` 存放规则/约束/教训（Phase 0/1 的踩坑已被 Trae 自动沉淀进去）。
4. 本次在**单 workspace 内**完成文件级隔离实证；"Agent 会话中跨项目读不到"由目录名编码机制保证（同一会话不会加载其他 workspace 的目录），如需行为级复核可另开 workspace 实测。

## Phase 3.1 Git 扩展验证结果（2026-11-20 实测）

| 项 | 结果 | 实证 |
|---|---|---|
| Git 环境 | ✅ git 2.41.0，全局身份已配置 | `郭伊鉴` / `13304832+guo-yijian123456@user.noreply.gitee.com`（Gitee） |
| Trae 内置 Git | ✅ PASS | 临时测试仓库全链路实测：`init → add → commit → log → checkout -b → 分支提交` 全部成功（两次提交 14c0bf5 / a17dc04） |
| server-git MCP（npm 版） | ❌ 下架（清单预判命中） | `npm view @modelcontextprotocol/server-git` 返回 404 |
| server-git MCP（PyPI 版） | ✅ PASS | `uvx mcp-server-git` 启动成功，**14 个工具**发现，`git_status`（识别 feature/test 分支）+ `git_log`（读回 2 条提交）真实调用通过 |

### 结论与启用方式（如实）

1. **首选内置 Git 已足够**：Agent 通过 Shell 直接执行 git 命令，提交/分支/推送全可用，无需任何 MCP。团队协作规范沿用《GitHub 竞赛团队协作使用手册》（分支规范/PR 评审/tag 发版）。
2. **server-git 为可选增强**：仅在需要"Agent 以结构化工具调用方式读仓库状态"时追加。**未写入 `.trae/mcp.json`**，原因：当前 workspace（文档目录）不是 git 仓库，`--repository` 无处可指，配了启动即失败。在真实项目仓库里启用时，把下面条目加入该项目的 `.trae/mcp.json`（`--repository` 换成真实仓库路径，uvx 路径照抄本机绝对路径）：

```json
"git": {
  "command": "C:/Users/gyj/AppData/Local/Microsoft/WinGet/Packages/astral-sh.uv_Microsoft.Winget.Source_8wekyb3d8bbwe/uvx.exe",
  "args": ["mcp-server-git", "--repository", "<你的仓库绝对路径>"]
}
```

3. 验证资产（可复跑）：`C:\Users\gyj\AppData\Local\Temp\agent-verify\git-test-repo`（测试仓库）+ `verify-mcp.mjs` 已新增 `git` 条目，复跑命令 `node verify-mcp.mjs git`。

## Phase 3.2 Docker 扩展验证结果（2026-11-20，进行中 → 待重启）

| 项 | 结果 | 实证 |
|---|---|---|
| Docker Desktop 安装 | ✅ 成功 | 官方安装包静默安装完成，`C:\Program Files\Docker\Docker\Docker Desktop.exe` 存在；`docker --version` = 29.9.3 |
| `@docker/mcp-server`（npm） | ❌ 404 下架 | 与 server-git 同一情况，不在 npm registry |
| Docker 引擎启动 | ❌ 阻塞 | `docker info` 报 `npipe:////./pipe/docker_engine` 不存在；Docker Desktop 后端进程启动后退出 |
| 根因 | WSL2 未就绪 | ① 该机 WSL2 内核未安装；② `wsl --install` 报 **403 已禁止**（微软源被拦）；③ 旧版 `wsl_update_x64.msi`(5.10.16) 装不上 build 26200（1603）；④ 改用 `winget install Microsoft.WSL` → 成功装上 **WSL 3.0.5** |
| 第二阻塞 | C 盘空间不足 | 安装 WSL 时报仅剩 9,700 KB；清理前 C 盘剩余 0.69 GB / 182.7 GB |
| 磁盘清理 | ✅ 已执行（经用户授权） | 删除 pip 残留解包目录 2336 MB、本次下载的安装包 611 MB、wct*.tmp 255 MB、编辑器更新包残留 694 MB；清理后剩余 **4.5 GB** |
| 当前状态 | ⏸ 待重启 | `wsl --status` 报"WSL2 无法启动：虚拟机平台未启用"；`HypervisorPresent=False`、`CBS_RebootPending=True` → **必须重启系统**激活虚拟化层 |
| 已确认前提 | ✅ | `VirtualMachinePlatform` / `Microsoft-Windows-Subsystem-Linux` 两个功能均"已启用"；CPU 虚拟化固件已开（VirtualizationFirmwareEnabled=True） |

### 重启后的续跑步骤（如实记录，未完成）

1. 重启后执行 `wsl --status`，确认默认版本 2 且无"未启用虚拟机平台"报错。
2. 启动 Docker Desktop，轮询 `docker info --format "{{.ServerVersion}}"` 直到返回版本号。
3. 验证 `docker mcp` 子命令（Docker 官方内置 MCP 工具套件，替代下架的 `@docker/mcp-server`）：
   - 执行 `docker mcp version` / `docker mcp tools list` 确认可用；
   - 若可用，把 `docker mcp gateway run` 写入项目 `.trae/mcp.json`（stdio 传输）。
4. 如实记录 PASS/FAIL，再进入 Phase 4。

> 备注：本机 C 盘长期偏紧（清理前仅 0.69 GB 空闲），Docker/WSL 运行还需额外数 GB（WSL 虚拟磁盘随使用增长）。建议后续把 Docker 数据盘或 WSL 发行版迁移到 D 盘（D 盘剩 20 GB），否则可能反复触发空间不足。

### D 盘迁移执行记录（2026-11-20，已完成，待重启验证）

用户决定把 Docker 全部迁到 D 盘。已执行：

| 步骤 | 结果 | 说明 |
|---|---|---|
| Docker 卸载 | ✅ | `Docker Desktop Installer.exe uninstall --quiet` 退出码 0 |
| 重装到 D 盘 | ✅ | `install --quiet --accept-license --backend=wsl-2 --installation-dir=D:\Docker` 退出码 0；`D:\Docker\resources\bin\docker.exe --version` = 29.9.3 |
| 程序占用 | D 盘 3,306 MB | 系统 PATH 已写入 `D:\Docker\resources\bin`（新会话生效） |
| 数据目录 | `D:\DockerData` | `%APPDATA%\Docker\settings-store.json` 写入 `"DataFolder": "D:\\DockerData"`（已从二进制确认键名有效） |
| WSL swap | `D:\WSL\swap.vhdx` | `%USERPROFILE%\.wslconfig`：memory=4GB、swap=2GB |
| 磁盘现状 | C: 5.16 GB / D: 16.82 GB | 迁移前 C 盘仅 0.69 GB，现空出约 4.5 GB |
| 安装包清理 | ✅ | D:\Temp 下 595 MB 安装包已删除 |

仍待重启：`HypervisorPresent=False` + `CBS_RebootPending=True`（虚拟化功能已启用但未生效）。

### 重启后验证结果（2026-11-20，Phase 3.2 完成）

| 验证项 | 结果 | 实证 |
|---|---|---|
| 虚拟化生效 | ✅ | 重启后 `HypervisorPresent=True`、`CBS_RebootPending=False`、`wsl --status` 默认版本 2 |
| WSL 发行版 | ✅ | `docker-desktop` 发行版自动创建，`uname -a` 返回 Linux 6.18.58.1-microsoft-standard-WSL2 |
| Docker daemon | ✅ | `docker info` 返回 ServerVersion 29.9.3，driver=overlay2 |
| 容器列表 | ✅ | `docker ps -a` 正常返回（当前无容器） |
| `docker mcp` 工具 | ✅ | `docker mcp version` = v0.44.3；`docker mcp tools list` 列出 8 个工具 |
| MCP 网关协议验证 | ✅ PASS | `docker mcp gateway run` 完成 initialize + tools/list（serverInfo: Docker AI MCP Gateway 2.0.1，8 工具），`mcp-find` 真实调用成功 |
| 拉取镜像运行容器 | ❌ 网络受限 | `docker run hello-world` 报连不上 `registry-1.docker.io`（Client.Timeout）→ 本机网络无法访问 Docker Hub，非 Docker 配置问题 |

**关键踩坑（重要经验）**：Docker Desktop 由 Trae 的 Shell 命令启动时，会在该命令结束时被**连带杀掉**（进程树/作业对象回收），表现为"后端启动后几秒消失、日志停在 `waiting for docker daemon`"。改用 WMI 脱离进程树启动即可：

```powershell
Invoke-CimMethod -ClassName Win32_Process -MethodName Create -Arguments @{ CommandLine = '"D:\Docker\Docker Desktop.exe"' }
```

**已写入 `.trae/mcp.json` 的条目**（替代下架的 `@docker/mcp-server`）：

```json
"docker": {
  "command": "D:/Docker/resources/bin/docker.exe",
  "args": ["mcp", "gateway", "run"]
}
```

> 依赖提醒：该 MCP 条目要求 Docker Desktop 处于运行状态；Docker 未启动时该条目会连接失败。当前 Docker 引擎在重启后不会自动拉起（`AutoStart=false`），如需长期可用，建议在 Docker Desktop 设置中开启开机自启。另：Docker 数据目录当前仍在 `C:\Users\gyj\AppData\Local\Docker\wsl`（`DataFolder` 设置在发行版首次创建时未生效），C 盘因此再降到约 2.6 GB；如需彻底迁到 D 盘，需在 Docker Desktop 设置里改 Disk image location 后重启引擎。

### Docker 数据迁 D 盘（2026-11-20，最终成功）

| 尝试 | 结果 |
|---|---|
| 停止 Docker + `wsl --shutdown`，robocopy 迁移 `wsl` 目录到 `D:\DockerData\wsl` | ✅ 文件迁移成功（1466 MB + 96 MB） |
| 修改注册表 `Lxss\{guid}\BasePath` 指向 D 盘 | ❌ 被覆盖：Docker 启动时重新注册回 `C:\Users\gyj\AppData\Local\Docker\wsl\main` |
| `settings-store.json` 写入 `DataFolder: D:\DockerData\wsl\disk` | ❌ 未生效：引擎启动后仍在 C 盘生成 `docker_data.vhdx`(1484 MB) 与 `ext4.vhdx`(96 MB) |

**结论（第一次尝试）**：本机 Docker Desktop（29.9.3 / 4.99）的 WSL 发行版位置**不受 `DataFolder` 控制**，手工搬移会在启动时被重置。补充验证：二进制中存在的 `WslDataFolder` 键同样被忽略。

### ✅ 最终解决方案：目录联接（Junction，已成功）

Docker 4.99 把路径硬编码为 `%LOCALAPPDATA%\Docker\wsl`，因此不搬文件、而是**把这个路径本身变成指向 D 盘的联接**：

```powershell
# 1. 停止 Docker 并注销发行版
& "D:\Docker\resources\bin\docker.exe" desktop stop
wsl --unregister docker-desktop

# 2. 删除 C 盘旧目录，在 D 盘建真实目录
Remove-Item "C:\Users\gyj\AppData\Local\Docker\wsl" -Recurse -Force
New-Item -ItemType Directory -Path "D:\DockerData\wsl" -Force

# 3. 创建目录联接（C 路径 → D 路径）
New-Item -ItemType Junction -Path "C:\Users\gyj\AppData\Local\Docker\wsl" -Target "D:\DockerData\wsl"

# 4. 启动（必须用 WMI 脱离进程树，见上节）
Invoke-CimMethod -ClassName Win32_Process -MethodName Create -Arguments @{ CommandLine = '"D:\Docker\Docker Desktop.exe"' }
```

验证结果：

| 验证项 | 结果 |
|---|---|
| 数据物理位置 | ✅ `D:\DockerData\wsl\disk\docker_data.vhdx`(1498 MB) + `main\ext4.vhdx`(96 MB) |
| Docker 引擎 | ✅ `docker info` = 29.9.3 |
| 容器/MCP | ✅ `docker ps -a` 正常；`docker mcp version` = v0.44.3 |
| C 盘释放 | ✅ 2.65 GB → **4.15 GB**（D 盘 15.23 GB） |

> 注意：升级 Docker Desktop 时若重建发行版，联接仍有效（因为路径没变），不会退回 C 盘。若某次升级后 C 盘又出现 `wsl` 实体目录，重复上面第 1~3 步即可。

## Phase 4 多 Agent 协同（如实结论 + 落地物）

| 项 | 结果 | 说明 |
|---|---|---|
| Agent RPC / `/agent-call` / cluster 记忆 | ❌ 无法落地 | Trae 无运行时承载，不做 |
| 共享记忆文件 | ✅ 已创建 | `docs/AGENT_SHARED.md`（含使用规则、团队共识、环境事实、分工表四段结构） |
| 代码协同 | ✅ 已有 | 9 实例 = 9 workspace + 同一 GitHub 仓库，走分支 + PR（沿用《GitHub 竞赛团队协作使用手册》） |
| 任务协同 | ✅ 已有 | GitHub Issues / 项目看板 |
| Git 同步验证 | ✅ 已跑通 | 已推送到 `git@github.com:GYJ-RUIBEN/demo-test.git` 的 main 分支（commit `11af9b2`，fast-forward 推送，无强制操作）；team mate `git pull` 即同步 |

## Phase 5 验收结果（2026-11-20，全部重跑实测）

| 验收项 | 结果 | 实证 |
|---|---|---|
| MCP 底座 5 包 discover + 真实调用 | ✅ 5/5 PASS | filesystem（list_directory）、sqlite（SELECT 1）、memory（写读删）、fetch（抓 example.com）、run-python（print(1+1)=2）全部重跑通过 |
| 四层记忆映射 | ✅ | project_memory / topics / session_memory 均在位；user_profile 按需生成 |
| 跨项目隔离 | ✅ | `memory/projects` 下 18 个项目目录，抽读互不相通 |
| Git 扩展 | ✅ 内置可用 | 临时仓库 commit / 分支 / log 通过；`server-git` MCP（PyPI）PASS；**推送已实测**（demo-test main，fast-forward） |
| Docker 扩展可列容器 | ✅ | `docker ps` 正常返回；`docker mcp` v0.44.3 |
| 团队共享记忆 Git 同步 | ✅ 已跑通 | `docs/AGENT_SHARED.md` + 清单已推送到 `demo-test` main（commit `11af9b2`，fast-forward） |
| 密钥明文零写入 | ✅ | 记忆目录正则扫描（sk-/api_key/password/secret/ghp_/AKIA）**零命中** |

### 验收中新发现并已修复的问题

- **uvx 三包一度启动失败**（sqlite / fetch / run-python）：报 `No interpreter found in search path or registry`。根因：`python` 在 PATH 中只剩 Microsoft Store 的占位 stub（真解释器在 `D:\develop\anaconda\python.exe` 但未入 PATH），而 `UV_PYTHON_PREFERENCE=only-system` 禁止 uv 自行下载。
  - 修复：设置用户环境变量 `UV_PYTHON = D:\develop\anaconda\python.exe`；修复后三包全部 PASS。
  - 影响：该变量同时修复了 Trae 启动 MCP 服务器时的解释器查找，重启 Trae 后 5 个包应全部正常连接。
