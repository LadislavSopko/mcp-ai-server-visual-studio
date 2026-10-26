# AI Debugging Guide for MCP AI Server (vs-mcp)

## Overview

MCP AI Server exposes 19 debugging tools that let AI assistants interact with the Visual Studio debugger. The debugger is **state-based** — you must check the current mode before calling tools, and use **polling** to detect state changes.

There are two fundamental ways to start a debug session:
1. **`DebugStart`** — Press F5, launches the startup project with debugger attached
2. **`DebugAttach`** — Attach to an already-running process (local, Docker, or WSL)

In both cases, once in Break mode, all inspection tools work identically.

## Debugger Modes

| Mode | Meaning | How you get here |
|------|---------|-----------------|
| **Design** | No debug session active | Default state, or after DebugStop / program exits |
| **Running** | App is executing | After DebugStart, DebugAttach, or DebugContinue |
| **Break** | Paused at breakpoint/exception | Breakpoint hit, DebugBreak, or DebugStep completed |

## Available Tools (19)

### Debug Control (10 tools)

| Tool | Required Mode | Description |
|------|--------------|-------------|
| `DebugGetMode` | Any | Returns current mode: Design, Running, or Break |
| `DebugStart` | Design | Start debugging (F5). Fire-and-forget — returns immediately |
| `DebugStop` | Running/Break | Stop debugging session (detach + terminate) |
| `DebugBreak` | Running | Pause the running application |
| `DebugContinue` | Break | Resume execution |
| `DebugStep` | Break | Step over/into/out. Direction: `"over"`, `"into"`, `"out"` |
| `ImmediateExecute` | Break | Execute expression with side effects (e.g. `myVar = 42`) |
| `DebugListTransports` | Any | List available transports (Default, Docker, WSL, etc.) |
| `DebugListProcesses` | Any | List processes on a transport for attachment |
| `DebugAttach` | Design | Attach debugger to a running process by name or PID |

### Debug Inspection (5 tools)

| Tool | Required Mode | Description |
|------|--------------|-------------|
| `DebugGetCallstack` | Break | Get call stack of current thread |
| `DebugGetLocals` | Break | Get local variables in current frame |
| `DebugEvaluate` | Break | Evaluate expression read-only (e.g. `myList.Count`) |
| `OutputRead` | Any | Read VS Output window pane (Build, Debug, etc.) |
| `ErrorListGet` | Any | Get errors/warnings from VS Error List |

### Breakpoint Management (4 tools)

| Tool | Required Mode | Description |
|------|--------------|-------------|
| `BreakpointSet` | Any | Set breakpoint by file+line or function name |
| `BreakpointRemove` | Any | Remove breakpoint at file+line |
| `BreakpointList` | Any | List all breakpoints in solution |
| `ExceptionSettingsSet` | Any | Configure break-on-exception |

## Core Concept: Polling

MCP is request-response — there are **no push notifications**. You must poll `DebugGetMode` to detect state changes.

```
DebugStart → returns immediately
              (VS is building + launching in background)

poll DebugGetMode → "Design"  (still building)
wait 3-5 seconds
poll DebugGetMode → "Running" (app launched)
wait 2 seconds
poll DebugGetMode → "Break"   (breakpoint hit!)
→ now inspect: callstack, locals, evaluate
```

The same polling applies after `DebugAttach` + `DebugBreak`, or after `DebugContinue`.

## Workflows

### 1. Attach to a Running Process — THE PRIMARY AI WORKFLOW

This is the most natural and powerful debugging workflow for AI:
- The application is already running (launched by user, by a script, or as a service)
- VS has the source code open in a solution
- The AI attaches, pauses, investigates, and detaches

```
1. DebugListProcesses ()                        → see what's running locally
2. DebugAttach (processName: "MyApp")            → attach debugger
3. DebugGetMode                                 → confirm "Running"
4. BreakpointSet (functionName: "HandleRequest") → set breakpoint on hot path
5. poll DebugGetMode                            → wait for "Break"
   ... or use DebugBreak to pause immediately:
5. DebugBreak                                    → pause the app NOW
6. DebugGetCallstack                            → see where we are
7. DebugGetLocals                               → see all variable values
8. DebugEvaluate ("request.Url")                 → evaluate specific expression
9. DebugStep (direction: "over")                 → step forward
10. DebugGetLocals                              → see how values changed
11. DebugContinue                                → let it run
12. DebugStop                                    → detach when done
```

**When to use attach vs start:**
- **`DebugAttach`**: Process already running. Web servers, services, Docker containers, user-launched apps. This is 90% of real debugging.
- **`DebugStart`**: Need VS to build and launch the startup project (F5). Good for console apps during development.

### 2. Attach to Docker Container

For debugging .NET apps running inside Docker containers on the local machine.

```
1. DebugListTransports                                  → verify "Docker (Linux Container)" available
2. DebugListProcesses (transport: "Docker (Linux Container)",
                         qualifier: "my-container-name")  → find the dotnet process
3. BreakpointSet (functionName: "MyController.Get")      → set breakpoint first
4. DebugAttach (processName: "dotnet",
                 transport: "Docker (Linux Container)",
                 qualifier: "my-container-name")          → attach
5. poll DebugGetMode                                    → wait for breakpoint or use DebugBreak
6. DebugGetCallstack → DebugGetLocals                 → investigate
```

**Note:** The process name inside a container is usually `dotnet` (for `dotnet run`) or the executable name (e.g. `MyApp`). Use `DebugListProcesses` to discover it.

### 3. Attach to WSL Process

For debugging .NET apps running in Windows Subsystem for Linux.

```
1. DebugListProcesses (transport: "WSL")         → find WSL processes
2. DebugAttach (processName: "MyApp",
                 transport: "WSL")                 → attach
3. DebugBreak                                     → pause
4. DebugGetCallstack → DebugGetLocals          → investigate
```

### 4. User-Initiated Debug (TDD Workflow)

The user launches debug manually in VS (F5, or right-click → Debug Test). The AI investigates the state. **This is the most common workflow for TDD.**

```
User: "I started debugging, it hit a breakpoint, help me investigate"

1. DebugGetMode                → confirm "Break"
2. DebugGetCallstack           → see the full call chain
3. DebugGetLocals              → see all local variables
4. DebugEvaluate ("this.Name")  → drill into specific values
5. DebugStep (direction: "over")→ step to next line
6. DebugGetLocals              → see how values changed
7. DebugEvaluate ("result")     → check return value
```

The AI doesn't need to launch the debug session — it just needs `DebugGetMode` to return "Break", and all inspection tools work regardless of who started the session.

### 5. TDD: Debug a Failing Test

Unit tests cannot be launched in debug mode via MCP. The AI sets breakpoints, the user launches "Debug Test":

```
AI:   BreakpointSet (functionName: "MyService.ProcessOrder")
AI:   "Please right-click the failing test → Debug Test in VS"

User: runs Debug Test in VS

AI:   poll DebugGetMode → "Break"         ← breakpoint hit!
AI:   DebugGetCallstack                    → see the exact code path
AI:   DebugGetLocals                       → see variable values
AI:   DebugEvaluate ("order.Items.Count")   → check specifics
AI:   DebugStep (direction: "over")         → step to see what happens next
AI:   DebugGetLocals                       → values after the step
AI:   "Found the bug: order.Items is empty because X calls Y before Z"
```

### 6. AI-Initiated Debug Session (F5)

When the AI wants to build, launch, and debug the startup project:

```
1. BreakpointSet (functionName: "MyClass.MyMethod")     ← set breakpoint
2. DebugStart                                            ← launch (F5)
3. poll DebugGetMode until "Break"                      ← wait for breakpoint hit
4. DebugGetCallstack                                    ← see where we are
5. DebugGetLocals                                       ← see variable values
6. DebugEvaluate ("myObject.Property")                   ← evaluate specific expressions
7. DebugStep (direction: "over")                         ← step through code
8. DebugGetLocals                                       ← see updated values
9. DebugContinue                                         ← resume execution
10. DebugStop                                            ← end session
```

### 7. Set Breakpoints Before User Debugs

The AI can prepare breakpoints, then the user launches debug:

```
AI:   BreakpointSet (functionName: "OrderService.ProcessOrder")
AI:   BreakpointSet (file: "D:\\Path\\Validator.cs", line: 42)
AI:   "Breakpoints set. Please run your test in debug mode (right-click → Debug Test)"

User: runs debug test in VS

AI:   poll DebugGetMode → "Break"
AI:   DebugGetCallstack → investigate
```

### 8. Investigate a Crash/Exception

```
1. ExceptionSettingsSet ("System.NullReferenceException", breakWhenThrown: true)
2. "Please run the failing scenario — I configured VS to break on NullReferenceException"
3. User runs the app or test
4. DebugGetMode → "Break" (exception hit!)
5. DebugGetCallstack → see exactly where it crashed
6. DebugGetLocals → see null values
7. DebugEvaluate ("myObj") → confirm the null reference
8. OutputRead (pane: "Debug") → check debug output for clues
```

### 9. Read Build/Test Output

No debug session needed:

```
OutputRead ()                        → list available panes
OutputRead (pane: "Build")           → see build results
OutputRead (pane: "Debug")           → see debug trace output
OutputRead (pane: "Tests")           → see test results
ErrorListGet (severity: "error")    → see only errors
ErrorListGet (severity: "all")      → see everything
```

### 10. Modify Values at Runtime

When paused at a breakpoint, you can change variable values to test hypotheses:

```
DebugGetLocals                          → see current values
ImmediateExecute ("retryCount = 0")      → reset a counter
ImmediateExecute ("isDebug = true")      → flip a flag
ImmediateExecute ("name = \"test\"")     → set a string
DebugContinue                            → continue with modified values
```

## Breakpoint Types

### File + Line Breakpoint
```
BreakpointSet (file: "D:\\Path\\MyClass.cs", line: 42)
```
- Use **full Windows path** for reliability
- The file must exist and be part of the solution

### Function Breakpoint
```
BreakpointSet (functionName: "MyClass.MyMethod")
```
- More reliable than file+line (no path issues)
- Binds when the assembly loads
- Use `ClassName.MethodName` format

### Conditional Breakpoint
```
BreakpointSet (functionName: "ProcessItem", condition: "item.Id > 100")
```

## Transport System

Visual Studio supports debugging across different environments through **transports**. Use `DebugListTransports` to see what's available on the current machine.

### Common Transports

| Transport Name | Qualifier | Use Case |
|---------------|-----------|----------|
| `Default` | `""` (empty) or machine name | Local processes, or remote with Remote Debugger |
| `Docker (Linux Container)` | Container name or ID | Docker containers on local machine |
| `WSL` | `""` (empty) | Processes in Windows Subsystem for Linux |

### Discovery Flow

```
1. DebugListTransports          → see what's installed
2. DebugListProcesses           → for local (default)
   DebugListProcesses (transport: "Docker (Linux Container)",
                         qualifier: "container-name")  → for Docker
   DebugListProcesses (transport: "WSL")              → for WSL
3. DebugAttach (processName: ..., transport: ..., qualifier: ...)
```

## Docker Debugging Prerequisites

For `DebugAttach` to work with Docker containers, the container needs a **debugger runtime** installed. VS uses `vsdbg` (Visual Studio Debugger for .NET Core).

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

For projects already using VS Container Tools, `DebugAttach` via MCP works the same way — VS has the transport set up.

### Minimal Docker Setup (Without VS Container Tools)

If you manage Docker yourself:

```bash
# 1. Build your app in Debug mode
docker build -t myapp:debug -f Dockerfile.debug .

# 2. Run with port exposed
docker run -d --name myapp-debug -p 5000:5000 myapp:debug

# 3. Now use MCP tools to attach:
#    DebugListProcesses(transport: "Docker (Linux Container)", qualifier: "myapp-debug")
#    DebugAttach(processName: "dotnet", transport: "Docker (Linux Container)", qualifier: "myapp-debug")
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
# DebugListProcesses(transport: "WSL")
# DebugAttach(processName: "MyApp", transport: "WSL")
```

## Mode Validation

All inspection tools validate the debugger mode and return helpful errors:

```json
{
  "error": "Debugger must be in Break mode to get locals. Current mode: Running. Poll DebugGetMode to wait for a breakpoint hit."
}
```

**Always check mode before calling inspection tools**, or be prepared to handle these errors gracefully.

## Important Behaviors

### Fire-and-Forget Start
`DebugStart` returns immediately. VS builds the project and launches it in the background. This can take seconds to minutes for large solutions. **Always poll `DebugGetMode` after starting.**

### No Internal Timeouts
DTE operations have **no internal timeout** — they await until VS completes the operation. The MCP client (e.g. Claude Code) controls timeouts externally. This is by design: DTE COM calls are not cancellable once started on the UI thread, so an internal timeout would only return null while leaving VS blocked.

### Function Breakpoint Validation
Function breakpoints (`BreakpointSet` with `functionName`) are **pre-validated via Roslyn** before calling DTE. If the symbol doesn't exist in the solution, the tool returns an error immediately instead of risking a VS hang. Use `ClassName.MethodName` format.

### Managed Frame Navigation
When stopped in native code (e.g., CLR internals), `DebugGetLocals` automatically navigates to the first managed (.NET) frame so you see relevant C#/VB/F# variables instead of native pointers.

### Cross-Frame Expression Evaluation
`DebugEvaluate` searches across all stack frames if the expression can't be evaluated in the current frame. This helps when stopped in framework code but you need to see your variables.

## Variable Tree Navigation

Local variables are trees. `DebugGetLocals` returns root-level variables with `hasMembers` indicating expandable nodes. Use `DebugEvaluate` to drill into children by expression path.

```
1. DebugGetLocals                           → root variables
   args: string[]       (hasMembers: false)
   testCases: List<T>   (hasMembers: true, memberCount: 7)
   httpClient: HttpClient (hasMembers: true, memberCount: 1)

2. DebugEvaluate ("testCases")               → expand testCases
   → members: { "[0]": "TestCase", "[1]": "TestCase", ... "Count": "7" }

3. DebugEvaluate ("testCases[0]")            → drill into first item
   → members: { "Data": "Dictionary<string,string>", "Description": "...", "Name": "..." }

4. DebugEvaluate ("testCases[0].Data")       → drill into dictionary
   → members: { "[0]": "KeyValuePair<string,string>", "Count": "4" }

5. DebugEvaluate ("testCases[0].Data.ElementAt(0)")  → get first entry
   → members: { "Key": "username", "Value": "alice_doe" }
```

**Pattern:** Each `DebugEvaluate` call goes one level deeper. The expression path is standard C# — array indexing, property access, LINQ methods all work.

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
1. DebugGetMode                         → "Running" (stuck in while loop)
2. DebugBreak                            → pause the app
3. DebugGetLocals                       → see done=false
4. ImmediateExecute ("done = true")      → flip the flag
5. DebugContinue                         → loop exits, execution continues to next code
```

### Bypassing Environment Checks

The app exits if `ANTHROPIC_API_KEY` is not set. The AI sets a breakpoint before the check, injects a fake value, and continues past the exit:

```
1. BreakpointSet (file: "Program.cs", line: 20)      → before the if(IsNullOrEmpty) check
2. DebugStart                                          → launch the app
3. DebugBreak + ImmediateExecute ("done = true")     → escape the while loop first
4. DebugContinue                                       → hits breakpoint at line 20
5. ImmediateExecute ("apiKey = \"fake-key\"")          → inject value BEFORE the check
6. DebugContinue                                       → IsNullOrEmpty returns false, app continues!
```

**Key insight:** Set the variable BEFORE stepping over the `if` check. If you step first, you're already inside the `if` block heading toward `Environment.Exit`.

### Investigating Selected Code

The user selects code they want to investigate. The AI reads the selection, sets breakpoints, and inspects at runtime:

```
1. GetSelection                                    → read what the user selected
2. BreakpointSet (file: "...", line: startLine)   → breakpoint on the selected code
3. [navigate to the breakpoint — escape loops, bypass checks as needed]
4. DebugGetLocals                                 → see variable state at that point
5. DebugStep (direction: "over")                   → execute the selected line
6. DebugGetLocals                                 → see result (e.g. testCases.Count = 7)
7. DebugStep (direction: "over")                   → continue through the selection
```

This pattern lets the user say "investigate this code" by selecting it, and the AI does the rest.

## Tips for AI Agents

1. **Prefer `DebugAttach` over `DebugStart`** when the process is already running. This is the most natural AI debugging workflow.

2. **Poll, don't assume.** After `DebugStart`, `DebugAttach`, `DebugContinue`, or `DebugStep`, always check `DebugGetMode` before inspecting state.

3. **Prefer function breakpoints** over file+line when you know the method name. They're more reliable across code changes. Symbols are pre-validated via Roslyn — you get an immediate error if the function doesn't exist.

4. **The user can start debug themselves.** You don't always need `DebugStart` or `DebugAttach`. The most common TDD workflow is: user runs "Debug Test" in VS, AI investigates when it hits Break.

5. **Use `OutputRead` and `ErrorListGet` freely** — they work in any mode, no debug session needed.

6. **Don't call inspection tools in Design or Running mode** — they'll return errors. Always check mode first or handle the error gracefully.

7. **Use `ImmediateExecute` carefully** — it can modify program state. `DebugEvaluate` is read-only and safer for just looking at values.

8. **When debugging tests:** You cannot launch a test in debug mode via MCP. Ask the user to right-click → Debug Test. Once they hit a breakpoint, you have full access to inspect and step.

9. **After `DebugStep`**, the mode is still Break — you can immediately call inspection tools without polling.

10. **After `DebugContinue`**, poll for the next Break or Design (program exited).

11. **Use `BreakpointList`** to see what breakpoints exist before adding duplicates.

12. **Use `DebugListTransports`** to discover Docker/WSL support before trying to attach. Not every VS installation has the same transports.

13. **For Docker/WSL**, the process name is usually `dotnet` for .NET apps. Use `DebugListProcesses` to discover the exact process name and PID.

14. **After attaching to Docker/WSL**, all tools work identically to local debugging — callstack, locals, evaluate, step, breakpoints all work the same way.
