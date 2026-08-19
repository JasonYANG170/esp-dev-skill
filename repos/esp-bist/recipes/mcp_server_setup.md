# BIST MCP 服务器：接入 AI 助手（Cursor / VS Code）

> **适用摘要**: 把 ESP-BIST 仓库自带的 MCP 服务器（`mcp-server/server.py`，FastMCP + BM25）接到 Cursor 或 VS Code，让 AI 助手通过 6 个工具（`search_bist_docs`、`get_api_reference`、`get_architecture_info`、`search_kconfig_options`、`search_source_code`、`get_supported_socs`）直接查询仓库的真实文档/头文件/Kconfig/源码，而不是凭记忆猜测；并用 `ingest.py` 在内容变更时重新生成 `data/*.json` 快照（CI 由 `mcp_data_drift` 强制）。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-bist/resources/`, source/examples in `repos/esp-bist/`, and this recipe path `repos/esp-bist/recipes/mcp_server_setup.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "BIST MCP server"
- "Cursor / VS Code 接 BIST"
- "AI 助手查 BIST 文档"
- "search_bist_docs / get_api_reference"
- "ingest.py 重新生成 data"
- "mcp_data_drift CI 报错"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考文件 | `mcp-server/server.py`、`mcp-server/ingest.py`、`mcp-server/search.py`、`mcp-server/requirements.txt` |
| 编辑器配置 | `.cursor/mcp.json`、`.vscode/mcp.json`（仓库已自带，用 `${workspaceFolder}` 占位） |
| Python | Python 3.12（CI 用 `python:3.12-slim`）；建议 venv |
| 依赖 | `rank_bm25>=0.2.2`（其余 `mcp` 由平台预装或随 FastMCP 引入） |
| 范围 | 仅开发辅助工具；不在目标 SoC 运行，不参与安全鉴定（见 `docs/en/tool_qualification.rst` 的排除说明） |

## 分步说明

### 1. 服务器与六个工具（来自 `mcp-server/server.py`）

服务器是 `FastMCP("esp-bist")` 实例，加载 `mcp-server/data/*.json` 预构建快照，由 `search.py` 的 `BISTSearchEngine`（`rank_bm25.BM25Okapi`）做检索。

| 工具 | 签名 | 作用 |
|---|---|---|
| `search_bist_docs` | `(query: str, soc_target: str = "")` | 全文检索 RST 文档与 README（安全需求、架构、验证…），返回排序后的文档块 |
| `get_api_reference` | `(name: str, soc_target: str = "")` | 按函数/类型/模块名查 API：签名、参数、返回值、Doxygen、Kconfig 守卫 |
| `get_architecture_info` | `(topic: str, soc_target: str = "")` | 取架构/模块设计/内存模型/安全文档章节 |
| `search_kconfig_options` | `(query: str)` | 搜 Kconfig 符号：名称、类型、默认值、依赖、help 文本 |
| `search_source_code` | `(query: str, file_type: str = "all")` | 在 `src/bist` 内定位 C 函数实现，带文件路径与行号；`file_type` 可选 `header`/`source`/`all` |
| `get_supported_socs` | `(soc_target: str = "")` | 返回 SoC 支持矩阵（CPU、频率、SRAM、PMA、XT WDT、CSR 数等） |

> 每个工具的 docstring 里有 `When to use` / `How to use` / `Example` 三段，是调用方判断"该用哪个工具"的权威依据。

### 2. 一次性安装（venv）

```bash
cd esp-bist/mcp-server
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt     # 仅 rank_bm25>=0.2.2
```

`requirements.txt` 内容：
```
# Extra pip dependencies (mcp and mcp_platform are pre-installed on the platform)
rank_bm25>=0.2.2
```

### 3. 工作区级接入（仓库自带配置，开箱即用）

仓库根目录已带两份 `${workspaceFolder}` 占位的配置，打开工程即自动注册。

**Cursor — `.cursor/mcp.json`：**
```json
{
  "mcpServers": {
    "ESP-BIST": {
      "command": "${workspaceFolder}/mcp-server/.venv/bin/python3",
      "args": ["${workspaceFolder}/mcp-server/server.py"],
      "env": {}
    }
  }
}
```
验证：Settings → MCP → ESP-BIST，应列出六个工具。

**VS Code (Copilot Chat) — `.vscode/mcp.json`：**
```json
{
  "servers": {
    "ESP-BIST": {
      "type": "stdio",
      "command": "${workspaceFolder}/mcp-server/.venv/bin/python3",
      "args": ["${workspaceFolder}/mcp-server/server.py"],
      "cwd": "${workspaceFolder}/mcp-server",
      "dev": { "watch": "mcp-server/**/*.{py,json}" }
    }
  }
}
```
验证：运行 `MCP: List Servers`，`ESP-BIST` 应为 *running*。`dev.watch` 会在 `mcp-server/` 文件变动时自动重启（便于调试 `ingest.py`；Cursor 不支持此 hint）。

### 4. 用户级（全局）安装

把绝对路径写进用户 profile，工程外也能用。

**Cursor（`~/.cursor/mcp.json`）：**
```json
{
  "mcpServers": {
    "ESP-BIST": {
      "command": "/absolute/path/to/esp-bist/mcp-server/.venv/bin/python3",
      "args": ["/absolute/path/to/esp-bist/mcp-server/server.py"]
    }
  }
}
```

**VS Code（`MCP: Open User Configuration`）：**
```json
{
  "servers": {
    "ESP-BIST": {
      "type": "stdio",
      "command": "/absolute/path/to/esp-bist/mcp-server/.venv/bin/python3",
      "args": ["/absolute/path/to/esp-bist/mcp-server/server.py"]
    }
  }
}
```

### 5. 自托管远程服务器（团队共享）

标准 FastMCP 应用，支持 `stdio` 与 `streamable-http`：

```bash
cd mcp-server
.venv/bin/python ingest.py ..          # 1. 先刷新内容
# 2. 把 server.py 挂到 HTTP transport（见 MCP Python SDK 文档）
# 3. 团队成员在各自编辑器注册托管 URL
```

**Cursor（`~/.cursor/mcp.json`）：**
```json
{
  "mcpServers": {
    "ESP-BIST": { "url": "https://your-host.example.com/mcp", "type": "http" }
  }
}
```

**VS Code：**
```json
{
  "servers": {
    "ESP-BIST": { "type": "http", "url": "https://your-host.example.com/mcp" }
  }
}
```

> 因 ESP-BIST 是开源项目，任何托管部署必须用你自己可控的基础设施；不要依赖无法审计/下线的第三方托管。

### 6. 内容同步：`ingest.py`（来自 `mcp-server/ingest.py`）

`mcp-server/data/*.json` 是仓库内容的预构建 JSON 快照。下列任一变更后必须重新生成：
- `docs/en/*.rst`
- `src/bist/**/*.h`（公开头文件）
- `src/bist/Kconfig`
- `src/bist/**/*.c`
- 任何 `README.md`

```bash
cd mcp-server
.venv/bin/python ingest.py ..
```

`ingest.py` 流水线（5 步，确定性、路径无关）：

| 步骤 | 输出文件 | 解析对象 |
|---|---|---|
| 1. 文档 | `docs.json` | RST 按节标题切块 + README 按 Markdown 标题切块 |
| 2. API | `api.json` | 头文件中的函数签名 + Doxygen + 枚举/typedef |
| 3. Kconfig | `kconfig.json` | `src/bist/Kconfig` 的 config 项（类型/默认/依赖/help） |
| 4. 源码 | `source.json` | `src/bist/**/*.c` 按函数定义切块（含行号） |
| 5. SoC | `socs.json` | 4 个 SoC（C3/C5/C6/H2）的能力矩阵 |

> `_relativize()` 把所有路径字段改写为相对 `repo_root`，故两台机器跑出字节一致的 `data/*.json`——这正是 CI 能强制 drift 检查的前提。生成后把 `data/*.json` 的 diff 连同触发刷新的源码改动一起提交。

### 7. CI 强制（`mcp_data_drift` job）

`.gitlab-ci.yml` 的 `mcp_data_drift`（stage `Lint`）在 MR 触及被索引内容时重跑 `ingest.py`，若提交的快照不匹配则失败：

```yaml
mcp_data_drift:
  stage: Lint
  image: python:3.12-slim
  rules:
    - changes:
        - docs/en/**/*
        - src/bist/**/*.c
        - src/bist/**/*.h
        - src/bist/Kconfig*
        - "**/README.md"
        - mcp-server/ingest.py
        - mcp-server/requirements.txt
  script:
    - pip install -r mcp-server/requirements.txt
    - python mcp-server/ingest.py .
    - git diff --quiet mcp-server/data/ || (git diff --stat mcp-server/data/; exit 1)
```

MR 报错修复：本地跑一次 `python mcp-server/ingest.py .`，提交 `mcp-server/data/*.json` 的 diff。

### 8. 本地调试

直接前台跑服务器看 stdout/stderr：
```bash
cd mcp-server
.venv/bin/python server.py
```

用官方 MCP Inspector 交互式调用工具：
```bash
npx @modelcontextprotocol/inspector .venv/bin/python server.py
```

### 9. 快照覆盖量（来自 `docs/en/mcp_server.rst`）

- 285 个文档块（13 个 RST + 3 个 README）
- 56 个 API 条目（16 个公开头文件）
- 18 个 Kconfig 选项
- 50 个 C 函数体（13 个源文件）
- 4 个 SoC（C3/C5/C6/H2）

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 编辑器里 ESP-BIST 不在服务器列表 | venv 未建或路径不对 | 先 `python3 -m venv .venv && .venv/bin/pip install -r requirements.txt`；工作区配置依赖 `${workspaceFolder}/mcp-server/.venv/bin/python3` |
| 工具返回 "No results found" | `data/*.json` 为空或未生成 | 跑 `.venv/bin/python ingest.py ..` 重新生成 |
| `mcp_data_drift` CI 失败 | 改了文档/头文件/Kconfig 但没重新 ingest | 本地 `python mcp-server/ingest.py .`，提交 `data/*.json` diff |
| BM25 检索质量差 | 未装 `rank_bm25`，回退到子串匹配 | 确认 `requirements.txt` 已装（`rank_bm25>=0.2.2`）；`BM25Okapi` 为 None 时走 `_fallback_scores` |
| 全局安装报 command not found | 用了相对路径 | 用户级配置必须写绝对路径（`/absolute/path/...`） |
| Cursor 不自动重启服务器 | Cursor 不支持 `dev.watch` | 改完 `server.py` 手动重启；VS Code 才支持 watch |
| 误把 MCP 服务器当安全件 | 它是开发辅助，不在 SoC 运行 | `docs/en/tool_qualification.rst` 已将其排除在安全鉴定范围外 |

## 参考

- `docs/en/mcp_server.rst`（六工具说明、安装、CI、覆盖量）
- `mcp-server/server.py`（FastMCP 实例 + 六个 `@mcp.tool()`）
- `mcp-server/ingest.py`（五步确定性流水线 + `_relativize`）
- `mcp-server/search.py`（`BISTSearchEngine`、BM25Okapi、tokenize）
- `mcp-server/requirements.txt`（`rank_bm25>=0.2.2`）
- `.cursor/mcp.json`、`.vscode/mcp.json`（工作区级配置模板）
- `.gitlab-ci.yml` 的 `mcp_data_drift` job（drift 强制）
