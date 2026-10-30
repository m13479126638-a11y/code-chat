# EPIP 插件构建与接入标准 v1.1（正式版）

适用版本：AI-Agent-Chat v26.11、local_bridge_server v26.11。
本标准自 v26.11 起取代 v1.0 及之前的所有草案。

## 修订摘要（v1.0 → v1.1）

| # | 变更 | 性质 |
|---|---|---|
| 1 | plugin.json 新增 `hidden_from_agent` 字段 | 新增 |
| 2 | plugin.json 新增 `service` 字段（服务插件声明） | 新增 |
| 3 | 新增第 12 章"服务插件"完整条款 | 新增 |
| 4 | 第 4 章清单字段表补充两个新字段 | 修订 |
| 5 | 第 8 章调用协议补充 hidden 插件行为 | 修订 |
| 6 | 第 11 章安全清单补充服务插件条目 | 修订 |

参考实现：ai_ork.epip（AI 编排端口 v1.0.0）。

---

# 第一部分：通用插件

## 1. 插件是什么

EPIP 插件是一个以 `.epip` 为扩展名的 ZIP 包，用于向 Agent 主 AI 提供任务相关提示词，并在主 AI 显式请求时执行包内 Python 入口。

插件包放在桥接服务根目录下：

```text
ai-agent/plugin/*.epip
```

桥接刷新时会完成扫描、清单校验和安全解包，但不会运行插件。勾选启用也只会保存启用状态并下发提示词，仍不会运行插件。只有主 AI 调用 `action:"run"` 时才执行。

## 2. 运行与目录结构

```text
<桥接根目录>/
├─ ai-agent/
│  ├─ plugin/
│  │  └─ echo_demo.epip             # 作者交付的原始 ZIP 包
│  └─ plugin_runtime/
│     └─ echo_demo/                 # 桥接安全解包后的运行文件
└─ bridge_workspace/
   └─ plugins/
      └─ echo_demo/
         └─ workspace/              # 每次执行的默认工作目录
            └─ tmp/
```

每次执行都会启动独立 Python 子进程，使用独立工作目录、最小环境变量和单次 RPC 令牌。RPC 服务只绑定 `127.0.0.1`，进程结束后关闭，令牌随之失效。

进程级隔离不是操作系统容器，也不代表插件绝对安全。只安装来源可信的插件。

## 3. 最小包结构

ZIP 根目录必须直接包含以下文件，不要再套一层文件夹：

```text
plugin.json
prompt.md
main.py
```

`epip_client.py` 和 `epip_runner.py` 是桥接保留文件名，插件入口不能使用这两个名字。

## 4. plugin.json 清单

```json
{
  "epip_version": 1,
  "id": "echo_demo",
  "name": "回声与 RPC 示例",
  "version": "1.0.0",
  "description": "演示参数、helper 与桥接 RPC。",
  "entry": "main.py",
  "prompt": "prompt.md",
  "timeout": 60
}
```

字段要求：

| 字段 | 必填 | 规则 |
|---|---|---|
| `epip_version` | 是 | 当前固定为 `1`。 |
| `id` | 是 | 1～64 位，以字母或数字开头，仅含字母、数字、下划线、连字符。同一目录中 ID 必须唯一。 |
| `name` | 否 | 界面显示名，最长 80 字。 |
| `version` | 否 | 版本字符串，最长 32 字；建议使用语义化版本。 |
| `description` | 否 | 插件说明，最长 500 字。 |
| `entry` | 是 | 包内真实存在的 `.py` 文件，默认 `main.py`。 |
| `prompt` | 是 | 包内真实存在的提示词文本，默认 `prompt.md`，最大 64KB。 |
| `timeout` | 否 | 默认执行超时秒数，桥接限制为 1～300 秒，默认 60 秒。 |
| `hidden_from_agent` | 否 | `true` 时插件对 Agent 隐藏：提示词不下发、AI 不可 `run`、AI 的 `list` 过滤之；前端插件菜单仍可见。仅服务插件应使用。详见第 12 章。 |
| `service` | 否 | 服务插件声明：`{"mount": "前缀", "desc": "说明"}`。启用后桥接挂载 `/api/<前缀>/*` 路由。详见第 12 章。 |

所有包内路径必须使用 `/`，禁止绝对路径、盘符、反斜杠、空路径段、`.`、`..`、重复路径与符号链接。

## 5. prompt.md 编写标准

提示词只注入 Agent 和 Agent-2 的主 AI，不注入普通 Chat、Agent 工作 AI 或 Agent-2 其他成员。（声明 `hidden_from_agent` 的插件连这一步也跳过——提示词完全不下发给任何 AI。）

推荐写清四点：任务场景、插件 ID 与 run 调用示例、params 字段说明、返回结果含义。

提示词不能覆盖系统规则、用户要求或桥接安全策略。不要存放秘密。

## 6. Python 入口标准

```python
def run(params):
    return {"result": "value"}
```

- 输入 JSON 最大 256KB；返回值应可 JSON 序列化。
- `print()` 进 `output`；异常进 `output` 且 `success=false`。
- 不要启动常驻服务、无限循环或后台进程。

## 7. 桥接通讯接口

```python
from epip_client import bridge
py_result = bridge.python("print('hello')", timeout=60)
```

禁止直接导入或调用 `subprocess`、`ctypes`、`winreg`、`multiprocessing`、`os.system`、`os.popen`、`os.startfile`、`os.spawn*`、`os.exec*`、`pty.spawn`、`asyncio.create_subprocess_*`，以及通过 `__import__`/`importlib.import_module` 动态加载上述模块。`getattr` 动态解析、`eval`/`exec` 同样禁止（桥接静态审查与 SecurityGate 双重拦截）。

## 8. 主 AI 调用协议

| `action` | 用途 | 是否执行插件 |
|---|---|---|
| `list` | 读取当前扫描缓存（结果过滤 hidden 插件） | 否 |
| `refresh` | 重新识别目录、校验并解包（同上过滤） | 否 |
| `info` | 查看指定插件的清单、状态和提示词 | 否 |
| `run` | 执行已启用插件的 `run(params)`（hidden 插件返回"对 Agent 隐藏，不可被 AI 调用"，fail-closed） | 是 |

## 9. 包与运行限制

| 项目 | 限制 |
|---|---:|
| `.epip` 包大小 | 最大 16MB |
| 解压后总大小 | 最大 32MB |
| 单个包内文件 | 最大 8MB |
| 包内文件数 | 最多 128 个 |
| 提示词 | 最大 64KB |
| 包内 Python 源码合计 | 最大 1MB |
| 单次输入 `params` | 最大 256KB |
| 单次执行输出 | 最多保留 30000 字 |
| 默认执行超时 | 60 秒，可配置 1～300 秒 |

## 10. 打包、安装与调试

```bash
python -m zipfile -c echo_demo.epip plugin.json prompt.md main.py
```

安装：复制到 `ai-agent/plugin/` → 启动桥接并握手 → 插件菜单刷新 → 核对并勾选启用 → 在 Agent 或 Agent-2 主 AI 中发起相关任务。

## 11. 安全发布清单

- 只在可信来源之间分发，并公布版本与 SHA-256。
- 不包含秘密、令牌、用户数据或无关大文件。
- 不利用 ZIP 路径、符号链接或重复 ID 覆盖其他插件。
- 不直接创建进程，所有终端需求都走 `epip_client.bridge`。
- 不在导入模块时产生副作用。
- 任务能在声明的超时内结束，不启动常驻进程。
- 声明 `service` 的插件：其 HTTP 路由必须自带独立鉴权（如一次性 token），权限面不得触及桥接管理端点与用户既有数据；服务路由由桥接原生代码代管，不属于插件 Python 审查范围，但路由实现必须 fail-closed。

---

# 第二部分：服务插件（v1.1 新增）

## 12. 服务插件

### 12.1 定义

服务插件是一类**对 Agent 隐藏、由桥接代管 HTTP 服务**的特权插件。它不满足"主 AI 调用 run()"的使用模型，而是：

- 安装并勾选启用后，桥接在自身进程内挂载一组 HTTP 路由（`/api/<mount>/*`）
- 路由由**桥接原生代码**实现，不属于插件 Python 源码审查范围（插件包的 `main.py` 照常走扫描，仅作占位/说明）
- 路由自带**独立鉴权**（服务 token），与桥接主 token 隔离
- 未安装或未启用时，相关路由一律 404——服务插件不启用即等同死代码

### 12.2 声明方式

```json
{
  "id": "ai_ork",
  "hidden_from_agent": true,
  "service": { "mount": "aiork", "desc": "AI 编排端口：Chat 会话控制接口" }
}
```

| 字段 | 规则 |
|---|---|
| `hidden_from_agent` | 必须为 `true`。服务插件对 Agent 隐身：提示词不下发、AI 不可 `run`、AI 的 `list`/`refresh` 结果过滤之。前端插件菜单照常显示（带"服务"徽标），由 user 决定是否启用。 |
| `service.mount` | 路由前缀，仅含字母数字连字符，1～32 位；最终路由为 `/api/<mount>/*`。禁止与桥接既有路由冲突（bridge、plugins、agent-data、chats、uploads、task 等保留前缀不可用）。 |
| `service.desc` | 服务说明，最长 200 字，展示于插件菜单徽标提示。 |

### 12.3 行为规则

1. **挂载时机**：插件扫描 valid 且勾选启用时挂载；停用、删除或校验失败时立即卸载。不重启也生效。
2. **路由鉴权**：服务路由不得使用桥接主 token。推荐模式：主前端用桥接 token 调 `/api/<mount>/register`（或同类端点）换取**服务 token**（如 ork_token），外部调用方只持服务 token。
3. **权限边界**：服务 token 的权限面必须收敛——只能操作服务插件自建并标记的数据（如 `_ork:true` 会话），对用户手建数据默认只读。
4. **无执行能力**：服务插件面向的场景若为"对话编排"（如 AI 编排端口），则不得提供代码/终端/工具执行类指令。是否允许执行类能力由具体服务规范定义，默认禁止。
5. **fail-closed**：服务 token 校验失败、额度耗尽、目标数据不存在或权限不足时一律拒绝，不得有默认放行路径。
6. **常驻服务的澄清**：服务插件不违反"禁止常驻服务"条款——HTTP 服务由桥接进程代管，插件本身无常驻进程；插件包 `main.py` 依旧禁止启动常驻进程/无限循环。

### 12.4 参考实现：ai_ork（AI 编排端口）

首个服务插件：

- 端口：`/api/aiork/register`（桥接主 token，换取 ork_token）、`/api/aiork/exec`（ork_token，统一指令端口）
- 指令面：会话 CRUD、消息流读写、LLM 对话调用（写回会话）、额度管理、token 轮换
- 架构红线：Chat 模式无工具能力——端口不含任何执行类指令；ork_token 只能写 `_ork:true` 会话
- 死代码原则：未安装/未启用 ai_ork 时，路由 404，前端相关面板隐藏

### 12.5 服务插件审核要求

- 服务插件的**包内容**（plugin.json/prompt.md/main.py）与普通插件同等审查。
- 服务插件的**路由实现**属于桥接代码变更，随桥接版本审计；新增服务插件必须随桥接发版，不能仅靠投放 `.epip` 包获得新路由。
- 分发服务插件时必须同时公布：包 SHA-256、配套桥接最低版本、服务 token 获取方式、权限边界说明。
