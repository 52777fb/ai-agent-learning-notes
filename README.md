# AI Agent Concepts: Function Calling, Skills, MCP, Plugins, and Automation

> Date: 2026-05-16  
> Scope: A study note distilled from a discussion about modern LLM agents, tool use, MCP, skills, plugins, web integrations, local program automation, and automation reliability levels.

## 1. Big Picture

Modern LLM agents are not just "chat models". A useful agent usually combines:

- **LLM reasoning**: understands the user goal, plans, chooses actions, summarizes results.
- **Function calling / tool calling**: lets the model request external actions with structured parameters.
- **Tools / APIs / local commands**: actually execute actions outside the model.
- **MCP servers / connectors**: expose external tools, resources, and prompts to AI applications in a standard way.
- **Skills**: reusable ability packages that teach the agent how to perform a class of tasks.
- **Workflows**: explicit task steps, decision points, checks, and loops.
- **Plugins**: larger extension containers that may include skills, MCP servers, apps/connectors, scripts, templates, and assets.

A compact mental model:

```text
User goal
  -> Skill / workflow tells the agent how to do this kind of task
  -> LLM decides the next action
  -> Function calling formats a tool request
  -> MCP / connector / local runtime executes through tools or APIs
  -> Result returns to the LLM
  -> LLM explains, continues, or asks for confirmation
```

## 2. Function Calling

### 2.1 What It Is

Function calling, also called tool calling, is the ability for an LLM to request that an external function or tool be called.

The model itself does not really fetch data, write files, send emails, query databases, or control software. It outputs a structured request such as:

```json
{
  "name": "get_order_status",
  "arguments": {
    "order_id": "8291"
  }
}
```

Then the host application or backend executes the function and returns the result to the model.

### 2.2 Core Flow

```text
User asks a question
  -> LLM decides whether external ability is needed
  -> LLM chooses a function/tool
  -> LLM converts natural language into structured arguments
  -> Backend executes the function/tool
  -> Backend returns result
  -> LLM integrates the result into a user-facing answer
```

Example:

```text
User: Check order 8291.
LLM: call get_order_status({ order_id: "8291" })
Backend: queries order system
Backend result: { status: "shipped", eta: "2026-05-18" }
LLM: "Order 8291 has shipped and is expected on 2026-05-18."
```

### 2.3 What Function Calling Is Not

It is not the model magically gaining the ability itself.

More accurate:

```text
The model has tool-use ability.
The product or host system provides tools.
Function calling is the structured mechanism by which the model requests tool use.
```

### 2.4 Example: Creating a Downloadable TXT File

When an AI product lets the user download a generated `.txt` file, the model usually creates the text content, while the product system creates the actual downloadable file.

Typical flow:

```text
User asks for a txt file
  -> LLM generates text content
  -> Product backend or browser wraps the text as a file object
  -> File is stored temporarily or represented as a Blob
  -> Frontend displays a download button
  -> User downloads the file
```

The conceptual tool may look like:

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

A workflow is simply a process for getting work done.

More precisely:

```text
Workflow = steps + order + conditions + checks + retry/branch logic
```

Example code-fix workflow:

```text
Read issue
  -> inspect relevant files
  -> implement minimal change
  -> run targeted tests
  -> if tests fail, inspect logs and fix
  -> summarize changes
```

Workflow itself is not necessarily technically complex, but it is important because it makes agent behavior more controlled, repeatable, and auditable.

## 4. Skill

### 4.1 Narrow and Broad Meanings

In a narrow product sense, a skill may be a folder with a `SKILL.md` file and optional scripts, templates, references, and assets.

In a broader conceptual sense, a skill can be understood as:

```text
Prompt/instructions + workflow + tools/scripts + reference files/templates + quality standards
```

It is a reusable ability package for a class of tasks.

### 4.2 What Skills Solve

Skills help with:

- Reusing a known process instead of re-explaining it every time.
- Making agent behavior more stable.
- Encoding team or domain standards.
- Packaging supporting materials such as scripts, templates, examples, and assets.

### 4.3 Example Skill Structure

```text
code-review-skill/
  SKILL.md
  scripts/
  templates/
  references/
  assets/
```

Possible `SKILL.md` contents:

```md
# Code Review Skill

Use this skill when the user asks for a code review.

Workflow:
1. Inspect the diff first.
2. Prioritize correctness, security, data loss, and regressions.
3. Avoid style comments unless they affect real behavior.
4. Return findings first, ordered by severity.
5. Include file and line references.
```

### 4.4 Skill vs Function Calling

```text
Skill: tells the agent how to do a kind of task.
Function calling: lets the agent request a concrete external action.
```

Example:

```text
Code Review Skill says:
  "Review PRs by checking correctness/security/regressions first."

Function calling does:
  fetch_pr(repo, number)
  fetch_diff(repo, number)
  create_comment(repo, pr, body)
```

### 4.5 Skill vs Workflow

```text
Workflow = the steps.
Skill = workflow + instructions + tools + templates + references + standards.
```

## 5. MCP

### 5.1 What MCP Is

MCP stands for Model Context Protocol.

It is a standard protocol for connecting AI applications to external tools, resources, and prompt templates.

MCP is not the model itself, not a full agent framework, and not the skill itself.

Better mental model:

```text
MCP = the bridge / socket / standard connector
External system = the actual capability package
LLM = the reasoning system that decides what to use
```

### 5.2 Why MCP Exists

Without a standard like MCP, every AI product must integrate every external service separately.

```text
N AI clients x M external tools = many custom integrations
```

With MCP:

```text
External service implements an MCP server
AI client supports MCP
They communicate through a shared protocol
```

### 5.3 MCP Architecture

```text
AI host application
  -> MCP client
  -> MCP server
  -> real external system or local program
```

Examples of AI hosts:

- Codex
- Claude Desktop
- Cursor
- Other IDEs or AI apps

Examples of external systems:

- GitHub
- Gmail
- Slack
- Google Drive
- Databases
- Local file system
- Blender, Unity, CAD tools

## 6. MCP's Three Main Elements

MCP servers commonly expose three categories:

```text
Tools     = actions the AI can request
Resources = context/data the AI can read
Prompts   = reusable task templates
```

### 6.1 Tools

Tools are executable capabilities.

Examples:

```text
get_pull_request
create_issue_comment
search_email
run_sql_query
create_file
spawn_actor
export_scene
```

Tools may be read-only or may have side effects.

```text
get_order_status = read/query tool
send_email = side-effect tool
delete_file = dangerous side-effect tool
```

### 6.2 Resources

Resources are readable context.

Examples:

```text
github://repo/acme/app/pull/42
file:///project/README.md
postgres://schema/public/orders
docs://handbook/refund-policy
slack://channel/support/thread/123
```

Resources help the model understand context before acting.

```text
Resource = something to read.
Tool = something to do.
```

### 6.3 Prompts

Prompts are reusable task templates exposed by the MCP server.

Examples:

```text
review_pull_request
summarize_issue
draft_release_notes
debug_slow_query
```

They may be lightweight instructions, not necessarily a full skill.

### 6.4 Example: GitHub MCP Server

A GitHub MCP server might expose:

```text
Tools:
  - fetch_pr
  - create_comment
  - create_branch
  - merge_pr

Resources:
  - repository files
  - PR diff
  - issue content
  - CI logs

Prompts:
  - review_pr
  - summarize_issue
  - draft_release_notes
```

## 7. MCP Server, Connector, Integration, Plugin

The same external capability package may be named differently depending on context.

```text
Technical MCP context: GitHub MCP Server
Product context: GitHub connector / GitHub integration
Installable extension context: GitHub plugin
Engineering shorthand: GitHub tool server
```

Important distinction:

```text
MCP server = exposes tools/resources/prompts through MCP.
Connector/integration = product-facing connection to an external service.
Plugin = larger extension container that may include skills, MCP servers, connectors, scripts, assets, and configuration.
```

## 8. Plugin

A plugin is best understood as a higher-level extension container.

It may include:

```text
Plugin
  -> Skills
  -> MCP servers
  -> Apps/connectors
  -> Scripts
  -> Templates
  -> Assets
  -> Permissions/configuration
```

So a plugin is not the same as a skill or MCP server, but it can contain either or both.

Mental model:

```text
Skill = operation manual
MCP server = standardized tool/context interface
Connector = connection to a specific external service
Plugin = packaged extension bundle
```

## 9. Web Service Integrations

For services such as GitHub, Gmail, Google Drive, Slack, Notion, and Linear, the agent usually does not operate the website like a human clicking buttons.

Instead, the normal flow is:

```text
User authorizes the service
  -> Product receives an access token or app installation permission
  -> Backend or connector calls the external service API
  -> MCP server/connector exposes API capabilities as tools/resources/prompts
  -> LLM requests tool calls
  -> Results return to the model
```

Example:

```text
User: Review PR #42.
LLM: calls fetch_pull_request(repo, 42)
GitHub connector: calls GitHub API
GitHub API: returns PR data and diff
LLM: analyzes and summarizes findings
```

### 9.1 Authorization

For private services, authorization is usually done through OAuth or a service-specific app authorization flow.

```text
User connects GitHub
  -> GitHub login/authorization page
  -> user grants selected permissions/repositories
  -> product receives credentials/tokens
  -> later API calls use those credentials
```

The browser does not need to stay open after authorization. The product uses stored credentials or refresh mechanisms.

### 9.2 Public vs Private vs Local

```text
Public web content:
  May be accessed through normal web browsing or public APIs.

Private GitHub/Gmail/Drive content:
  Requires authorization.

Local repository already on disk:
  Can be read directly without GitHub login.
```

## 10. Local Program Integrations

Local programs are different from web services because there may be no remote API.

But local software can still expose local interfaces.

Common connection methods:

```text
Localhost HTTP/WebSocket
Socket or named pipe
Plugin inside the local application
Script API
COM/Automation on Windows
Command-line interface
File exchange
Operating system accessibility APIs
Vision/OCR and mouse/keyboard fallback
```

### 10.1 Does the Program Need to Be Open?

It depends:

```text
Controlling the current running project:
  Usually yes, the program must be open or launched by the agent.

Editing a file directly:
  Not necessarily.

Running batch/headless operations:
  Not necessarily; CLI mode may be enough.
```

### 10.2 Examples

Blender:

```text
LLM
  -> MCP client
  -> Blender MCP server
  -> local socket/WebSocket
  -> Blender plugin
  -> bpy Python API
  -> current Blender scene
```

Excel on Windows:

```text
LLM
  -> local tool
  -> COM Automation
  -> Excel application
  -> workbook modified/saved
```

Unity:

```text
LLM
  -> Unity MCP server
  -> local port/WebSocket
  -> Unity Editor package
  -> Unity Editor API
  -> scene/prefab/project changes
```

## 11. Engineering, Creative, and Entertainment Software

Professional software often has script APIs, SDKs, plugin systems, or command interfaces.

Examples:

```text
Blender: Python API
Unity: C# Editor API
Unreal: Python/Remote Control/C++ plugins
AutoCAD: AutoLISP, .NET API, COM, ObjectARX
Excel/Word/PPT: file libraries, COM, Microsoft Graph
```

This makes them good candidates for agent control through tools or MCP servers.

However, APIs usually do not cover everything.

```text
Core modeling/data operations: often API-accessible
Batch import/export: often API-accessible
Parameters and scripts: often API-accessible
Some wizards/dialogs/plugins: sometimes UI-only
License/login/popups: often UI-only
Old or third-party modules: often UI-only
```

## 12. UI and Vision Automation

If a program has no usable API, no CLI, and no semantic UI interface, an agent may have to operate it visually.

Typical loop:

```text
Take screenshot
  -> OCR or visual recognition
  -> understand current UI state
  -> click/type/drag/press keys
  -> observe result
  -> repeat
```

This is powerful but fragile.

Common problems:

- UI changes break procedures.
- Resolution, scaling, language, and theme affect recognition.
- Popups and loading states interrupt flows.
- It is harder to verify success.
- It may expose private information on screen.
- Mis-clicks are risky.

## 13. Ideal Automation Layers

A useful way to rank automation reliability:

```text
L4: Domain API automation
    Most reliable. Example: GitHub API, Blender Python API, Excel file library.

L3: Command API / UI semantic tree
    Reliable if app exposes commands, controls, selections, and state semantically.

L2: Vision + OCR + mouse/keyboard
    General but fragile. Similar to human screen operation.

L1: Human-assisted automation
    AI suggests or prepares; user confirms or performs key steps.

L0: Not automated or should not be automated
    Sensitive, unsafe, prohibited, or too unreliable.
```

For complex professional applications, the ideal future interface may combine:

```text
1. Domain API
   create_part, run_simulation, export_drawing

2. Command API
   invoke_command(command_id, args)

3. UI semantic tree
   windows, controls, selected objects, panels, fields

4. Vision fallback
   screenshot, OCR, click, drag, type
```

MCP can carry these capabilities, but MCP itself does not define the full domain semantics. The domain or software ecosystem still needs to define what commands, objects, selections, and states mean.

## 14. Future Direction

A likely direction for agent-enabled software:

```text
Software designed for humans only
  -> software with APIs for developers
  -> software with semantic interfaces for agents
  -> software with controlled, auditable automation paths
```

MCP solves the connection layer:

```text
How does the AI app connect to tools/resources/prompts?
```

But complex software still needs higher-level semantic standards:

```text
How does CAD describe faces, sketches, constraints, selected objects, and available commands?
How does a game engine describe scenes, actors, assets, and editor actions?
How does a chat app safely expose selected messages and draft replies?
```

Future systems will likely mix:

- API automation where available.
- Semantic UI/control trees where possible.
- Vision automation as fallback.
- User confirmation for sensitive actions.
- Permissions, logging, and auditing for trust.

## 15. Key Takeaways

```text
Function calling:
  The LLM requests a structured tool call.

Workflow:
  The sequence and logic for doing a task.

Skill:
  A reusable ability package: instructions + workflow + tools/scripts + references + standards.

MCP:
  A standard bridge connecting AI apps to tools, resources, and prompts.

MCP server:
  A service that exposes tools/resources/prompts over MCP.

Connector/integration:
  Product-facing connection to an external service.

Plugin:
  A larger extension bundle that may include skills, MCP servers, connectors, scripts, templates, and assets.

Web integrations:
  Usually use authorization + backend API calls.

Local integrations:
  Use local ports, plugins, scripts, COM, CLI, files, accessibility APIs, or vision fallback.

Automation reliability:
  API > semantic UI > vision/OCR > human-assisted.
```
