# chiangching COMSOL MCP 完整使用指南

本文说明如何从零开始使用 `comsol-mcp`：启动 COMSOL Server、启动 MCP Server、加载并锁定 visible-main 模型、在 COMSOL Desktop 中查看同一个模型、日常调用工具、查看日志，以及正确关闭。

## 1. 核心概念

`comsol-mcp` 采用 **attach-first** 工作流：

1. 用户先手动启动 `COMSOL Multiphysics Server`。
2. MCP 连接到这个已经运行的 Server。
3. MCP 加载并锁定一个主 `.mph` 模型，称为 visible-main model。
4. COMSOL Desktop 也连接到同一个 Server，并查看同一个 server-side model。
5. 后续 MCP 工具修改参数、几何、网格、物理场、求解器和 study 时，Desktop 中可以看到同一模型的变化。

这个项目不是把 COMSOL 当作黑箱 batch runner，而是让自动化过程保持可见。

## 2. 启动前检查

默认路径如下：

```powershell
$Repo = "E:\hycodespace\Forks\Ching-Chiang__comsol-mcp"
$ComsolRoot = "D:\Softwares_02\COMSOL64\Multiphysics"
```

进入仓库：

```powershell
cd E:\hycodespace\Forks\Ching-Chiang__comsol-mcp
```

推荐使用仓库内的 Conda Python：

```powershell
.\.conda\python.exe --version
```

设置运行环境变量：

```powershell
$env:COMSOL_ROOT = "D:\Softwares_02\COMSOL64\Multiphysics"
$env:COMSOL_SERVER_MCP_HOME = "$PWD\comsol-server-home"
```

`COMSOL_SERVER_MCP_HOME` 用于保存状态、日志、输出和快照。默认目录是：

```text
comsol-server-home/
```

该目录已被 `.gitignore` 忽略，适合存放运行产物。

## 3. 启动 COMSOL Multiphysics Server

先手动启动 `COMSOL Multiphysics Server`。启动后，在 Server 控制台中找到真实监听端口，例如：

```text
2036
```

后续 MCP 和 COMSOL Desktop 必须连接到同一个 host 和 port。不要猜端口，以 Server 控制台或系统网络工具显示的真实端口为准。

如果没有 Server 在监听，MCP 在 visible-main 工作流中不会静默启动一个 fallback server，而是会提示你手动启动 Server 并提供端口。

## 4. 启动 MCP Server

推荐使用 console script 启动：

```powershell
cd E:\hycodespace\Forks\Ching-Chiang__comsol-mcp
$env:COMSOL_ROOT = "D:\Softwares_02\COMSOL64\Multiphysics"
$env:COMSOL_SERVER_MCP_HOME = "$PWD\comsol-server-home"
.\.conda\Scripts\comsol-mcp.exe
```

如果该入口不可用，可以用仓库根目录下的薄入口文件：

```powershell
cd E:\hycodespace\Forks\Ching-Chiang__comsol-mcp
$env:COMSOL_ROOT = "D:\Softwares_02\COMSOL64\Multiphysics"
$env:COMSOL_SERVER_MCP_HOME = "$PWD\comsol-server-home"
.\.conda\python.exe mcp_server.py
```

保持这个 MCP Server 进程运行。不要用一次性脚本连接、加载模型后马上退出；这样会销毁 Python/JVM API client，可能让可视化工作流变得不稳定。

### 关于 `python -m comsol_mcp.mcp_server`

当前仓库中 `comsol_mcp/__init__.py` 会导入 `mcp_server`。因此直接运行：

```powershell
.\.conda\python.exe -m comsol_mcp.mcp_server
```

可能出现类似警告：

```text
RuntimeWarning: 'comsol_mcp.mcp_server' found in sys.modules ...
Tool already exists: server_info
```

这表示模块被导入/执行了两次，导致工具重复注册。优先使用 `.\.conda\Scripts\comsol-mcp.exe` 或 `.\.conda\python.exe mcp_server.py`。

## 5. 配置 MCP Host

如果通过 Claude Desktop 或其他 MCP host 使用，可参考如下配置。路径按本机环境写死时更稳定：

```json
{
  "mcpServers": {
    "comsol-mcp-server": {
      "command": "E:/hycodespace/Forks/Ching-Chiang__comsol-mcp/.conda/Scripts/comsol-mcp.exe",
      "args": [],
      "env": {
        "COMSOL_ROOT": "D:/Softwares_02/COMSOL64/Multiphysics",
        "COMSOL_SERVER_MCP_HOME": "E:/hycodespace/Forks/Ching-Chiang__comsol-mcp/comsol-server-home"
      }
    }
  }
}
```

如果 MCP host 以仓库根目录作为工作目录，也可以使用 Python 入口：

```json
{
  "mcpServers": {
    "comsol-mcp-server": {
      "command": "E:/hycodespace/Forks/Ching-Chiang__comsol-mcp/.conda/python.exe",
      "args": ["mcp_server.py"],
      "env": {
        "COMSOL_ROOT": "D:/Softwares_02/COMSOL64/Multiphysics",
        "COMSOL_SERVER_MCP_HOME": "E:/hycodespace/Forks/Ching-Chiang__comsol-mcp/comsol-server-home"
      }
    }
  }
}
```

## 6. 检查端口并加载 visible-main 模型

先检查 COMSOL Server 端口是否可用。假设 Server 在 `127.0.0.1:2036`：

```text
check_server_port("127.0.0.1", 2036)
```

如果返回 port not live 或 connection refused，先回到 COMSOL Server 窗口确认它正在运行，并使用真实端口。

加载并锁定主模型：

```text
start_visible_main_workflow(
  "127.0.0.1",
  2036,
  "E:/hycodespace/Forks/Ching-Chiang__comsol-mcp/comsol-server-home/test-models/first_use_visible_main.mph"
)
```

成功后，MCP 会连接 Server、加载 `.mph`，记录模型的 tag、label、path，并进入 visible-main lock。

大模型加载可能超过 MCP host 的单次工具调用超时，此时使用异步入口：

```text
start_visible_main_workflow_async(
  "127.0.0.1",
  2036,
  "C:/path/to/model.mph"
)
```

拿到 `job_id` 后轮询：

```text
visible_main_workflow_status("<job_id>")
```

## 7. 在 COMSOL Desktop 中查看同一个模型

visible-main 加载成功后：

1. 打开 COMSOL Desktop。
2. 连接到同一个 COMSOL Multiphysics Server。
3. 使用同一个 host 和 port。
4. 导入或切换到 Server 中已经加载的模型。

如果 Desktop 里是空模型，通常说明 Desktop 还没有导入 MCP 已经加载的 server-side model。重新连接同一个 Server，并选择已加载的 visible-main 模型。

## 8. 常用 MCP 工具

查看工作流状态：

```text
workflow_info()
```

验证当前 MCP 是否仍锁定同一个主模型：

```text
verify_visible_main_session()
```

查看模型树：

```text
model_tree()
```

读取参数：

```text
get_parameters()
```

设置参数：

```text
set_parameters("[{\"name\":\"param1\",\"expression\":\"1.0\"}]")
```

计算表达式：

```text
evaluate_expressions("[{\"name\":\"Tmax\",\"expression\":\"T\",\"aggregate\":\"max\"}]", 20000)
```

一次可见迭代：设置参数、求解、提取指标、保存快照：

```text
run_visible_main_iteration(
  "trial_001",
  "[{\"name\":\"param1\",\"expression\":\"1.0\"}]",
  ""
)
```

保存主模型快照：

```text
save_main_model_snapshot("trial_001")
```

运行 study：

```text
run_study("")
```

长时间求解建议异步运行：

```text
run_study_async("")
run_study_status("<job_id>")
```

## 9. visible-main 安全边界

`load_visible_main_model()` 或 `start_visible_main_workflow()` 成功后，MCP 会记录当前主模型身份：

- tag
- label
- path

在 strict guard 下，工具分为三类：

| 类别 | 行为 |
|---|---|
| SAFE_READ | 始终允许，例如 `workflow_info`、`model_tree`、`get_parameters`、`evaluate_expressions` |
| SAFE_VISIBLE_MAIN_WRITE | 仅当当前模型身份匹配时允许，例如 `set_parameters`、`ensure_mesh`、`run_study`、`save_main_model_snapshot` |
| RESTRICTED | 锁定时阻止，例如 `model_create`、`model_load`、`prune_loaded_models`、`commit_current_main_model` |

这个机制用于避免 Desktop 正在查看的主模型被意外切换、覆盖或清理。

只有维护场景才考虑解锁：

```text
unlock_visible_main("maintenance reason")
```

解锁后要格外小心，因为 restricted tools 可能改变当前模型身份。

## 10. 日志、状态和输出

常用运行文件：

```text
comsol-server-home/status.json
comsol-server-home/workflow_state.json
comsol-server-home/logs/server.log
comsol-server-home/logs/operations.jsonl
comsol-server-home/outputs/
```

建议健康检查或人工排错日志放在：

```text
comsol-server-home/logs/pipeline-checks/<YYYYMMDD_HHMMSS>/
```

这些目录默认不进入 git，适合保存运行证据、状态快照、导出图片、表格和求解记录。

## 11. 正确关闭

### 关闭 MCP Server

在运行 MCP Server 的终端中按：

```text
Ctrl+C
```

看到 PowerShell prompt 返回即可：

```powershell
PS E:\hycodespace\Forks\Ching-Chiang__comsol-mcp>
```

如果退出时看到：

```text
KeyboardInterrupt
asyncio.exceptions.CancelledError
```

这通常只是 `Ctrl+C` 取消 stdio MCP 服务时的调用栈，可以视为正常关闭信号。

### 关闭 COMSOL Server 和 Desktop

MCP Server 退出不会自动关闭你手动启动的 `COMSOL Multiphysics Server`。需要在 COMSOL Server 自己的窗口或控制台里单独关闭。

COMSOL Desktop 也是独立客户端，需要单独关闭或断开连接。

## 12. 常见错误速查

### `ModuleNotFoundError: No module named 'mcp'`

当前 Python 环境没有安装依赖，或用错了解释器。优先使用：

```powershell
.\.conda\Scripts\comsol-mcp.exe
```

或：

```powershell
.\.conda\python.exe mcp_server.py
```

### `Tool already exists: server_info`

通常是 `python -m comsol_mcp.mcp_server` 触发模块重复导入。改用：

```powershell
.\.conda\Scripts\comsol-mcp.exe
```

或：

```powershell
.\.conda\python.exe mcp_server.py
```

### `Connection refused`

目标端口没有 COMSOL Server 监听。检查：

```text
check_server_port("127.0.0.1", 2036)
```

然后回到 COMSOL Server 控制台确认真实端口。

### `Not connected to a COMSOL Multiphysics Server`

MCP 当前未连接 COMSOL Server。先启动 COMSOL Server，确认端口，再运行 visible-main workflow。

### Desktop 里看不到模型

MCP 已经在 Server 端加载模型，但 Desktop 还没有导入同一个 server-side model。让 Desktop 连接同一个 Server 和端口，并选择已加载模型。

### 快照改变了 Desktop 模型名

优先使用：

```text
save_main_model_snapshot("label")
run_visible_main_iteration("label", "[]", "")
```

这些工具使用 save-copy 语义，目标是保存快照但不改变当前主模型路径。

## 13. 最小推荐流程

```powershell
cd E:\hycodespace\Forks\Ching-Chiang__comsol-mcp
$env:COMSOL_ROOT = "D:\Softwares_02\COMSOL64\Multiphysics"
$env:COMSOL_SERVER_MCP_HOME = "$PWD\comsol-server-home"
.\.conda\Scripts\comsol-mcp.exe
```

然后在 MCP host 中：

```text
check_server_port("127.0.0.1", 2036)
start_visible_main_workflow("127.0.0.1", 2036, "C:/path/to/model.mph")
verify_visible_main_session()
workflow_info()
```

之后打开 COMSOL Desktop，连接同一个 Server，并导入已加载模型。
