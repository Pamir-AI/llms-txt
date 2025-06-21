# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a comprehensive MCP (Model Context Protocol) server hub that provides hardware control capabilities for embedded systems. The project consists of:

- **Multiple MCP Servers** - Camera, microphone, speaker, and e-ink display control
- **Centralized Service Management** - systemd service with JSON configuration for managing multiple MCPs
- **Development Utilities** - Testing and validation tools

## Architecture

The repository follows a monorepo structure with individual MCP projects under `projects/`:

```
distiller-cm5-mcp-hub/
├── run_all_mcps.py          # Service manager for all MCPs
├── mcp_config.json          # Configuration for enabled MCPs and ports
├── mcp.service              # systemd service definition
├── manage_mcp_service.sh    # Service management helper script
└── projects/                # Individual MCP implementations
    ├── camera-mcp/         # Pi Camera control
    ├── mic-mcp/           # Audio recording and transcription
    ├── speaker-mcp/       # Text-to-speech (Piper TTS)
    └── eink-mcp/          # E-ink display control
```

Each MCP project is self-contained with its own `pyproject.toml`, dependencies, and `server.py` entry point.

## Common Commands

### Development Setup
```bash
# Install root dependencies
uv sync
```

### Running Individual MCPs
```bash
# From project directory (e.g., projects/speaker-mcp/)
uv run python server.py --transport sse --port 8001

# Available transports: stdio, sse, streamable-http
# Each MCP uses different default ports (8001-8003)
```

### Service Management
```bash
# Install and manage the centralized service
./manage_mcp_service.sh install
./manage_mcp_service.sh enable
./manage_mcp_service.sh start
./manage_mcp_service.sh status
./manage_mcp_service.sh logs
./manage_mcp_service.sh config    # Edit mcp_config.json
./manage_mcp_service.sh restart
```

### Code Quality
```bash
# For projects with dev dependencies (speaker-mcp, camera-mcp, mic-mcp)
cd projects/[project-name]/

# Code formatting
uv run black .
uv run isort .

# Type checking  
uv run mypy .

# Linting (where available)
uv run flake8 .

# Testing
uv run pytest
```

### Testing and Validation
```bash
# Run sanity checks
python sanity_check.py

# Test individual projects
cd projects/[project-name]/
python test_server.py  # if exists
```

## Hardware MCP Development Rules (CRITICAL)

### Environment Setup Requirements

**FUNDAMENTAL PRINCIPLE**: Always use UV with specific system-site-packages inclusion for hardware projects:

```bash
# Critical setup sequence for hardware MCPs
echo "3.11" > .python-version
uv venv --python /usr/bin/python3.11 --system-site-packages
uv add fastmcp numpy pillow
uv add /path/to/distiller_cm5_sdk.whl
uv sync
```

### Hardware-First Development Protocol

**MANDATORY**: Hardware verification MUST come first before any MCP development:

1. **Hardware Verification** (before any MCP code):
```bash
# Test system hardware access
uv run python -c "import lgpio; print('✅ GPIO access OK')"

# Test SDK import  
uv run python -c "import distiller_cm5_sdk; print('✅ SDK import OK')"

# Test hardware initialization
uv run python -c "
from distiller_cm5_sdk.hardware.eink.eink import EinkDriver
driver = EinkDriver()
result = driver.initialize()
print(f'✅ Hardware init: {result}')
driver.cleanup()
"
```

2. **Stop Rule**: If ANY hardware test fails, DO NOT proceed with MCP development until fixed.

3. **No Simulation Mode**: Hardware control servers must control real hardware - simulation mode is not acceptable.

### Hardware-Specific Requirements

- **GPIO Library Priority**: lgpio (preferred) > pigpio (fallback) > RPi.GPIO (legacy)
- **Available GPIO Pins**: GPIO2, GPIO3, GPIO12, GPIO14, GPIO15, GPIO17, GPIO22-25, GPIO27
- **Reserved GPIO Pins**: GPIO0-1, GPIO4-5, GPIO6-11, GPIO13, GPIO16, GPIO18-21, GPIO26
- **Hardware Components**: Audio codec, RP2040 microcontroller, E-Ink display, Camera/display MIPI

### MCP Server Development Guidelines

**Project Structure** for hardware MCPs:
- Separate hardware control logic from MCP server logic
- Implement graceful degradation when hardware is unavailable
- Include proper error handling for GPIO operations
- Document required config.txt modifications

**Error Handling Requirements**:
- Always implement graceful degradation for hardware failures
- Provide meaningful error messages for hardware unavailability
- Include fallback behaviors when possible
- Clean up GPIO resources properly in finally blocks

## MCP Development from Scratch

### Project Initialization Process

1. **Create new MCP project directory**:
```bash
# From projects/ directory
uv init {mcp-server-name}
cd {mcp-server-name}
```

2. **Environment Setup**:
```bash
# Set Python version
echo "3.11" > .python-version

# Create virtual environment (with system-site-packages for hardware projects)
uv venv --python /usr/bin/python3.11 --system-site-packages

# Add core dependencies
uv add "mcp[cli]>=1.1.2"
uv add "fastmcp>=2.8.1"

# Sync dependencies
uv sync
```

3. **Project Structure**:
```
{mcp-server-name}/
├── .python-version
├── main.py              # Entry point
├── pyproject.toml       # Dependencies and config
├── README.md
├── uv.lock
├── tests/
│  ├── test_{mcp_server_name}.py
│  └── __init__.py
└── src/
   └── {project_name}/
      ├── server.py      # MCP server logic
      ├── {domain}.py    # Business logic
      ├── models.py      # Data models
      ├── utils.py       # Utilities
      └── __init__.py
```

### Key Development Patterns

**FastMCP Server Template**:
```python
from fastmcp import FastMCP

# Create MCP server
mcp = FastMCP("Your MCP Server Name")

@mcp.tool()
async def get_status() -> str:
    """Get service status and health information."""
    return "Status: Service running"

def main(transport: str = 'streamable-http', host: str = '0.0.0.0', port: int = 8000):
    """Main server function with transport configuration."""
    mcp.run(transport=transport, host=host, port=port)
```

**Testing Pattern**:
```python
import pytest
from fastmcp import Client
from src.{project_name}.server import mcp

@pytest.mark.asyncio
async def test_server_status():
    async with Client(mcp) as client:
        result = await client.call_tool("get_status", {})
        assert "Status:" in result[0].text
```

## Key Configuration Files

- `mcp_config.json` - Controls which MCPs are enabled and their port assignments
- `mcp.service` - systemd service definition for production deployment
- Individual `pyproject.toml` files define dependencies and tool configurations for each MCP
- `.python-version` files - Critical for UV environment management (must contain "3.11")

## Transport Protocols

MCPs support multiple transport protocols:
- **stdio**: Traditional MCP using stdin/stdout
- **sse**: Server-Sent Events for web applications  
- **streamable-http**: RESTful HTTP API

## Service Architecture

The centralized service (`run_all_mcps.py`) reads `mcp_config.json` and manages multiple MCP processes:
- Starts enabled MCPs in their respective project directories
- Monitors processes and restarts them on failure
- Provides centralized logging through systemd journal
- Handles graceful shutdown

## Port Assignments

- Camera MCP: 8001
- Mic MCP: 8002  
- Speaker MCP: 8003
- Additional MCPs use incremental ports

## Dependencies

Projects use different dependency sets:
- **Core**: `mcp[cli]`, `fastmcp`, `fastapi`, `uvicorn`, `pydantic`
- **Hardware**: `picamera2`, `distiller-cm5-sdk` (for CM5-specific functionality)
- **Development**: `pytest`, `black`, `isort`, `mypy`, `flake8`

## Development Workflow

### For Hardware MCPs (CRITICAL PROCESS):

1. **Hardware Verification First**: Test GPIO access, SDK imports, and hardware initialization
2. **Environment Setup**: Use UV with --system-site-packages flag
3. **Hardware Integration**: Create hardware abstraction layer with proper error handling
4. **MCP Development**: Only after hardware is verified and working
5. **Testing**: Test with real hardware, not simulation

### For General MCPs:

1. Work within individual project directories under `projects/`
2. Each MCP is independently testable and deployable
3. Use the centralized service for production deployment
4. Test changes with `python sanity_check.py` before committing
5. Follow the existing code style and patterns in each project

## UV Environment Management (CRITICAL)

**Golden Rule**: `uv run python` should equal `source .venv/bin/activate && python`

**Common Issues and Solutions**:
1. **Wrong Python Version**: Check `.python-version` file (must be "3.11")
2. **Missing System Packages**: Recreate venv with `--system-site-packages`
3. **Dependency Issues**: Use `uv add package` not `uv pip install package`

**Verification Commands**:
```bash
# Check Python version consistency
uv run python -c "import sys; print('UV:', sys.version_info)"
source .venv/bin/activate && python -c "import sys; print('Manual:', sys.version_info)"

# Test hardware imports (for hardware MCPs)
uv run python -c "import lgpio; print('Hardware libs OK')"
```

## Connection Information

When running individual MCPs:
- **Local**: `http://127.0.0.1:{port}/mcp/`
- **Remote**: `http://{machine-ip}:{port}/mcp/`
- Get IP: `hostname -I | awk '{print $1}'`

## Important Notes

- **Never use simulation mode** for hardware control servers
- **Always verify hardware first** before MCP development
- **Use system-site-packages** for hardware MCPs to access system GPIO/hardware libraries
- **Follow hardware-first development protocol** for any hardware-related MCP servers
- **Each MCP project is self-contained** and independently deployable
