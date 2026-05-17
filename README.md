# AI 智能体概念笔记：Function Calling、Skill、MCP、插件与自动化

> 日期：2026-05-16  
> 范围：这是一份从讨论中整理出的学习笔记，覆盖现代 LLM 智能体、工具调用、MCP、Skill、插件、网页服务集成、本地程序自动化，以及自动化可靠性层级。

## 1. 总览

现代 LLM 智能体不只是“聊天模型”。一个可用的智能体通常由这些部分组合而成：

- **LLM 推理能力**：理解用户目标、规划步骤、选择动作、总结结果。
- **Function Calling / Tool Calling**：让模型用结构化参数请求外部工具执行动作。
- **工具 / API / 本地命令**：真正执行模型本身做不了的事情。
- **MCP Server / Connector**：用标准方式把外部工具、资源、提示模板暴露给 AI 应用。
- **Skill**：可复用的能力包，告诉智能体某类任务应该如何专业地完成。
- **Workflow**：明确的步骤、顺序、条件分支、检查点和重试逻辑。
- **Plugin**：更大的扩展容器，可以包含 Skill、MCP Server、Connector、脚本、模板和资源文件。

一个简化心智模型：

```text
用户目标
  -> Skill / Workflow 告诉智能体这类任务该怎么做
  -> LLM 判断下一步动作
  -> Function Calling 把动作格式化为工具请求
  -> MCP / Connector / 本地运行环境执行工具或 API
  -> 结果返回给 LLM
  -> LLM 解释结果、继续执行，或请求用户确认
```

## 2. Function Calling

### 2.1 它是什么

Function Calling，也常叫 Tool Calling，是 LLM 请求调用外部函数或工具的能力。

模型本身并不会真的查数据、写文件、发邮件、查询数据库或控制软件。它输出的是一个结构化调用请求，例如：

```json
{
  "name": "get_order_status",
  "arguments": {
    "order_id": "8291"
  }
}
```

然后由宿主应用或后端真正执行这个函数，并把结果返回给模型。

### 2.2 核心流程

```text
用户提出问题
  -> LLM 判断是否需要外部能力
  -> LLM 选择函数或工具
  -> LLM 把自然语言转换成结构化参数
  -> 后端执行函数或工具
  -> 后端返回结果
  -> LLM 把结果整合成面向用户的回答
```

例子：

```text
用户：查一下订单 8291。
LLM：调用 get_order_status({ order_id: "8291" })
后端：查询订单系统
后端结果：{ status: "shipped", eta: "2026-05-18" }
LLM：订单 8291 已发货，预计 2026-05-18 送达。
```

### 2.3 它不是什么

Function Calling 不是模型“神奇地获得了某种能力”。

更准确的说法是：

```text
模型具备工具使用能力。
产品或宿主系统提供工具。
Function Calling 是模型请求使用工具的结构化机制。
```

### 2.4 例子：生成可下载的 TXT 文件

当 AI 产品让用户下载一个生成的 `.txt` 文件时，通常是模型生成文本内容，产品系统负责把文本包装成真正可下载的文件。

典型流程：

```text
用户要求生成 txt 文件
  -> LLM 生成文本内容
  -> 产品后端或浏览器把文本包装成文件对象
  -> 文件被临时存储，或在浏览器中表示为 Blob
  -> 前端显示下载按钮
  -> 用户下载文件
```

概念上的工具可能像这样：

```json
{
  "name": "create_downloadable_file",
  "arguments": {
    "filename": "notes.txt",
    "mime_type": "text/plain",
    "content": "..."
  }
}
```

## 3. Workflow

Workflow 可以简单理解为“干事的流程”。

更准确地说：

```text
Workflow = 步骤 + 顺序 + 条件 + 检查点 + 重试/分支逻辑
```

一个代码修复流程例子：

```text
读取 issue
  -> 检查相关文件
  -> 实现最小改动
  -> 运行针对性测试
  -> 如果测试失败，查看日志并修复
  -> 总结改动
```

Workflow 本身不一定有多高技术含量，但它很重要，因为它能让智能体的行为更可控、更可复现、更容易审计。

## 4. Skill

### 4.1 狭义和广义理解

在狭义产品语境里，Skill 可能是一个包含 `SKILL.md` 的文件夹，并可附带脚本、模板、参考资料和资源文件。

在更广义的理解里，Skill 可以看成：

```text
提示词/说明 + 工作流 + 工具/脚本 + 参考文件/模板 + 质量标准
```

它是面向某一类任务的可复用能力包。

### 4.2 Skill 解决什么问题

Skill 的价值包括：

- 复用已有流程，不用每次重新解释。
- 让智能体表现更稳定。
- 沉淀团队或领域规范。
- 打包脚本、模板、示例、参考资料等辅助材料。

### 4.3 Skill 结构示例

```text
code-review-skill/
  SKILL.md
  scripts/
  templates/
  references/
  assets/
```

一个可能的 `SKILL.md`：

```md
# 代码审查 Skill

当用户要求代码审查时使用本 Skill。

流程：
1. 先检查 diff。
2. 优先关注正确性、安全性、数据丢失和回归风险。
3. 除非风格问题会影响真实行为，否则不要纠结风格。
4. 先输出 findings，并按严重程度排序。
5. 给出文件和行号引用。
```

### 4.4 Skill 和 Function Calling 的区别

```text
Skill：告诉智能体某类任务应该怎么做。
Function Calling：让智能体请求一个具体外部动作。
```

例子：

```text
代码审查 Skill 规定：
  “审查 PR 时优先看 correctness/security/regression。”

Function Calling 执行：
  fetch_pr(repo, number)
  fetch_diff(repo, number)
  create_comment(repo, pr, body)
```

### 4.5 Skill 和 Workflow 的区别

```text
Workflow = 任务步骤。
Skill = Workflow + 说明 + 工具 + 模板 + 参考资料 + 标准。
```

## 5. MCP

### 5.1 MCP 是什么

MCP 全称 Model Context Protocol。

它是一套标准协议，用来把 AI 应用连接到外部工具、资源和提示模板。

MCP 不是模型本体，不是完整智能体框架，也不是 Skill 本身。

更好的心智模型：

```text
MCP = 桥 / 插口 / 标准连接方式
外部系统 = 真正的能力包
LLM = 决定该用什么能力的推理系统
```

### 5.2 为什么需要 MCP

如果没有 MCP 这样的标准，每个 AI 产品都要单独适配每个外部服务。

```text
N 个 AI 客户端 x M 个外部工具 = 大量重复集成
```

有了 MCP：

```text
外部服务实现 MCP Server
AI 客户端支持 MCP
双方按同一协议通信
```

### 5.3 MCP 架构

```text
AI 宿主应用
  -> MCP Client
  -> MCP Server
  -> 真实外部系统或本地程序
```

AI 宿主应用例子：

- Codex
- Claude Desktop
- Cursor
- 其他 IDE 或 AI 应用

外部系统例子：

- GitHub
- Gmail
- Slack
- Google Drive
- 数据库
- 本地文件系统
- Blender、Unity、CAD 工具

## 6. MCP 的三大要素

MCP Server 通常暴露三类东西：

```text
Tools     = AI 可以请求执行的动作
Resources = AI 可以读取的上下文/数据
Prompts   = 可复用的任务提示模板
```

### 6.1 Tools

Tools 是可执行能力。

例子：

```text
get_pull_request
create_issue_comment
search_email
run_sql_query
create_file
spawn_actor
export_scene
```

Tool 可以是只读的，也可以有副作用。

```text
get_order_status = 只读/查询工具
send_email = 有副作用的工具
delete_file = 高风险副作用工具
```

### 6.2 Resources

Resources 是可读取上下文。

例子：

```text
github://repo/acme/app/pull/42
file:///project/README.md
postgres://schema/public/orders
docs://handbook/refund-policy
slack://channel/support/thread/123
```

Resources 帮助模型在行动前理解背景。

```text
Resource = 给模型读的东西。
Tool = 让系统做的事情。
```

### 6.3 Prompts

Prompts 是 MCP Server 暴露的可复用任务模板。

例子：

```text
review_pull_request
summarize_issue
draft_release_notes
debug_slow_query
```

它们可能只是轻量提示，不一定是完整 Skill。

### 6.4 GitHub MCP Server 例子

一个 GitHub MCP Server 可能暴露：

```text
Tools:
  - fetch_pr
  - create_comment
  - create_branch
  - merge_pr

Resources:
  - 仓库文件
  - PR diff
  - issue 内容
  - CI 日志

Prompts:
  - review_pr
  - summarize_issue
  - draft_release_notes
```

## 7. MCP Server、Connector、Integration、Plugin

同一个外部能力包，在不同语境下可能有不同叫法。

```text
MCP 技术语境：GitHub MCP Server
产品功能语境：GitHub connector / GitHub integration
可安装扩展语境：GitHub plugin
工程简称：GitHub tool server
```

重要区别：

```text
MCP Server = 通过 MCP 暴露 tools/resources/prompts 的服务。
Connector / Integration = 产品层面对某个外部服务的连接。
Plugin = 更大的扩展容器，可以包含 Skill、MCP Server、Connector、脚本、资源和配置。
```

## 8. Plugin

Plugin 更适合理解为一个上层扩展容器。

它可能包含：

```text
Plugin
  -> Skills
  -> MCP Servers
  -> Apps / Connectors
  -> Scripts
  -> Templates
  -> Assets
  -> Permissions / Configuration
```

所以 Plugin 不等于 Skill，也不等于 MCP Server，但它可以包含二者之一或二者都有。

心智模型：

```text
Skill = 操作手册
MCP Server = 标准化工具/上下文接口
Connector = 连接某个外部服务的通道
Plugin = 打包好的扩展包
```

## 9. 网页服务集成

对于 GitHub、Gmail、Google Drive、Slack、Notion、Linear 这类服务，智能体通常不是像人一样打开网页点按钮。

更常见的流程是：

```text
用户授权服务
  -> 产品获得 access token 或 app installation 权限
  -> 后端或 connector 调用外部服务 API
  -> MCP Server / Connector 把 API 能力暴露成 tools/resources/prompts
  -> LLM 请求工具调用
  -> 结果返回给模型
```

例子：

```text
用户：审查 PR #42。
LLM：调用 fetch_pull_request(repo, 42)
GitHub connector：调用 GitHub API
GitHub API：返回 PR 数据和 diff
LLM：分析并总结问题
```

### 9.1 授权

对于私有服务，授权通常通过 OAuth 或服务专属 App 授权流程完成。

```text
用户连接 GitHub
  -> GitHub 登录/授权页面
  -> 用户授予选定权限/仓库
  -> 产品获得凭据/token
  -> 之后 API 调用使用这些凭据
```

授权完成后，浏览器不需要一直开着。产品会使用已保存的凭据或刷新机制。

### 9.2 公开、私有、本地的区别

```text
公开网页内容：
  可以通过普通网页访问或公开 API 读取。

私有 GitHub/Gmail/Drive 内容：
  需要授权。

已经在本地磁盘上的仓库：
  可以直接读取本地文件，不需要 GitHub 登录。
```

## 10. 本地程序集成

本地程序不同于网页服务，因为它们不一定有远程 API。

但本地软件仍然可以暴露本地接口。

常见连接方式：

```text
localhost HTTP / WebSocket
Socket 或 named pipe
本地程序内插件
脚本 API
Windows COM / Automation
命令行接口
文件交换
操作系统无障碍 API
视觉/OCR + 鼠标键盘兜底
```

### 10.1 程序是否需要打开

取决于任务类型：

```text
控制当前运行中的项目：
  通常需要程序已经打开，或由 agent 启动它。

直接编辑文件：
  不一定需要打开原程序。

运行批处理/headless 操作：
  不一定需要 GUI，命令行模式可能就够。
```

### 10.2 例子

Blender：

```text
LLM
  -> MCP Client
  -> Blender MCP Server
  -> 本地 socket / WebSocket
  -> Blender 插件
  -> bpy Python API
  -> 当前 Blender 场景
```

Windows 上的 Excel：

```text
LLM
  -> 本地工具
  -> COM Automation
  -> Excel 应用
  -> 工作簿被修改/保存
```

Unity：

```text
LLM
  -> Unity MCP Server
  -> 本地端口 / WebSocket
  -> Unity Editor Package
  -> Unity Editor API
  -> 场景 / Prefab / 项目设置变化
```

## 11. 工程、创作和娱乐软件

专业软件通常有脚本 API、SDK、插件系统或命令接口。

例子：

```text
Blender: Python API
Unity: C# Editor API
Unreal: Python / Remote Control / C++ 插件
AutoCAD: AutoLISP、.NET API、COM、ObjectARX
Excel/Word/PPT: 文件库、COM、Microsoft Graph
```

这些软件很适合通过工具或 MCP Server 被 agent 控制。

但 API 通常无法覆盖所有功能。

```text
核心建模/数据操作：通常可通过 API 访问
批量导入导出：通常可通过 API 访问
参数和脚本：通常可通过 API 访问
某些向导/对话框/插件：可能只有 UI
许可证/登录/弹窗：通常只有 UI
老旧模块或第三方模块：可能只有 UI
```

## 12. UI 与视觉自动化

如果一个程序没有可用 API、没有 CLI、也没有语义化 UI 接口，agent 可能只能通过视觉方式操作它。

典型循环：

```text
截图
  -> OCR 或视觉识别
  -> 理解当前 UI 状态
  -> 点击 / 输入 / 拖拽 / 按键
  -> 观察结果
  -> 重复
```

这种方式通用，但比较脆弱。

常见问题：

- UI 改版会破坏流程。
- 分辨率、缩放、语言、主题会影响识别。
- 弹窗和加载状态会打断流程。
- 很难确认操作是否真的成功。
- 屏幕上可能暴露隐私信息。
- 误点风险较高。

## 13. 理想自动化层级

一种有用的自动化可靠性分层：

```text
L4：领域 API 自动化
    最可靠。例如 GitHub API、Blender Python API、Excel 文件库。

L3：命令 API / UI 语义树
    如果应用能以语义方式暴露命令、控件、选区和状态，就比较可靠。

L2：视觉 + OCR + 鼠标键盘
    通用但脆弱，类似人类看屏幕操作。

L1：人工辅助自动化
    AI 提建议或准备内容，用户确认或执行关键步骤。

L0：不可自动化或不应自动化
    敏感、不安全、被禁止或过于不可靠的场景。
```

对于复杂专业软件，理想的未来接口可能会混合：

```text
1. 领域 API
   create_part, run_simulation, export_drawing

2. 命令 API
   invoke_command(command_id, args)

3. UI 语义树
   windows, controls, selected objects, panels, fields

4. 视觉兜底
   screenshot, OCR, click, drag, type
```

MCP 可以承载这些能力，但 MCP 本身不定义完整领域语义。具体领域或软件生态仍然需要定义命令、对象、选区和状态到底是什么意思。

## 14. 未来方向

Agent 化软件的一个可能方向：

```text
只为人类设计的软件
  -> 为开发者提供 API 的软件
  -> 为 agent 提供语义接口的软件
  -> 提供可控、可审计自动化路径的软件
```

MCP 解决的是连接层问题：

```text
AI 应用如何连接 tools/resources/prompts？
```

但复杂软件仍然需要更高层的语义标准：

```text
CAD 如何描述面、草图、约束、选中对象和可用命令？
游戏引擎如何描述场景、Actor、资源和编辑器动作？
聊天应用如何安全暴露选中的消息和回复草稿？
```

未来系统很可能混合使用：

- 有 API 时优先 API 自动化。
- 可以使用语义化 UI/控件树时优先使用它。
- 没有更好接口时用视觉自动化兜底。
- 敏感动作需要用户确认。
- 通过权限、日志和审计建立信任。

## 15. 关键总结

```text
Function Calling：
  LLM 请求一个结构化工具调用。

Workflow：
  完成任务的步骤和逻辑。

Skill：
  可复用能力包：说明 + 工作流 + 工具/脚本 + 参考资料 + 标准。

MCP：
  连接 AI 应用与工具、资源、提示模板的标准桥梁。

MCP Server：
  通过 MCP 暴露 tools/resources/prompts 的服务。

Connector / Integration：
  产品层面对某个外部服务的连接。

Plugin：
  更大的扩展包，可以包含 Skill、MCP Server、Connector、脚本、模板和资源。

网页服务集成：
  通常通过授权 + 后端 API 调用实现。

本地程序集成：
  使用本地端口、插件、脚本、COM、CLI、文件、无障碍 API 或视觉兜底。

自动化可靠性：
  API > 语义 UI > 视觉/OCR > 人工辅助。
```
