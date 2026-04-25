# LM Studio MCP Tools Troubleshooting: Case Study

> **System Context:** Windows 11 Home + WSL2 (Ubuntu 24.04) + Docker Desktop + LM Studio with MCP plugins

## Problem Statement

I had two non-functional MCP tools in LM Studio on a Windows PC running WSL2:

| Tool | Issue |
|------|------|
| **mcp-docker** | Reported "Docker not recognized" despite Docker Desktop being active |
| **mcp-searxng** | Initially loaded but failed with `spawn cmd.exe ENOENT` after PATH fix |

Both issues have been **successfully resolved**. This case study documents the diagnostic process, root causes, and solutions.

---

## Part 1: Diagnosing and Fixing mcp-docker

### Initial Error

```
2026-04-24 20:24:36 [ERROR]
[Plugin(mcp/mcp-docker)] stderr: 'docker' is not recognized as an internal or external command,
operable program or batch file.
[ERROR]
[Plugin(mcp/mcp-docker)] stderr: [MCPBridge/unexpected] Bridge process failed: MCP error -32000: Connection closed
```

### Initial Assumptions & Red Herrings

- Docker Desktop was running in system tray ✓
- I had already restarted LM Studio once ✗ (not enough)
- Thought: "Docker is running as a service, but the CLI tools aren't accessible" → **Wrong assumption**

### Diagnostic Process

#### Step 1: Check if WSL 2 was the culprit
Navigated to Docker Desktop Settings → Resources → WSL Integration  
Found that **"Use the WSL 2 based engine"** was **locked on** because I'm on Windows Home edition. This forced Docker to use WSL 2 backend exclusively — *but this wasn't the root cause*.

#### Step 2: Verify WSL 2 installation
- Opened PowerShell and ran `wsl --list --verbose`  
- **Discovery:** I was running this command inside a WSL terminal, not native Windows!  
- This returned a different `wsl` tool (WinRM Shell) instead of Windows Subsystem for Linux.

#### Step 3: Check WSL 2 from native Windows
I ran from native Command Prompt: `C:\Windows\System32\wsl.exe -l -v`  
**Confirmed:** WSL 2 was properly installed with Ubuntu-24.04 and docker-desktop both running. This meant WSL 2 itself wasn't the problem.

#### Step 4: Test Docker CLI directly
From native Command Prompt: `docker --version`  
**Result:** Failed with `'docker' is not recognized`  
**Conclusion:** Docker CLI files exist and work, but aren't in Windows PATH.

#### Step 5: Verify Docker executable exists
```bash
dir "C:\Program Files\Docker\Docker\resources\bin"
```
**Result:** Found `docker.exe` (43,100,080 bytes) along with other Docker tools. Confirmed the executable actually exists.

#### Step 6: Test with full path
```bash
"C:\Program Files\Docker\Docker\resources\bin\docker.exe" --version
```
**Result:** `Docker version 29.4.0, build 9d7ad9f`  
**Confirmed:** Docker works perfectly when called with full path.

### Root Cause Confirmed

The directory `C:\Program Files\Docker\Docker\resources\bin` is **not in system PATH**. The executable exists and works, but Windows can't find it via the default `docker` command.

### Solution Implementation

1. Opened Windows Environment Variables (`"environment variables"` search)
2. Clicked "Edit the system environment variables"
3. Clicked "Environment Variables..." button
4. Located **"Path"** variable in **System variables** section
5. Clicked "Edit..."
6. Added new entry: `C:\Program Files\Docker\Docker\resources\bin`
7. Saved all dialogs

### Verification of Docker Fix

- Closed all Command Prompt windows completely (important!)
- Opened fresh Command Prompt
- Ran: `docker --version`  
- **Result:** `Docker version 29.4.0, build 9d7ad9f` ✓ Success

---

## Part 2: mcp-searxng Issue and Resolution

### Timeline of Changes

**Before Docker PATH fix:**
```
2026-04-24 20:40:57 [DEBUG]
[LMSAuthenticator][Client=plugin:installed:mcp/searxng] Client created.
2026-04-24 20:40:58 [INFO]
[Plugin(mcp/searxng)] stdout: [Tools Prvdr.] Register with LM Studio
```
mcp-searxng would load and register successfully, but had underlying issues when used.

**After Docker PATH fix and LM Studio restart:**
```
2026-04-24 20:47:52 [DEBUG]
[LMSAuthenticator][Client=plugin:installed:mcp/searxng] Client created.
2026-04-24 20:47:53 [ERROR]
[Plugin(mcp/searxng)] stderr: [MCPBridge/unexpected] Bridge process failed: spawn cmd.exe ENOENT
```

This pattern repeated consistently every 3-5 seconds across multiple retry attempts.

### Root Cause Analysis

The error `spawn cmd.exe ENOENT` ("Error No Entry") means mcp-searxng is trying to spawn a child process using `cmd.exe`, but the subprocess can't find it.

The mcp-searxng configuration was:
```json
"searxng": {
  "command": "npx",
  "args": ["-y", "mcp-searxng"],
  "env": {
    "SEARXNG_URL": "http://localhost:8080"
  }
}
```

**Key discovery:** mcp-searxng uses `npx` to run the tool, which then spawns `cmd.exe` internally. When the subprocess was launched by LM Studio, it wasn't inheriting the updated system PATH that included `cmd.exe`.

### Why This Happened

1. **PATH environment variable was updated** — Docker was added to system PATH
2. **LM Studio was restarted, but the computer wasn't** — LM Studio's process inherited the old PATH from before the change
3. **mcp-searxng's subprocess had incomplete environment** — When it tried to spawn `cmd.exe`, the subprocess couldn't find it in its inherited PATH
4. **Testing revealed both components worked individually** — `npx --version` worked from Command Prompt, and SearXNG was running and accessible

### Solution Implementation

The fix was simple: **full system restart**

**Steps taken:**
1. Restarted the entire computer (not just LM Studio!)
2. After restart, opened LM Studio fresh
3. Both mcp-docker and mcp-searxng loaded successfully

### Why this worked

- A full system restart ensures all new processes inherit the updated system PATH
- All processes launched after restart, including LM Studio, get the complete and current environment variables
- This provided mcp-searxng's subprocess with access to `cmd.exe`

### Verification of mcp-searxng Fix

After restart:
```
2026-04-24 [DEBUG]
[LMSAuthenticator][Client=plugin:installed:mcp/searxng] Client created.
[INFO]
[Plugin(mcp/searxng)] stdout: [Tools Prvdr.] Register with LM Studio
```
mcp-searxng loads successfully — both tools now functional ✓

---

## Configuration File Reference

Final working configuration (saved at `~/.mcp/servers.json` or LM Studio's config location):
```json
{
  "mcpServers": {
    "searxng": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-searxng"
      ],
      "env": {
        "SEARXNG_URL": "http://localhost:8080"
      }
    },
    "MCP_DOCKER": {
      "command": "docker",
      "args": [
        "mcp",
        "gateway",
        "run"
      ],
      "env": {
        "LOCALAPPDATA": "C:\\Users\\RAZER ADMINISTRATOR\\AppData\\Local",
        "ProgramData": "C:\\ProgramData",
        "ProgramFiles": "C:\\Program Files"
      }
    }
  }
}
```

---

## Key Insights from This Session

### WSL vs Native Windows Testing Differences 🪟

I initially ran `wsl --list --verbose` inside a WSL terminal and got misleading output (WinRM Shell instead of actual WSL). The key insight: **the same command behaves differently depending on the execution context**. Always test from native Windows when troubleshooting Windows-specific tools.

> **Practical takeaway:** When debugging Windows applications, ensure you're testing from the correct shell environment — WSL and native Windows can yield different results for the same commands.

### Subprocess Environment Inheritance Issues 🔄

The mcp-searxng failure revealed a deeper issue: when LM Studio restarted but I didn't restart the OS, its process inherited the **old PATH** before my changes took effect. This caused downstream tools to fail when spawning subprocesses that needed system executables like `cmd.exe`.

> **Practical takeaway:** Application restarts alone don't update environment variables — only a full OS restart ensures all processes inherit updated system-level configurations.

### Log Timestamps Reveal Cause-and-Effect 📊

The error logs with precise timestamps showed exactly when changes took effect, making it easy to correlate:
- Which action caused which symptom
- That mcp-searxng suddenly failed after Docker PATH was added
- That both tools failed at the same time during restart, then succeeded together after full restart

> **Practical takeaway:** Timestamps are your friend! They help you trace cause-and-effect relationships and identify when issues appeared relative to your troubleshooting steps.

### The Stale Environment Problem 🔄

When I updated system PATH and restarted LM Studio (but not the OS), LM Studio's process still had the old environment variables cached. This is why mcp-searxng failed with `spawn cmd.exe ENOENT` — its subprocess couldn't find `cmd.exe` because LM Studio inherited the pre-update PATH.

> **Practical takeaway:** For system-level changes like PATH modifications, a full OS restart is often necessary to ensure all processes operate with updated environment variables.

---

## Resolution Summary

| Issue | Root Cause | Solution | Status |
|-------|-----------|----------|--------|
| mcp-docker not found | Docker executable exists but not in system PATH | Added `C:\Program Files\Docker\Docker\resources\bin` to system PATH, then restarted system | ✅ Fixed |
| mcp-searxng spawn cmd.exe ENOENT | Subprocess inherited stale PATH from LM Studio's pre-restart environment | Full system restart ensured all processes get updated PATH | ✅ Fixed |

**Both MCP tools are now fully functional and operational.**

---

## Quick Reference: Common Symptoms & Solutions

| Symptom | Likely Cause | First Thing to Check |
|--------|-------------|---------------------|
| `docker` not recognized, but Docker Desktop is running | Not in PATH | Run `C:\Program Files\Docker\...\docker.exe --version` with full path |
| Tool fails after PATH fix without OS restart | Stale environment | Full system restart required |
| WSL command returns different output than expected | Testing from wrong shell | Test from native Windows Command Prompt |

> **Pro tip:** When troubleshooting MCP tools on Windows, always verify you're testing from the correct shell (native vs. WSL) and that environment variables have been fully inherited by all processes!
