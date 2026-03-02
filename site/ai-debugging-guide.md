# AI Debugging Guide for MCP AI Server (vs-mcp)

## Overview

MCP AI Server exposes 19 debugging tools that let AI assistants interact with the Visual Studio debugger. The debugger is **state-based** — you must check the current mode before calling tools, and use **polling** to detect state changes.

There are two fundamental ways to start a debug session:
1. **`debug_start`** — Press F5, launches the startup project with debugger attached
2. **`debug_attach`** — Attach to an already-running process (local, Docker, or WSL)

In both cases, once in Break mode, all inspection tools work identically.

## Debugger Modes

| Mode | Meaning | How you get here |
|------|---------|-----------------|
| **Design** | No debug session active | Default state, or after debug_stop / program exits |
| **Running** | App is executing | After debug_start, debug_attach, or debug_continue |
| **Break** | Paused at breakpoint/exception | Breakpoint hit, debug_break, or debug_step completed |

## Available Tools (19)

### Debug Control (10 tools)

| Tool | Required Mode | Description |
|------|--------------|-------------|
| `debug_get_mode` | Any | Returns current mode: Design, Running, or Break |
| `debug_start` | Design | Start debugging (F5). Fire-and-forget — returns immediately |
| `debug_stop` | Running/Break | Stop debugging session (detach + terminate) |
| `debug_break` | Running | Pause the running application |
| `debug_continue` | Break | Resume execution |
| `debug_step` | Break | Step over/into/out. Direction: `"over"`, `"into"`, `"out"` |
| `immediate_execute` | Break | Execute expression with side effects (e.g. `myVar = 42`) |
| `debug_list_transports` | Any | List available transports (Default, Docker, WSL, etc.) |
| `debug_list_processes` | Any | List processes on a transport for attachment |
| `debug_attach` | Design | Attach debugger to a running process by name or PID |

### Debug Inspection (5 tools)

| Tool | Required Mode | Description |
|------|--------------|-------------|
| `debug_get_callstack` | Break | Get call stack of current thread |
| `debug_get_locals` | Break | Get local variables in current frame |
| `debug_evaluate` | Break | Evaluate expression read-only (e.g. `myList.Count`) |
| `output_read` | Any | Read VS Output window pane (Build, Debug, etc.) |
| `error_list_get` | Any | Get errors/warnings from VS Error List |

### Breakpoint Management (4 tools)

| Tool | Required Mode | Description |
|------|--------------|-------------|
| `breakpoint_set` | Any | Set breakpoint by file+line or function name |
| `breakpoint_remove` | Any | Remove breakpoint at file+line |
| `breakpoint_list` | Any | List all breakpoints in solution |
| `exception_settings_set` | Any | Configure break-on-exception |

## Core Concept: Polling

MCP is request-response — there are **no push notifications**. You must poll `debug_get_mode` to detect state changes.

```
debug_start → returns immediately
              (VS is building + launching in background)

poll debug_get_mode → "Design"  (still building)
wait 3-5 seconds
poll debug_get_mode → "Running" (app launched)
wait 2 seconds
poll debug_get_mode → "Break"   (breakpoint hit!)
→ now inspect: callstack, locals, evaluate
```

The same polling applies after `debug_attach` + `debug_break`, or after `debug_continue`.

## Workflows

### 1. Attach to a Running Process — THE PRIMARY AI WORKFLOW

This is the most natural and powerful debugging workflow for AI:
- The application is already running (launched by user, by a script, or as a service)
- VS has the source code open in a solution
- The AI attaches, pauses, investigates, and detaches

```
1. debug_list_processes ()                        → see what's running locally
2. debug_attach (processName: "MyApp")            → attach debugger
3. debug_get_mode                                 → confirm "Running"
4. breakpoint_set (functionName: "HandleRequest") → set breakpoint on hot path
5. poll debug_get_mode                            → wait for "Break"
   ... or use debug_break to pause immediately:
5. debug_break                                    → pause the app NOW
6. debug_get_callstack                            → see where we are
7. debug_get_locals                               → see all variable values
8. debug_evaluate ("request.Url")                 → evaluate specific expression
9. debug_step (direction: "over")                 → step forward
10. debug_get_locals                              → see how values changed
11. debug_continue                                → let it run
12. debug_stop                                    → detach when done
```

**When to use attach vs start:**
- **`debug_attach`**: Process already running. Web servers, services, Docker containers, user-launched apps. This is 90% of real debugging.
- **`debug_start`**: Need VS to build and launch the startup project (F5). Good for console apps during development.

### 2. Attach to Docker Container

For debugging .NET apps running inside Docker containers on the local machine.

```
1. debug_list_transports                                  → verify "Docker (Linux Container)" available
2. debug_list_processes (transport: "Docker (Linux Container)",
                         qualifier: "my-container-name")  → find the dotnet process
3. breakpoint_set (functionName: "MyController.Get")      → set breakpoint first
4. debug_attach (processName: "dotnet",
                 transport: "Docker (Linux Container)",
                 qualifier: "my-container-name")          → attach
5. poll debug_get_mode                                    → wait for breakpoint or use debug_break
6. debug_get_callstack → debug_get_locals                 → investigate
```

**Note:** The process name inside a container is usually `dotnet` (for `dotnet run`) or the executable name (e.g. `MyApp`). Use `debug_list_processes` to discover it.

### 3. Attach to WSL Process

For debugging .NET apps running in Windows Subsystem for Linux.

```
1. debug_list_processes (transport: "WSL")         → find WSL processes
2. debug_attach (processName: "MyApp",
                 transport: "WSL")                 → attach
3. debug_break                                     → pause
4. debug_get_callstack → debug_get_locals          → investigate
```

### 4. User-Initiated Debug (TDD Workflow)

The user launches debug manually in VS (F5, or right-click → Debug Test). The AI investigates the state. **This is the most common workflow for TDD.**

```
User: "I started debugging, it hit a breakpoint, help me investigate"

1. debug_get_mode                → confirm "Break"
2. debug_get_callstack           → see the full call chain
3. debug_get_locals              → see all local variables
4. debug_evaluate ("this.Name")  → drill into specific values
5. debug_step (direction: "over")→ step to next line
6. debug_get_locals              → see how values changed
7. debug_evaluate ("result")     → check return value
```

The AI doesn't need to launch the debug session — it just needs `debug_get_mode` to return "Break", and all inspection tools work regardless of who started the session.

### 5. TDD: Debug a Failing Test

Unit tests cannot be launched in debug mode via MCP. The AI sets breakpoints, the user launches "Debug Test":

```
AI:   breakpoint_set (functionName: "MyService.ProcessOrder")
AI:   "Please right-click the failing test → Debug Test in VS"

User: runs Debug Test in VS

AI:   poll debug_get_mode → "Break"         ← breakpoint hit!
AI:   debug_get_callstack                    → see the exact code path
AI:   debug_get_locals                       → see variable values
AI:   debug_evaluate ("order.Items.Count")   → check specifics
AI:   debug_step (direction: "over")         → step to see what happens next
AI:   debug_get_locals                       → values after the step
AI:   "Found the bug: order.Items is empty because X calls Y before Z"
```

### 6. AI-Initiated Debug Session (F5)

When the AI wants to build, launch, and debug the startup project:

```
1. breakpoint_set (functionName: "MyClass.MyMethod")     ← set breakpoint
2. debug_start                                            ← launch (F5)
3. poll debug_get_mode until "Break"                      ← wait for breakpoint hit
4. debug_get_callstack                                    ← see where we are
5. debug_get_locals                                       ← see variable values
6. debug_evaluate ("myObject.Property")                   ← evaluate specific expressions
7. debug_step (direction: "over")                         ← step through code
8. debug_get_locals                                       ← see updated values
9. debug_continue                                         ← resume execution
10. debug_stop                                            ← end session
```

### 7. Set Breakpoints Before User Debugs

The AI can prepare breakpoints, then the user launches debug:

```
AI:   breakpoint_set (functionName: "OrderService.ProcessOrder")
AI:   breakpoint_set (file: "D:\\Path\\Validator.cs", line: 42)
AI:   "Breakpoints set. Please run your test in debug mode (right-click → Debug Test)"

User: runs debug test in VS

AI:   poll debug_get_mode → "Break"
AI:   debug_get_callstack → investigate
```

### 8. Investigate a Crash/Exception

```
1. exception_settings_set ("System.NullReferenceException", breakWhenThrown: true)
2. "Please run the failing scenario — I configured VS to break on NullReferenceException"
3. User runs the app or test
4. debug_get_mode → "Break" (exception hit!)
5. debug_get_callstack → see exactly where it crashed
6. debug_get_locals → see null values
7. debug_evaluate ("myObj") → confirm the null reference
8. output_read (pane: "Debug") → check debug output for clues
```

### 9. Read Build/Test Output

No debug session needed:

```
output_read ()                        → list available panes
output_read (pane: "Build")           → see build results
output_read (pane: "Debug")           → see debug trace output
output_read (pane: "Tests")           → see test results
error_list_get (severity: "error")    → see only errors
error_list_get (severity: "all")      → see everything
```

### 10. Modify Values at Runtime

When paused at a breakpoint, you can change variable values to test hypotheses:

```
debug_get_locals                          → see current values
immediate_execute ("retryCount = 0")      → reset a counter
immediate_execute ("isDebug = true")      → flip a flag
immediate_execute ("name = \"test\"")     → set a string
debug_continue                            → continue with modified values
```

## Breakpoint Types

### File + Line Breakpoint
```
breakpoint_set (file: "D:\\Path\\MyClass.cs", line: 42)
```
- Use **full Windows path** for reliability
- The file must exist and be part of the solution

### Function Breakpoint
```
breakpoint_set (functionName: "MyClass.MyMethod")
```
- More reliable than file+line (no path issues)
- Binds when the assembly loads
- Use `ClassName.MethodName` format

### Conditional Breakpoint
```
breakpoint_set (functionName: "ProcessItem", condition: "item.Id > 100")
```

## Transport System

Visual Studio supports debugging across different environments through **transports**. Use `debug_list_transports` to see what's available on the current machine.

### Common Transports

| Transport Name | Qualifier | Use Case |
|---------------|-----------|----------|
| `Default` | `""` (empty) or machine name | Local processes, or remote with Remote Debugger |
| `Docker (Linux Container)` | Container name or ID | Docker containers on local machine |
| `WSL` | `""` (empty) | Processes in Windows Subsystem for Linux |

### Discovery Flow

```
1. debug_list_transports          → see what's installed
2. debug_list_processes           → for local (default)
   debug_list_processes (transport: "Docker (Linux Container)",
                         qualifier: "container-name")  → for Docker
   debug_list_processes (transport: "WSL")              → for WSL
3. debug_attach (processName: ..., transport: ..., qualifier: ...)
```

## Docker Debugging Prerequisites

For `debug_attach` to work with Docker containers, the container needs a **debugger runtime** installed. VS uses `vsdbg` (Visual Studio Debugger for .NET Core).

### What the Container Needs

1. **vsdbg installed inside the container** — This is the .NET Core debugger that VS connects to.

   In your Dockerfile, add:
   ```dockerfile
   # Install vsdbg for remote debugging
   RUN apt-get update && apt-get install -y curl unzip procps \
       && curl -sSL https://aka.ms/getvsdbgsh | bash /dev/stdin -v latest -l /vsdbg
   ```

   Or in an alpine-based image:
   ```dockerfile
   RUN apk add --no-cache curl bash unzip \
       && curl -sSL https://aka.ms/getvsdbgsh | bash /dev/stdin -v latest -l /vsdbg
   ```

2. **The app must be built in Debug configuration** — Release builds strip debug symbols.

   ```dockerfile
   # Build stage
   FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
   WORKDIR /src
   COPY . .
   RUN dotnet publish -c Debug -o /app

   # Runtime stage
   FROM mcr.microsoft.com/dotnet/aspnet:8.0
   WORKDIR /app
   COPY --from=build /app .

   # Install vsdbg
   RUN apt-get update && apt-get install -y curl unzip \
       && curl -sSL https://aka.ms/getvsdbgsh | bash /dev/stdin -v latest -l /vsdbg

   ENTRYPOINT ["dotnet", "MyApp.dll"]
   ```

3. **PDB files present** — The `.pdb` debug symbol files must be in the container alongside the DLLs. `dotnet publish -c Debug` includes them automatically.

4. **Source code in VS matches the container build** — VS maps the source from the container back to local files. The source must match or breakpoints won't bind.

### Docker Compose Debug Profile

For development, use a debug-specific compose file:

```yaml
# docker-compose.debug.yml
services:
  myapp:
    build:
      context: .
      dockerfile: Dockerfile.debug
    ports:
      - "5000:5000"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - DOTNET_USE_POLLING_FILE_WATCHER=1
    volumes:
      - ./src:/src  # optional: mount source for hot reload
```

### VS Container Tools (Automatic Setup)

If you use **Visual Studio Container Tools** (right-click project → Add → Docker Support), VS handles everything automatically:
- Generates a multi-stage Dockerfile with vsdbg
- Configures `launchSettings.json` with Docker profile
- F5 builds, deploys to container, and attaches — all automatic

For projects already using VS Container Tools, `debug_attach` via MCP works the same way — VS has the transport set up.

### Minimal Docker Setup (Without VS Container Tools)

If you manage Docker yourself:

```bash
# 1. Build your app in Debug mode
docker build -t myapp:debug -f Dockerfile.debug .

# 2. Run with port exposed
docker run -d --name myapp-debug -p 5000:5000 myapp:debug

# 3. Now use MCP tools to attach:
#    debug_list_processes(transport: "Docker (Linux Container)", qualifier: "myapp-debug")
#    debug_attach(processName: "dotnet", transport: "Docker (Linux Container)", qualifier: "myapp-debug")
```

## WSL Debugging Prerequisites

For WSL debugging:

1. **Same source code** — The VS solution must contain the same source as what's running in WSL
2. **Debug build** — The app must be built with `dotnet build -c Debug` inside WSL
3. **No extra setup needed** — VS 2022 has built-in WSL transport support, no remote debugger needed

```bash
# Inside WSL:
cd /mnt/c/MyProject
dotnet run --configuration Debug

# Then from MCP:
# debug_list_processes(transport: "WSL")
# debug_attach(processName: "MyApp", transport: "WSL")
```

## Mode Validation

All inspection tools validate the debugger mode and return helpful errors:

```json
{
  "error": "Debugger must be in Break mode to get locals. Current mode: Running. Poll debug_get_mode to wait for a breakpoint hit."
}
```

**Always check mode before calling inspection tools**, or be prepared to handle these errors gracefully.

## Important Behaviors

### Fire-and-Forget Start
`debug_start` returns immediately. VS builds the project and launches it in the background. This can take seconds to minutes for large solutions. **Always poll `debug_get_mode` after starting.**

### No Internal Timeouts
DTE operations have **no internal timeout** — they await until VS completes the operation. The MCP client (e.g. Claude Code) controls timeouts externally. This is by design: DTE COM calls are not cancellable once started on the UI thread, so an internal timeout would only return null while leaving VS blocked.

### Function Breakpoint Validation
Function breakpoints (`breakpoint_set` with `functionName`) are **pre-validated via Roslyn** before calling DTE. If the symbol doesn't exist in the solution, the tool returns an error immediately instead of risking a VS hang. Use `ClassName.MethodName` format.

### Managed Frame Navigation
When stopped in native code (e.g., CLR internals), `debug_get_locals` automatically navigates to the first managed (.NET) frame so you see relevant C#/VB/F# variables instead of native pointers.

### Cross-Frame Expression Evaluation
`debug_evaluate` searches across all stack frames if the expression can't be evaluated in the current frame. This helps when stopped in framework code but you need to see your variables.

## Variable Tree Navigation

Local variables are trees. `debug_get_locals` returns root-level variables with `hasMembers` indicating expandable nodes. Use `debug_evaluate` to drill into children by expression path.

```
1. debug_get_locals                           → root variables
   args: string[]       (hasMembers: false)
   testCases: List<T>   (hasMembers: true, memberCount: 7)
   httpClient: HttpClient (hasMembers: true, memberCount: 1)

2. debug_evaluate ("testCases")               → expand testCases
   → members: { "[0]": "TestCase", "[1]": "TestCase", ... "Count": "7" }

3. debug_evaluate ("testCases[0]")            → drill into first item
   → members: { "Data": "Dictionary<string,string>", "Description": "...", "Name": "..." }

4. debug_evaluate ("testCases[0].Data")       → drill into dictionary
   → members: { "[0]": "KeyValuePair<string,string>", "Count": "4" }

5. debug_evaluate ("testCases[0].Data.ElementAt(0)")  → get first entry
   → members: { "Key": "username", "Value": "alice_doe" }
```

**Pattern:** Each `debug_evaluate` call goes one level deeper. The expression path is standard C# — array indexing, property access, LINQ methods all work.

## Compact Output

All tools default to **compact** output format, optimized for token efficiency:

- **Callstack:** `"0": "Main|MyApp|Program.cs|14"` (function|module|file|line)
- **Processes:** `"svchost": "2360,2544,2588"` (name: PIDs) — paths stripped to just the name
- **Locals:** `"testCases": "List<TestCase>[7]"` (expandable with memberCount)
- **Breakpoints:** `"Program.cs:10": "bound|enabled"` (file:line: type|state)
- **Errors:** `"Validator.cs:42": "CS1002: ; expected"` (file:line: description)

Use `outputFormat: "verbose"` for full details when needed.

## Real-World Workflow Examples

### Escaping an Infinite Loop

The program has a `while(!done)` loop waiting for input. The AI breaks in, modifies the flag, and continues:

```
1. debug_get_mode                         → "Running" (stuck in while loop)
2. debug_break                            → pause the app
3. debug_get_locals                       → see done=false
4. immediate_execute ("done = true")      → flip the flag
5. debug_continue                         → loop exits, execution continues to next code
```

### Bypassing Environment Checks

The app exits if `ANTHROPIC_API_KEY` is not set. The AI sets a breakpoint before the check, injects a fake value, and continues past the exit:

```
1. breakpoint_set (file: "Program.cs", line: 20)      → before the if(IsNullOrEmpty) check
2. debug_start                                          → launch the app
3. debug_break + immediate_execute ("done = true")     → escape the while loop first
4. debug_continue                                       → hits breakpoint at line 20
5. immediate_execute ("apiKey = \"fake-key\"")          → inject value BEFORE the check
6. debug_continue                                       → IsNullOrEmpty returns false, app continues!
```

**Key insight:** Set the variable BEFORE stepping over the `if` check. If you step first, you're already inside the `if` block heading toward `Environment.Exit`.

### Investigating Selected Code

The user selects code they want to investigate. The AI reads the selection, sets breakpoints, and inspects at runtime:

```
1. get_selection                                    → read what the user selected
2. breakpoint_set (file: "...", line: startLine)   → breakpoint on the selected code
3. [navigate to the breakpoint — escape loops, bypass checks as needed]
4. debug_get_locals                                 → see variable state at that point
5. debug_step (direction: "over")                   → execute the selected line
6. debug_get_locals                                 → see result (e.g. testCases.Count = 7)
7. debug_step (direction: "over")                   → continue through the selection
```

This pattern lets the user say "investigate this code" by selecting it, and the AI does the rest.

## Tips for AI Agents

1. **Prefer `debug_attach` over `debug_start`** when the process is already running. This is the most natural AI debugging workflow.

2. **Poll, don't assume.** After `debug_start`, `debug_attach`, `debug_continue`, or `debug_step`, always check `debug_get_mode` before inspecting state.

3. **Prefer function breakpoints** over file+line when you know the method name. They're more reliable across code changes. Symbols are pre-validated via Roslyn — you get an immediate error if the function doesn't exist.

4. **The user can start debug themselves.** You don't always need `debug_start` or `debug_attach`. The most common TDD workflow is: user runs "Debug Test" in VS, AI investigates when it hits Break.

5. **Use `output_read` and `error_list_get` freely** — they work in any mode, no debug session needed.

6. **Don't call inspection tools in Design or Running mode** — they'll return errors. Always check mode first or handle the error gracefully.

7. **Use `immediate_execute` carefully** — it can modify program state. `debug_evaluate` is read-only and safer for just looking at values.

8. **When debugging tests:** You cannot launch a test in debug mode via MCP. Ask the user to right-click → Debug Test. Once they hit a breakpoint, you have full access to inspect and step.

9. **After `debug_step`**, the mode is still Break — you can immediately call inspection tools without polling.

10. **After `debug_continue`**, poll for the next Break or Design (program exited).

11. **Use `breakpoint_list`** to see what breakpoints exist before adding duplicates.

12. **Use `debug_list_transports`** to discover Docker/WSL support before trying to attach. Not every VS installation has the same transports.

13. **For Docker/WSL**, the process name is usually `dotnet` for .NET apps. Use `debug_list_processes` to discover the exact process name and PID.

14. **After attaching to Docker/WSL**, all tools work identically to local debugging — callstack, locals, evaluate, step, breakpoints all work the same way.
