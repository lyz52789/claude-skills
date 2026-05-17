---
name: mcp-connection-guard
description: Use when MCP tools fail with "API Error: 400", "MCP server disconnected", or deferred tools are unavailable. Also run at session start to verify MCP health.
---

# MCP Connection Guard

## Overview

Ensure Claude Code and MCP servers (especially mempalace) stay connected and communicate correctly. Detects protocol mismatches, stale processes, and disconnected deferred tools. Auto-repairs common failure modes.

## When to Use

- **Session start**: Verify MCP servers are healthy before relying on them
- **After "API Error: 400 Invalid request Error"** on MCP tool calls
- **After "MCP server disconnected"** warnings
- **After `pip install -U mempalace`**: Protocol patches may need reapplication
- **When deferred tools vanish** from the tool list

**Do NOT use for:** Non-MCP tools (native Bash/Read/Edit), or HTTP MCP servers using SSE transport.

## Core Pattern

```
Detect → Diagnose → Kill Stale → Re-register → Verify
```

### 1. Detect

Check which MCP servers are registered and their actual health:

```bash
claude mcp list
```

Also check for zombie processes:

```bash
ps aux | grep -i mempalace | grep -v grep
```

Red flags:
- `✗ Disconnected` or missing from `claude mcp list`
- Processes in `T` (stopped/trace) state
- Multiple stale `python -m mempalace.mcp_server` processes

### 2. Diagnose

**If mempalace MCP server starts but tools don't register**, check protocol compatibility:

```bash
# Test if server speaks Content-Length (standard MCP stdio)
python3 -c "
import subprocess, json, time
proc = subprocess.Popen(['mempalace-mcp'], stdin=subprocess.PIPE, stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True)
time.sleep(2)
req = json.dumps({'jsonrpc': '2.0', 'id': 1, 'method': 'initialize', 'params': {'protocolVersion': '2024-11-05', 'capabilities': {}, 'clientInfo': {'name': 'test', 'version': '1.0'}}})
proc.stdin.write(f'Content-Length: {len(req.encode(\"utf-8\"))}\\r\\n\\r\\n{req}')
proc.stdin.flush()
h = proc.stdout.readline()
b = proc.stdout.readline()
body = proc.stdout.readline()
print('HEADER:', h.strip())
print('BODY:', body.strip()[:200])
proc.terminate()
"
```

If the header starts with `Content-Length:`, the server speaks standard MCP protocol. If only line-based JSON works, the server needs patching.

### 3. Kill Stale

```bash
# Kill all stopped/traced mempalace MCP processes
ps aux | grep 'mempalace.mcp_server' | grep -v grep | awk '{print $2}' | xargs -r kill -9

# Also clear lock files if mine was interrupted
rm -f ~/.mempalace/locks/mine_palace_*.lock
```

### 4. Re-register

```bash
# Remove broken entry
claude mcp remove mempalace

# Re-add (picks up the patched server automatically)
claude mcp add mempalace -- mempalace-mcp
```

### 5. Verify

```bash
# Confirm server is connected
claude mcp list

# Try a lightweight tool call
mempalace status
```

**Important**: If deferred tools still show as unavailable after re-registering, **start a new Claude Code session**. Deferred tools are discovered at session startup.

## The Protocol Patch (mempalace-specific)

MemPalace 3.3.5's MCP server uses line-based JSON instead of standard MCP Content-Length framing. This causes Claude Code to send requests the server cannot parse, resulting in silent failures or 400 errors.

**Patch location**: `mempalace/mcp_server.py` main loop (around line 2296)

Replace the raw `readline()` + `json.loads()` loop with auto-detecting `_read_message()` and `_write_message()` helpers that support both Content-Length and line-based protocols.

**Auto-patch script** (save as `~/.local/bin/fix-mempalace-mcp`):

```bash
#!/bin/bash
# fix-mempalace-mcp — Auto-patch mempalace MCP server for Content-Length protocol

SITE_PACKAGES=$(python3 -c "import mempalace, os, inspect; print(os.path.dirname(inspect.getfile(mempalace)))")
TARGET="$SITE_PACKAGES/mcp_server.py"

if [[ ! -f "$TARGET" ]]; then
    echo "ERROR: mempalace not found"
    exit 1
fi

# Check if already patched
if grep -q "_use_content_length_protocol" "$TARGET"; then
    echo "Already patched"
    exit 0
fi

echo "Patching $TARGET ..."

python3 << 'PYEOF'
import re, sys

path = sys.argv[1]
with open(path, 'r') as f:
    content = f.read()

# Find the main() function and replace its while loop
old_loop = '''    while True:
        try:
            line = sys.stdin.readline()
            if not line:
                break
            line = line.strip()
            if not line:
                continue
            request = json.loads(line)
            response = handle_request(request)
            if response is not None:
                sys.stdout.write(json.dumps(response, ensure_ascii=False) + "\\n")
                sys.stdout.flush()
        except KeyboardInterrupt:
            break
        except Exception as e:
            logger.error(f"Server error: {e}")'''

new_helpers = '''# Protocol auto-detection: None=undetected, True=Content-Length, False=line-based
_use_content_length_protocol = None


def _read_message():
    """Read one JSON-RPC message from stdin.

    Supports both standard MCP Content-Length protocol and legacy line-based
    protocol.  Auto-detects the protocol based on the first non-empty line.
    """
    global _use_content_length_protocol

    line = sys.stdin.readline()
    if not line:
        return None  # EOF

    line = line.strip()
    if not line:
        return _read_message()  # Skip empty lines between messages

    if line.lower().startswith("content-length:"):
        _use_content_length_protocol = True
        try:
            content_length = int(line.split(":", 1)[1].strip())
        except (ValueError, IndexError):
            _use_content_length_protocol = False
            return json.loads(line)

        sys.stdin.readline()  # blank separator
        body = sys.stdin.read(content_length)
        return json.loads(body)

    if _use_content_length_protocol is None:
        _use_content_length_protocol = False
    return json.loads(line)


def _write_message(response):
    """Write a JSON-RPC response to stdout."""
    body = json.dumps(response, ensure_ascii=False)

    if _use_content_length_protocol is not False:
        body_bytes = body.encode("utf-8")
        sys.stdout.write(f"Content-Length: {len(body_bytes)}\\r\\n\\r\\n")
        sys.stdout.write(body)
        sys.stdout.write("\\n")
    else:
        sys.stdout.write(body + "\\n")
    sys.stdout.flush()


''' + old_loop.replace(
    '''            line = sys.stdin.readline()
            if not line:
                break
            line = line.strip()
            if not line:
                continue
            request = json.loads(line)''',
    '''            request = _read_message()
            if request is None:
                break'''
).replace(
    '''                sys.stdout.write(json.dumps(response, ensure_ascii=False) + "\\n")
                sys.stdout.flush()''',
    '''                _write_message(response)'''
)

if old_loop not in content:
    print("ERROR: Could not find the loop to patch. Manual intervention needed.")
    sys.exit(1)

content = content.replace(old_loop, new_helpers)

with open(path, 'w') as f:
    f.write(content)

print("Patch applied successfully.")
PYEOF "$TARGET"

echo "Done. Restart Claude Code session to pick up changes."
```

Make it executable and run after every mempalace update:

```bash
chmod +x ~/.local/bin/fix-mempalace-mcp
fix-mempalace-mcp
```

## Quick Reference

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `API Error: 400` on MCP call | Protocol mismatch | Run `fix-mempalace-mcp`, restart session |
| `✗ Disconnected` in `claude mcp list` | Server crashed | Kill stale processes, re-register |
| Deferred tools "no longer available" | Session lost connection | Start new Claude Code session |
| Multiple `T` state mempalace processes | SIGSTOP or debugger attach | `kill -9` stale PIDs |
| `mempalace status` works but MCP fails | Hooks OK, MCP layer broken | Patch server, re-register |

## SessionStart Hook (Optional)

Add to `~/.claude/settings.json` for automatic health checks on every session start:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'if command -v fix-mempalace-mcp >/dev/null 2>&1; then fix-mempalace-mcp >/dev/null 2>&1; fi'",
            "shell": "bash"
          }
        ]
      }
    ]
  }
}
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Re-registering without restarting session | Deferred tools stay unavailable — always start a new session after re-register |
| Patching but not killing stale processes | Old `T` state processes interfere — kill them first |
| Assuming `claude mcp list` ✓ means tools work | Health check only verifies process exists, not protocol compatibility |
| Patching the wrong Python environment | Use `python3 -m site` to confirm which site-packages is active |
