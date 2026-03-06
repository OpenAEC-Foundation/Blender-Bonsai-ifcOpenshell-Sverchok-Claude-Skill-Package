# Blender MCP Server Integration with Claude - Research Report

**Date:** 2026-03-06
**Researcher:** research-claude-md agent

---

## Table of Contents

1. [Overview of Blender MCP Servers](#1-overview-of-blender-mcp-servers)
2. [ahujasid/blender-mcp (Primary)](#2-ahujasidblender-mcp-primary)
3. [poly-mcp/Blender-MCP-Server](#3-poly-mcpblender-mcp-server)
4. [CommonSenseMachines/blender-mcp](#4-commonsensemachinesblender-mcp)
5. [dhakalnirajan/blender-open-mcp](#5-dhakalnirajnblender-open-mcp)
6. [Architecture and Protocol Details](#6-architecture-and-protocol-details)
7. [Complete Tool Reference (ahujasid)](#7-complete-tool-reference-ahujasid)
8. [Installation and Configuration](#8-installation-and-configuration)
9. [Claude Code Skills and MCP Tool Interaction](#9-claude-code-skills-and-mcp-tool-interaction)
10. [Limitations and Requirements](#10-limitations-and-requirements)
11. [OpenAEC Foundation AI-deployment-repo](#11-openaec-foundation-ai-deployment-repo)
12. [Sources](#12-sources)

---

## 1. Overview of Blender MCP Servers

Multiple Blender MCP server implementations exist, each with different capabilities and target audiences. All share the same fundamental concept: bridging Blender's Python API with AI language models through the Model Context Protocol (MCP).

| Project | Author | Stars | Key Differentiator |
|---------|--------|-------|-------------------|
| `ahujasid/blender-mcp` | Siddharth Ahuja | ~17.5k | Primary/most popular, Claude-native, Poly Haven + Hyper3D + Sketchfab |
| `poly-mcp/Blender-MCP-Server` | PolyMCP team | — | 51 tools, FastAPI-based HTTP, enterprise-grade queue system |
| `CommonSenseMachines/blender-mcp` | CSM.ai | — | "Text to 4D Worlds", CSM.ai asset generation integration |
| `dhakalnirajan/blender-open-mcp` | Nirajan Dhakal | — | Uses Ollama for local LLMs instead of Claude |
| `skyiron/blender-mcp-claude` | skyiron | — | Fork/variant of ahujasid's original |
| `VxASI/blender-mcp-vxai` | VxASI | — | Natural language control variant |

---

## 2. ahujasid/blender-mcp (Primary)

**Repository:** https://github.com/ahujasid/blender-mcp
**Current Version:** 1.5.5
**Author:** Siddharth Ahuja (@sidahuj)
**Website:** https://blender-mcp.com/ (unofficial sites exist -- the author warns against them)

This is the primary and most widely-used Blender MCP implementation. It has approximately 17,500 GitHub stars and is the de-facto standard for connecting Claude to Blender.

### Key Features (v1.5.5)

- Two-way communication between Claude AI and Blender via socket-based server
- Object manipulation: create, modify, delete 3D objects
- Material control: apply and modify materials and colors
- Scene inspection: detailed information about the current Blender scene
- Code execution: run arbitrary Python code in Blender from Claude
- Poly Haven asset integration (HDRIs, textures, models)
- Hyper3D Rodin AI-generated 3D models
- Hunyuan3D AI-generated 3D models
- Sketchfab model search and download
- Viewport screenshot capture
- Remote host execution support
- Telemetry (anonymous, opt-out available)

---

## 3. poly-mcp/Blender-MCP-Server

**Repository:** https://github.com/poly-mcp/Blender-MCP-Server

A more enterprise-oriented implementation that transforms Blender into an MCP server with 50+ tools accessible through HTTP endpoints.

### Key Differences from ahujasid/blender-mcp

- Uses **FastAPI + Uvicorn** (HTTP REST API) instead of raw TCP sockets
- Provides **51 tools** across 13 categories
- Thread-safe execution queue for concurrent operations
- Real-time monitoring dashboard
- Automatic dependency installation
- Built for the PolyMCP framework

### Tool Categories

| Category | Description |
|----------|-------------|
| Object Operations | Create, delete, duplicate, select objects |
| Transformations | Move, rotate, scale, apply transforms |
| Materials & Shading | Create materials, add textures, setup shader nodes |
| Modeling | Add modifiers, boolean operations, mesh editing |
| Animation | Keyframes, timeline control, NLA editor |
| Camera & Lighting | Camera setup, light creation, HDRI environments |
| Rendering | Render settings, output configuration, rendering |
| Physics | Rigid body, cloth, fluid simulations |
| Geometry Nodes | Procedural generation, node tree creation |
| File Operations | Import/export FBX, OBJ, USD, etc. |
| Scene Management | Scene info, cleanup, optimization |
| Batch Operations | Multi-object creation and transformation |
| Advanced | Particle systems, force fields, grease pencil |

### Configuration

```
HOST = "0.0.0.0"
PORT = 8000
AUTO_INSTALL_PACKAGES = True
THREAD_SAFE_OPERATIONS = True
ENABLE_CACHING = True
```

**Requirements:** Blender 3.0.0+, FastAPI, Uvicorn, Pydantic, docstring-parser, NumPy

---

## 4. CommonSenseMachines/blender-mcp

**Repository:** https://github.com/CommonSenseMachines/blender-mcp
**Name:** "Text to 4D Worlds in Blender"

A three-part integration connecting Blender, CSM.ai (asset generation platform), and LLM agents.

### Architecture

- **CSM.ai** = Asset generator and manager (vector search-based 3D model retrieval)
- **MCP Server** = Bridge between LLM and Blender
- **Blender** = Execution environment

### Key Features

- CSM.ai asset search and retrieval (public and private sessions)
- Humanoid animation via Mixamo
- Text and image-based editing capabilities
- Interactive decoupled architecture

### Installation

```bash
git clone https://github.com/CommonSenseMachines/blender-mcp.git
pip install -e .
```

Requires CSM API credentials configured in the addon settings.

---

## 5. dhakalnirajan/blender-open-mcp

**Repository:** https://github.com/dhakalnirajan/blender-open-mcp

An open-source alternative that uses **Ollama** for locally-running LLMs instead of cloud-based Claude.

### Key Features

- Local AI models (llama3.2, Gemma3, etc.) via Ollama
- No cloud API subscription required
- MCP-structured communication
- PolyHaven asset integration
- Basic 3D operations (create, modify, render)

### Requirements

- Blender 3.0+, Ollama installed, Python 3.10+

---

## 6. Architecture and Protocol Details

### Communication Architecture (ahujasid/blender-mcp)

```
┌──────────────────┐     MCP Protocol      ┌──────────────────┐     TCP Socket     ┌──────────────────┐
│                  │  ◄──────────────────►  │                  │  ◄──────────────►  │                  │
│  Claude Desktop  │                        │    MCP Server    │                    │  Blender Addon   │
│  or Claude Code  │                        │  (server.py)     │                    │  (addon.py)      │
│                  │                        │                  │                    │                  │
│  Natural language│                        │  Translates MCP  │                    │  TCP socket      │
│  commands from   │                        │  tool calls to   │                    │  server on       │
│  user            │                        │  JSON commands   │                    │  port 9876       │
│                  │                        │  over TCP        │                    │                  │
└──────────────────┘                        └──────────────────┘                    └──────────────────┘
```

### Two-Component System

1. **Blender Addon (`addon.py`):**
   - Registers as a Blender addon under "Interface: Blender MCP"
   - Creates a TCP socket server within Blender (default: `localhost:9876`)
   - Receives JSON commands and executes them using Blender's Python API (`bpy`)
   - Runs commands in Blender's main thread for thread safety
   - Accessible from Blender's N-panel sidebar under "BlenderMCP" tab

2. **MCP Server (`src/blender_mcp/server.py`):**
   - Python server implementing the Model Context Protocol
   - Connects to the Blender addon via TCP socket
   - Exposes MCP tools that Claude can invoke
   - Launched via `uvx blender-mcp` (using the `uv` package manager)

### Communication Protocol

The system uses a JSON-based protocol over TCP sockets:

**Commands** (sent to Blender):
```json
{
    "type": "command_name",
    "params": {
        "key": "value"
    }
}
```

**Responses** (returned from Blender):
```json
{
    "status": "success",
    "result": { ... }
}
```

Or on error:
```json
{
    "status": "error",
    "message": "Error description"
}
```

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `BLENDER_HOST` | `localhost` | Host address for Blender socket server |
| `BLENDER_PORT` | `9876` | Port for Blender socket server |
| `DISABLE_TELEMETRY` | `false` | Set to `true` to disable all telemetry |

---

## 7. Complete Tool Reference (ahujasid/blender-mcp v1.5.5)

These are the 22 MCP tools defined in `server.py`, extracted directly from the source code:

### Scene & Object Tools

| Tool | Description | Parameters |
|------|-------------|------------|
| `get_scene_info` | Get detailed information about the current Blender scene | None |
| `get_object_info` | Get detailed information about a specific object | `object_name` (string) |
| `get_viewport_screenshot` | Capture screenshot of 3D viewport | `max_size` (int, default 800) |
| `execute_blender_code` | Execute arbitrary Python code in Blender (step-by-step) | `code` (string) |

### Poly Haven Integration

| Tool | Description | Parameters |
|------|-------------|------------|
| `get_polyhaven_status` | Check if PolyHaven integration is enabled | None |
| `get_polyhaven_categories` | Get categories for a specific asset type | `asset_type` (string: hdris/textures/models/all) |
| `search_polyhaven_assets` | Search for assets with optional filtering | `asset_type`, `categories` (optional) |
| `download_polyhaven_asset` | Download and import a Polyhaven asset | `asset_id`, `asset_type`, `resolution`, `file_format` (optional) |
| `set_texture` | Apply a downloaded Polyhaven texture to an object | `object_name`, `texture_id` |

### Sketchfab Integration

| Tool | Description | Parameters |
|------|-------------|------------|
| `get_sketchfab_status` | Check if Sketchfab integration is enabled | None |
| `search_sketchfab_models` | Search for models with optional filtering | `query`, `categories` (optional), `count` (default 20), `downloadable` (default true) |
| `get_sketchfab_model_preview` | Get preview thumbnail of a model | `uid` (string) |
| `download_sketchfab_model` | Download and import a Sketchfab model | `uid`, `target_size` (required, in Blender units) |

### Hyper3D Rodin Integration

| Tool | Description | Parameters |
|------|-------------|------------|
| `get_hyper3d_status` | Check if Hyper3D Rodin integration is enabled | None |
| `generate_hyper3d_model_via_text` | Generate 3D asset from text description | `text_prompt` (English), `bbox_condition` (optional list) |
| `generate_hyper3d_model_via_images` | Generate 3D asset from image references | `input_image_paths` or `input_image_urls`, `bbox_condition` (optional) |
| `poll_rodin_job_status` | Check if generation task is completed | `subscription_key` (MAIN_SITE) or `request_id` (FAL_AI) |
| `import_generated_asset` | Import generated asset after completion | `name`, `task_uuid` or `request_id` |

### Hunyuan3D Integration

| Tool | Description | Parameters |
|------|-------------|------------|
| `get_hunyuan3d_status` | Check if Hunyuan3D integration is enabled | None |
| `generate_hunyuan3d_model` | Generate 3D asset via text or image | `text_prompt` (optional), `input_image_url` (optional) |
| `poll_hunyuan_job_status` | Check if generation task is completed | `job_id` (string) |
| `import_generated_asset_hunyuan` | Import generated Hunyuan3D asset | `name`, `zip_file_url` (string) |

---

## 8. Installation and Configuration

### Prerequisites

- **Blender:** 3.0 or newer (3.6+ recommended for full functionality)
- **Python:** 3.10 or newer
- **uv package manager:** Required for running the MCP server

### Step 1: Install uv Package Manager

**macOS:**
```bash
brew install uv
```

**Windows (PowerShell):**
```powershell
powershell -c "irm https://astral.sh/uv/install.ps1 | iex"

# Add to PATH
$localBin = "$env:USERPROFILE\.local\bin"
$userPath = [Environment]::GetEnvironmentVariable("Path", "User")
[Environment]::SetEnvironmentVariable("Path", "$userPath;$localBin", "User")
```

**Linux/Other:** See https://docs.astral.sh/uv/getting-started/installation/

### Step 2: Install the Blender Addon

1. Download `addon.py` from https://github.com/ahujasid/blender-mcp
2. Open Blender
3. Go to **Edit > Preferences > Add-ons**
4. Click **"Install..."** and select the `addon.py` file
5. Enable the addon by checking **"Interface: Blender MCP"**

### Step 3: Configure Claude

#### Claude Desktop

Edit `claude_desktop_config.json` (via Claude > Settings > Developer > Edit Config):

```json
{
    "mcpServers": {
        "blender": {
            "command": "uvx",
            "args": [
                "blender-mcp"
            ]
        }
    }
}
```

#### Claude Code (CLI)

```bash
claude mcp add blender uvx blender-mcp
```

#### Windows (Claude Desktop or Cursor)

```json
{
    "mcpServers": {
        "blender": {
            "command": "cmd",
            "args": [
                "/c",
                "uvx",
                "blender-mcp"
            ]
        }
    }
}
```

#### With Telemetry Disabled

```json
{
    "mcpServers": {
        "blender": {
            "command": "uvx",
            "args": ["blender-mcp"],
            "env": {
                "DISABLE_TELEMETRY": "true"
            }
        }
    }
}
```

#### With Remote Blender Host

```json
{
    "mcpServers": {
        "blender": {
            "command": "uvx",
            "args": ["blender-mcp"],
            "env": {
                "BLENDER_HOST": "192.168.1.100",
                "BLENDER_PORT": "9876"
            }
        }
    }
}
```

### Step 4: Start the Connection in Blender

1. In Blender, go to the 3D View sidebar (press **N** if not visible)
2. Find the **"BlenderMCP"** tab
3. Optionally enable the **Poly Haven** checkbox for asset integration
4. Click **"Connect to Claude"**

### Important Notes

- Only run **one instance** of the MCP server (either Claude Desktop or Cursor, not both)
- The first command may not go through; subsequent commands should work
- Always save your work before using `execute_blender_code`

---

## 9. Claude Code Skills and MCP Tool Interaction

### How Skills and MCP Tools Relate

Based on Anthropic's official documentation and community resources:

**MCP servers** provide **connectivity** -- they give Claude access to external systems, services, and platforms (like Blender).

**Skills** provide **expertise** -- they give Claude the domain knowledge, procedural instructions, and workflow logic needed to use those connections effectively.

### Key Principles

1. **A single skill can orchestrate multiple MCP servers.** For example, a "create BIM visualization" skill could use both a Blender MCP server and a file system MCP server.

2. **A single MCP server can support dozens of different skills.** The Blender MCP server could be used by skills for architectural visualization, game asset creation, IFC model inspection, etc.

3. **Skills activate contextually.** When Claude receives a task, it reviews available skill descriptions. If a skill's `description` field matches the task context, Claude loads the full skill instructions and applies them.

4. **MCP tools appear as available tools.** When an MCP server is connected, its tools (like `get_scene_info`, `execute_blender_code`) appear alongside Claude's built-in tools. Skills can reference these tools in their instructions.

### Practical Interaction Pattern

```
User Request
    │
    ▼
Claude Code loads matching Skill (from SKILL.md)
    │
    ▼
Skill instructions reference MCP tools
    │
    ▼
Claude invokes MCP tools (e.g., blender-mcp tools)
    │
    ▼
MCP Server translates to Blender commands
    │
    ▼
Blender executes and returns results
```

### Example: A Blender Skill Using MCP Tools

A skill for creating architectural scenes could contain instructions like:

```markdown
# SKILL.md
name: create-architectural-scene
description: Create architectural 3D scenes in Blender from natural language descriptions

## Instructions
1. First call `get_scene_info` to understand the current scene state
2. Use `execute_blender_code` to set up the scene with proper units (meters)
3. Search for architectural assets using `search_polyhaven_assets`
4. Download and place assets using `download_polyhaven_asset`
5. Apply materials using `set_texture`
6. Set up camera and lighting using `execute_blender_code`
```

### Important Considerations for Our Skills Package

- Skills should **assume MCP tools are available** when the server is connected, but should also **handle gracefully** when they are not
- Skills provide the **"what to do"** while MCP provides the **"how to connect"**
- Skills can reference MCP tool names directly in their instructions
- The combination enables domain-specific workflows that leverage Blender's full capabilities

---

## 10. Limitations and Requirements

### Technical Limitations

1. **`execute_blender_code` security risk:** Allows running arbitrary Python code in Blender. Always save work before using it. Not recommended for production environments without safeguards.

2. **Single instance only:** Only one MCP server instance should run at a time (either Claude Desktop or Cursor, not both).

3. **Complex operations:** Complex tasks may need to be broken down into smaller steps. Claude can sometimes be erratic with behavior.

4. **Poly Haven downloads:** Assets require downloading models, textures, and HDRI images. This can be slow and consume disk space.

5. **Hyper3D rate limits:** Free trial key allows limited generations per day. For more, you need your own API key from hyper3d.ai and fal.ai.

6. **Connection reliability:** The first command may not go through. Sometimes restarting both Claude and the Blender server is needed.

7. **Version compatibility:** Full functionality requires Blender 3.6+; basic features work with Blender 3.0+.

### System Requirements

| Requirement | Minimum |
|-------------|---------|
| Blender | 3.0 (3.6+ recommended) |
| Python | 3.10+ |
| uv package manager | Required |
| Claude API access | Required (subscription) |
| Network | Required for Poly Haven, Sketchfab, Hyper3D |

### Known Issues

- Connection issues on first command
- Timeout errors with complex requests
- Poly Haven integration can be erratic
- Windows may need additional PATH configuration for `uv`

---

## 11. OpenAEC Foundation AI-deployment-repo

**URL:** https://github.com/OpenAEC-Foundation/AI-deployment-repo

**Status:** Returns 404 (Not Found). The repository either does not exist, has been made private, or has been renamed/moved. No MCP configurations were found at this location.

---

## 12. Sources

### Primary Repositories
- [ahujasid/blender-mcp](https://github.com/ahujasid/blender-mcp) - Primary Blender MCP integration (17.5k stars)
- [poly-mcp/Blender-MCP-Server](https://github.com/poly-mcp/Blender-MCP-Server) - 51-tool enterprise Blender MCP server
- [CommonSenseMachines/blender-mcp](https://github.com/CommonSenseMachines/blender-mcp) - Text to 4D Worlds
- [dhakalnirajan/blender-open-mcp](https://github.com/dhakalnirajan/blender-open-mcp) - Ollama-based local LLM variant
- [skyiron/blender-mcp-claude](https://github.com/skyiron/blender-mcp-claude) - Fork variant

### Documentation and Articles
- [Blender MCP Official Site](https://blender-mcp.com/)
- [DEV.to: Blender MCP Seamless Integration](https://dev.to/mehmetakar/blender-mcp-seamless-integration-of-blender-with-claude-ai-302g)
- [YUV.AI: BlenderMCP Controlling Blender with Claude](https://yuv.ai/blog/blender-mcp)
- [CloudThat: Integrating Claude Desktop with Blender](https://www.cloudthat.com/resources/blog/integrating-claude-desktop-app-with-blender-using-mcp-server)
- [Vagon: How to Use Blender MCP with Claude AI](https://vagon.io/blog/how-to-use-blender-mcp-with-anthropic-claude-ai)
- [CSM.ai Blog: Text to 4D Worlds with MCP](https://www.csm.ai/blog/csm-blender-mcp)

### Claude Code and Skills
- [Claude Code MCP Documentation](https://code.claude.com/docs/en/mcp)
- [Anthropic Blog: Extending Claude Capabilities with Skills and MCP](https://claude.com/blog/extending-claude-capabilities-with-skills-mcp-servers)
- [Anthropic Blog: Skills Explained](https://claude.com/blog/skills-explained)
- [alexop.dev: Understanding Claude Code Full Stack](https://alexop.dev/posts/understanding-claude-code-full-stack/)
- [Playbooks: blender-mcp Tool Reference](https://playbooks.com/mcp/ahujasid/blender-mcp)
- [PulseMCP: Blender MCP Server](https://www.pulsemcp.com/servers/ahujasid-blender)

### MCP Directories
- [MCP.so: Blender Server](https://mcp.so/server/blender/ahujasid)
- [LobeHub: Blender MCP](https://lobehub.com/mcp/ahujasid-blender-mcp)
- [GitHub Topics: blender-mcp](https://github.com/topics/blender-mcp)
