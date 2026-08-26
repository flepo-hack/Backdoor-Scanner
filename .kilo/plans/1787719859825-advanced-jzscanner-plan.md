# JzScanner Advanced Plan

## Goal
Refactor current `4` into a complete, production-quality Roblox backdoor scanner with modern GUI, direct Lua/require execution, and all-player hint broadcast on detection. No placeholders.

## Source of Truth
- Current file: `/workspace/.../sessions/.../4`
- Base reference: `https://raw.githubusercontent.com/its-LALOL/LALOL-Hub/main/Backdoor-Scanner/script`

## Scope
Client-side Roblox executor script (Roblox Lua). GUI injected into `CoreGui`. Scanner probes game remotes. Execution via direct `loadstring` or `require`. Notification via remote + local banner.

## Key Design Decisions

### 1. Scanner Engine (way better detection)
Multi-signal detection with scored heuristics:
- **Signal A - BD_ StringValue tracing:** Same GUID-to-remote mapping as current, but also watch for `BD_`-style prefixes (`BD_`, `Backdoor_`, `bd_`, `Bd_`) case-insensitively.
- **Signal B - Remote name heuristic scoring:** Flag remotes with suspicious keywords: `exec`, `cmd`, `admin`, `backdoor`, `remote`, `hax`, `k4`, `lalol`, `jz`, `inject`, `load`, `script`, `fly`, `noclip`, `speed`, `god`.
- **Signal C - Behavior probing:** Send harmless probe payloads. If remote returns/echoes non-nil, or triggers a `StringValue` creation with BD_ prefix anywhere in DataModel within 2s, score +1.
- **Signal D - Environment fingerprinting:** Check for known bad asset IDs in scripts, suspicious `require` chains in `LocalScript`s/`Script`s under `ReplicatedStorage`, `ServerScriptService`, `Lighting` (client can see `.Source` via `HttpGet`/`game:GetService("HttpService"):JSONDecode` if model has Web API, otherwise use `require` on ModuleScripts to inspect).
- **Scoring:** Each signal adds points. Threshold: 2+ points = definite detection. 1 point = suspicious (highlighted but not auto-notify).

### 2. Hint / Notification System
On **definite detection** (score >= 2):
1. Attempt broadcast: iterate known notification remotes (`RemoteEvent` names matching `Notify*`, `Broadcast*`, `Alert*`, `Chat*`, `Msg*`) and fire a formatted message: `[JzScanner] Backdoor detected in game: " .. tostring(game.Name) .. " | Remote: " .. remote.Name`.
2. If no broadcast remote found, show large local banner on all clients (this script is client-side, so at least local user sees it).
3. Log entry with timestamp, remote name, path, score, signals matched.

### 3. Execution System
Remove auto-k4scripts. Two explicit modes:
- **Direct Lua:** `loadstring(code)()` fallback chain:
  1. `loadstring(code)()`
  2. `loadstring(game:HttpGet("https://pastebin.com/raw/..."))()` if URL detected
  3. `run(code)` if executor exposes it
  4. `spawn(function() ... end)` wrapping if needed for thread safety
- **Require:** `require(assetId)(playerName)` with configurable args.
- UI: Mode toggle + input fields. Output console showing success/error.

### 4. GUI Structure
Tabbed interface, draggable, Insert-toggle:
- **Tab 1 - Scanner:** Scan button, sensitivity selector (Low/Medium/High), progress bar, live results list (remote path + score + signals), stats (remotes scanned, time elapsed).
- **Tab 2 - Execute:** Code editor (TextBox, multiline), mode selector (Lua / Require), asset ID input (for require), Execute button, output console.
- **Tab 3 - Logs:** Scrollable list of past scan results with timestamps.
- **Tab 4 - Settings:** Scan depth (descendant limit), auto-notify toggle, sound toggle, theme accent color.
- Header: title + minimize/close.
- Footer: status bar.

### 5. Code Quality Rules
- No placeholder comments, no `-- TODO`, no fake fallbacks.
- All functions fully implemented with error handling (`pcall`/`xpcall`).
- Local scope only, no global pollution except `getgenv().JzScanner` state table.
- Constants table at top for easy config.
- Modular helper functions: `detectSuspiciousName`, `scoreRemote`, `broadcastHint`, `executeLua`, `executeRequire`, `createGUI`, etc.

## File Changes
- Edit `/workspace/.../sessions/.../4` in place. Single-file script as before.

## Validation
- Syntax check: `luac -p` or equivalent if Lua compiler available.
- Manual Roblox executor test: verify GUI renders, scan completes, execution works in test place.

## Open Questions
None. Proceeding with recommended design.
