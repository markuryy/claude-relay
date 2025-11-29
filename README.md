# Claude Relay

Real-time communication between Claude Code instances across multiple machines via WebSocket + MCP.

## What This Does

Enables Claude Code sessions on different machines to send messages to each other in real-time. Useful for:
- **Context sharing** - Share findings, file contents, or investigation results between sessions
- **Task handoffs** - Start a task on one machine, continue on another
- **Coordination** - Let one Claude Code instance know what another is doing

## Architecture

```
Machine A                              Machine B (Server Host)
┌─────────────────┐                   ┌─────────────────┐
│  Claude Code    │                   │  Claude Code    │
│      ↓          │                   │      ↓          │
│  MCP Server     │                   │  MCP Server     │
│      ↓          │                   │      ↓          │
│  WebSocket  ────┼── SSH Tunnel ─────┼─→ Relay Server  │
│  (localhost)    │   or direct       │   (port 9999)   │
└─────────────────┘                   └─────────────────┘
```

## Installation

```bash
git clone https://github.com/gvorwaller/claude-relay.git
cd claude-relay
npm install
```

## Quick Start

### 1. Start the Relay Server (on one machine)

```bash
node server.js
# [Claude Relay] Ready! Listening on ws://localhost:9999
```

### 2. Configure Claude Code (on each machine)

Add to your Claude Code MCP configuration:

```json
{
  "mcpServers": {
    "claude-relay": {
      "command": "node",
      "args": ["/path/to/claude-relay/mcp-server.js"],
      "env": {
        "RELAY_CLIENT_ID": "MACHINE_A",
        "RELAY_URL": "ws://localhost:9999"
      }
    }
  }
}
```

Set a unique `RELAY_CLIENT_ID` for each machine (e.g., "LAPTOP", "DESKTOP", "M1", "M2").

### 3. Connect Remote Machines via SSH Tunnel

If machines aren't on the same network, use SSH port forwarding:

```bash
# On the remote machine, tunnel to the server host
ssh -N -L 9999:localhost:9999 server-host &

# Or use autossh for auto-reconnecting
autossh -M 0 -N -L 9999:localhost:9999 server-host &
```

## MCP Tools

Once configured, Claude Code will have these tools:

| Tool | Description |
|------|-------------|
| `relay_send` | Send a message to peer Claude Code instance(s) |
| `relay_receive` | Get recent messages from peers |
| `relay_peers` | List currently connected instances |
| `relay_status` | Check connection health |

### Example Usage

**Send a message:**
```
Use relay_send to tell MACHINE_B: "Found the bug - it's in auth.js line 42"
```

**Check for messages:**
```
Use relay_receive to see if there are any messages from peers
```

**See who's online:**
```
Use relay_peers to list connected instances
```

## macOS Auto-Start (LaunchAgent)

### Relay Server (on server host)

```bash
# Copy the LaunchAgent
cp com.claude-relay.plist ~/Library/LaunchAgents/

# Edit the plist to fix paths for your system:
# - Update /usr/local/bin/node to your node path (use `which node`)
# - Update /Users/yourname/claude-relay to your install path

# Load it
launchctl load ~/Library/LaunchAgents/com.claude-relay.plist
```

### SSH Tunnel (on remote machines)

```bash
# Install autossh
brew install autossh

# Copy and edit the tunnel LaunchAgent
cp com.claude-relay-tunnel.plist ~/Library/LaunchAgents/

# Edit to set your server hostname and paths

# Load it
launchctl load ~/Library/LaunchAgents/com.claude-relay-tunnel.plist
```

## Testing

Use the interactive test client:

```bash
# Terminal 1: Start server
node server.js

# Terminal 2: Connect as client A
node test-client.js MACHINE_A

# Terminal 3: Connect as client B
node test-client.js MACHINE_B

# In either client:
send Hello from here!
peers
history
```

## Configuration

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `RELAY_PORT` | `9999` | Port for relay server |
| `RELAY_CLIENT_ID` | hostname | Identifier for this instance |
| `RELAY_URL` | `ws://localhost:9999` | Relay server WebSocket URL |

### Command Line Arguments

```bash
# Server
node server.js [port]
node server.js 8888

# MCP Server
node mcp-server.js --client-id=LAPTOP --relay-url=ws://192.168.1.100:9999
```

## Files

| File | Description |
|------|-------------|
| `server.js` | WebSocket relay server |
| `mcp-server.js` | MCP protocol server for Claude Code |
| `test-client.js` | Interactive test client |
| `com.claude-relay.plist` | macOS LaunchAgent for relay server |
| `com.claude-relay-tunnel.plist` | macOS LaunchAgent for SSH tunnel |

## Troubleshooting

**Connection refused:**
- Ensure relay server is running: `lsof -i :9999`
- If using SSH tunnel, verify it's active: `ps aux | grep ssh`

**MCP tools not appearing:**
- Restart Claude Code after adding MCP config
- Check MCP server is connecting: look for "Connected!" in logs

**Messages not arriving:**
- Use `relay_peers` to verify both instances are connected
- Check message history with `relay_receive`

## Security Notes

- The relay server has no authentication by default
- Designed for trusted local networks or SSH tunnels
- All traffic over SSH tunnel is encrypted
- Don't expose port 9999 to the internet without adding authentication

## License

MIT
