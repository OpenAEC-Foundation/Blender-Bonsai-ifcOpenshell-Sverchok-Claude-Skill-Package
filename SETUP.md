# Setup Guide: Blender MCP + Claude Code + BIM Stack

Complete setup instructions for connecting Claude Code to Blender via MCP, with Bonsai, IfcOpenShell and Sverchok.

## Architecture

```
Claude Code (CLI)
    |
    | MCP Protocol (stdio)
    |
uvx blender-mcp (MCP Server)
    |
    | TCP Socket (localhost:9876)
    |
Blender Addon (addon.py)
    |
    | bpy Python API
    |
Blender Scene (with Bonsai, IfcOpenShell, Sverchok)
```

- **Claude Code** sends MCP tool calls (`get_scene_info`, `execute_blender_code`, etc.)
- **blender-mcp server** translates these to JSON commands over TCP
- **Blender addon** receives JSON, executes Python via `bpy`, returns results
- **Skills** (in `.claude/skills/`) give Claude the domain knowledge to generate correct BIM/IFC code

## Prerequisites

| Component | Minimum version | How to check |
|-----------|----------------|--------------|
| Blender | 3.6+ (4.x/5.x recommended) | Blender > Help > About |
| Python | 3.10+ | `python --version` |
| Node.js | 18+ | `node --version` |
| uv package manager | Latest | `uv --version` |
| Claude Code CLI | Latest | `claude --version` |

## Step 1: Install Node.js

Required for Claude Code CLI.

1. Download Node.js (LTS) from https://nodejs.org/
2. Run the installer (default settings are fine)
3. Restart your terminal

```bash
# Verify
node --version
npm --version
```

## Step 2: Install Claude Code CLI

```bash
npm install -g @anthropic-ai/claude-code
```

You need an Anthropic account with an active subscription (Claude Pro, Team, or Enterprise).

```bash
# Verify
claude --version
```

## Step 3: Install uv package manager

uv is required to run the MCP server via `uvx`.

### Windows (PowerShell as Administrator)

```powershell
powershell -c "irm https://astral.sh/uv/install.ps1 | iex"
```

Add to PATH if not done automatically:

```powershell
$localBin = "$env:USERPROFILE\.local\bin"
$userPath = [Environment]::GetEnvironmentVariable("Path", "User")
[Environment]::SetEnvironmentVariable("Path", "$userPath;$localBin", "User")
```

**Restart your terminal** after installation.

### macOS

```bash
brew install uv
```

### Linux

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

```bash
# Verify
uv --version
uvx --version
```

## Step 4: Install Blender

Download from https://www.blender.org/download/. Version 3.6 or newer is required, 4.x or 5.x is recommended for full compatibility with all skills.

## Step 5: Install the Blender MCP addon

The addon creates a TCP server inside Blender that Claude Code connects to.

**Source:** https://github.com/ahujasid/blender-mcp (~17.5k stars, the standard Blender MCP implementation)

### Option A: Install from ZIP (recommended)

1. Go to https://github.com/ahujasid/blender-mcp
2. Click **Code > Download ZIP**
3. Extract the ZIP
4. Open Blender
5. Go to **Edit > Preferences > Add-ons**
6. Click **Install from Disk...**
7. Select the extracted `blender-mcp-main` folder (or `addon.py` directly)
8. Enable the addon: search for "BlenderMCP" and check the box

### Option B: Copy addon.py directly

1. Download `addon.py` from https://raw.githubusercontent.com/ahujasid/blender-mcp/main/addon.py
2. Copy to your Blender addons folder:
   - **Windows**: `%APPDATA%\Blender Foundation\Blender\<version>\scripts\addons\`
   - **macOS**: `~/Library/Application Support/Blender/<version>/scripts/addons/`
   - **Linux**: `~/.config/blender/<version>/scripts/addons/`
3. Restart Blender
4. Go to **Edit > Preferences > Add-ons**
5. Search for "BlenderMCP" and enable it

### Verify

After activation, a **BlenderMCP** tab appears in the 3D Viewport sidebar (press `N` to open the sidebar).

## Step 6: Install Bonsai addon

Bonsai (formerly BlenderBIM) is the native IFC authoring addon for Blender. It automatically installs IfcOpenShell as a dependency.

1. Go to https://blenderbim.org/download.html
2. Download the version matching your Blender version and OS
3. Open Blender
4. Go to **Edit > Preferences > Add-ons**
5. Click **Install from Disk...**
6. Select the downloaded `.zip` file
7. Enable: search for "Bonsai" (or "BlenderBIM") and check the box

### Verify

Open the Blender Python Console (Scripting workspace):

```python
import ifcopenshell
print(ifcopenshell.version)
```

This should print a version number. If it does, both Bonsai and IfcOpenShell are installed correctly.

## Step 7: Install Sverchok (optional)

Sverchok is a visual programming addon for parametric design. Only needed for parametric/generative workflows.

### Blender 4.2+

1. Open Blender
2. Go to **Edit > Preferences > Get Extensions**
3. Search for "Sverchok"
4. Click **Install**

### Older versions / manual install

1. Go to https://github.com/nortikin/sverchok
2. Download as ZIP (Code > Download ZIP)
3. Open Blender > Edit > Preferences > Add-ons > Install from Disk...
4. Select the ZIP, enable the addon

### Verify

After activation, a **Sverchok** menu appears in the Node Editor. Create a new node tree via **Add > Sverchok Node Tree** to test.

## Step 8: Install the skill package

The skills give Claude deep knowledge of Blender, Bonsai, IfcOpenShell, and Sverchok APIs.

### Option A: Copy skills into your project

```bash
git clone https://github.com/OpenAEC-Foundation/Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package.git

# Copy all skills
cp -r Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package/skills/ your-project/.claude/skills/

# Copy the MCP config
cp Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package/.mcp.json your-project/.mcp.json
```

### Option B: Install only what you need

```bash
# Only Blender skills
cp -r skills/blender/ your-project/.claude/skills/blender/

# Only IfcOpenShell skills
cp -r skills/ifcopenshell/ your-project/.claude/skills/ifcopenshell/

# Only Bonsai skills
cp -r skills/bonsai/ your-project/.claude/skills/bonsai/

# Only Sverchok skills
cp -r skills/sverchok/ your-project/.claude/skills/sverchok/

# Always copy the MCP config for Blender connectivity
cp .mcp.json your-project/.mcp.json
```

### Option C: Use OpenAEC Workspace Composer

The [OpenAEC Workspace Composer](https://github.com/OpenAEC-Foundation/OpenAEC-Workspace-Composer) can set up a complete workspace with skills and MCP configuration automatically.

## Step 9: MCP configuration

The `.mcp.json` file in the repository root tells Claude Code which MCP servers are available:

```json
{
  "mcpServers": {
    "blender": {
      "command": "uvx",
      "args": ["blender-mcp"],
      "env": {
        "BLENDER_PORT": "9876",
        "DISABLE_TELEMETRY": "true"
      }
    }
  }
}
```

This file is already included in the repository. When you clone or copy the repo, Claude Code automatically detects it.

### Alternative: add via CLI

```bash
claude mcp add blender -- uvx blender-mcp
```

### Windows: if uvx is not found

Use the full path in `.mcp.json`:

```json
{
  "mcpServers": {
    "blender": {
      "command": "C:\\Users\\<your-username>\\.local\\bin\\uvx.exe",
      "args": ["blender-mcp"],
      "env": {
        "BLENDER_PORT": "9876",
        "DISABLE_TELEMETRY": "true"
      }
    }
  }
}
```

## Step 10: Connect and verify

### Every time you want to work

1. **Start Blender**
2. Open the **3D Viewport sidebar** (press `N`)
3. Click the **BlenderMCP** tab
4. Click **"Connect to Claude"** (you should see: `Server running on port 9876`)
5. Start Claude Code in your workspace:
   ```bash
   cd your-project
   claude
   ```
6. Claude automatically detects the `.mcp.json` and connects to Blender

### Test the connection

Ask Claude:

```
Show me the current Blender scene
```

Claude should:
1. Call `get_scene_info` (fetch scene data)
2. Call `get_viewport_screenshot` (capture the viewport)
3. Tell you what is in the scene

## Available MCP Tools

When connected, Claude has access to these Blender MCP tools:

| Tool | Purpose |
|------|---------|
| `get_scene_info` | Read current scene state (objects, materials, settings) |
| `get_object_info` | Get details of a specific object |
| `execute_blender_code` | Run Python code directly in Blender |
| `get_viewport_screenshot` | Capture the 3D viewport as an image |
| `search_polyhaven_assets` | Search Poly Haven for HDRIs, textures, models |
| `download_polyhaven_asset` | Download and import Poly Haven assets |
| `search_sketchfab_models` | Search Sketchfab for 3D models |
| `download_sketchfab_model` | Download and import Sketchfab models |

### Mandatory workflow cycle

ALWAYS follow this cycle for every Blender interaction:

```
1. get_scene_info          -> understand current state
2. execute_blender_code    -> make changes
3. get_viewport_screenshot -> verify visually
```

## Troubleshooting

### "Could not connect to Blender"

- Is Blender open?
- Is "Connect to Claude" clicked in the BlenderMCP tab?
- Is another process using port 9876? Check: `netstat -an | grep 9876`

### "uvx not found"

- Is uv installed? Run `uv --version`
- Is `~/.local/bin` in your PATH?
- Restart your terminal after installing uv

### First command fails

Known issue. The first command after connecting may fail. Simply try again.

### Blender becomes unresponsive

- The addon runs on the main thread. Heavy operations temporarily block the UI
- If Blender crashes: restart Blender, re-enable the addon, click "Connect to Claude"
- Always save your work before starting a Claude Code session (`Ctrl+S`)

## Security note

The `execute_blender_code` tool runs arbitrary Python code in Blender. This is powerful but carries risk:

- Always save your work before starting
- Claude Code asks permission for each MCP tool call by default
- Review the code Claude wants to execute before approving
