# Distiller Dev‑Kit Quickstart Guide

Welcome! Your PamirAI **Distiller CM5** device is ready to run local AI models and Modular Command Processors (MCPs) out of the box. This guide walks you through the first things to try, plus where to dive deeper when you’re ready.

---

## 1 · Repo Map

| Capability                                   | Repo                       |
| -------------------------------------------- | -------------------------- |
| Local LLM server & CLI                       | **distiller-cm5-python**   |
| Speech / vision SDK (Whisper, Piper, etc.)   | **distiller-cm5-sdk**      |
| RP2040 firmware                              | **distiller-cm5-firmware** |
| Custom Linux kernel                          | **Distiller-CM5-Kernel**   |
| Background services (Wi‑Fi setup, OTA, etc.) | **distiller-cm5-services** |
| MCP Hub & examples                           | **distiller-cm5-mcp-hub**  |

*(All repos live under **************************************************************[https://github.com/Pamir-AI/](https://github.com/Pamir-AI/)**************************************************************)*

---

## 2 · Built‑In MCP Endpoints

| MCP           | Purpose                      | Port     |
| ------------- | ---------------------------- | -------- |
| `camera_mcp`  | JPEG frames & camera control | **8001** |
| `mic_mcp`     | Microphone stream & VAD      | **8002** |
| `speaker_mcp` | Speaker / TTS output         | **8003** |

*Port forwarding is pre‑enabled, so these services are reachable from your development machine once the device is on the same LAN.*

### 2.1 Manage MCP Services

**Path:** `/home/distiller/distiller-cm5-mcp-hub`

```bash
# 1 · Run the test suite (recommended)
cd /home/distiller/distiller-cm5-mcp-hub
./test_mcp_manager.py

# 2 · Install the systemd service units
./manage_mcp_service.sh install

# 3 · Enable services to start on boot
./manage_mcp_service.sh enable

# 4 · Start all services now
./manage_mcp_service.sh start

# 5 · Check status & view logs
./manage_mcp_service.sh status
```

All MCPs are declared in **`mcp_config.json`**. Edit this file to enable/disable, change ports, or add your own service, then rerun the *install* script:

```json
{
  "camera":  { "enabled": true, "port": 8001, "host": "0.0.0.0", "project_dir": "camera-mcp",  "description": "Camera MCP Service" },
  "mic":     { "enabled": true, "port": 8002, "host": "0.0.0.0", "project_dir": "mic-mcp",     "description": "Microphone MCP Service" },
  "speaker": { "enabled": true, "port": 8003, "host": "0.0.0.0", "project_dir": "speaker-mcp", "description": "Speaker MCP Service" }
}
```

Once **enabled**, each service auto‑starts at boot and is reachable from any device on your LAN at:

```
http://<DEVICE_IP>:<PORT>/sse 
# or in most case you can just use http://distiller.local:<PORT>/sse if you running everything locally
```

### 2.2 One‑Click install in **Cursor**

[![Install MCP Server](https://cursor.com/deeplink/mcp-install-dark.svg)](https://cursor.com/install-mcp?name=camera&config=eyJ1cmwiOiJodHRwOi8vZGlzdGlsbGVyLmxvY2FsOjgwMDEvc3NlIn0%3D)

> **Heads‑up:** Swap `<DEVICE_IP>` for your board actual address (e.g. `192.168.0.218`).
> Need remote access? [Ngrok](https://ngrok.com/docs/getting-started/) offers a free HTTPS tunnel so you can reach the MCP from anywhere.

#### Other supported clients *(how‑tos coming soon)*

* Claude Code
* OpenAI Playground / API
* Claude Pro

---

## 3 · Local LLM Server

Two lightweight Qwen models are pre‑downloaded:

* **Qwen 2.5 3B** (better reasoning, slower)
* **Qwen 3 1.7B** (faster, smaller RAM footprint)

Both live under **distiller‑cm5‑python/llm\_models/**.

### 3.1 Run in CLI

```bash
cd ~/distiller-cm5-python
uv run main.py            # REPL-style chat
```

### 3.2 Run as REST API

```bash
# From the repo root
uv run llm_server/server.py --model qwen-2_5-3b  # defaults to :8080
```

### 3.3 Sample HTTP call

```bash
curl -X POST http://<DEVICE_IP>:8080/v1/chat/completions \
     -H 'Content-Type: application/json' \
     -d '{
           "model": "qwen-2_5-3b",
           "messages": [{"role":"user","content":"Hello from PamirAI!"}]
         }'
```

> **Tip:** Explore `llm_server/handlers.py` for available routes (chat, embeddings, tools, …).

---

## 4 · Create Your Own MCP

1. Clone **distiller-cm5-mcp-hub** on your workstation.
2. ***\<TODO>***

📺 **Video walkthrough:** \<VIDEO\_TUTORIAL\_MCP\_PLACEHOLDER>

---

## 5 · Automate Everything in n8n

Use n8n’s HTTP node to call your MCP or LLM endpoints, then chain to hundreds of integrations (Slack, GitHub, sensors…).

***\<TODO>***

📺 **Video guide:** \<VIDEO\_TUTORIAL\_N8N\_PLACEHOLDER>

---

## 6 · Next Steps & Troubleshooting

| Need help?     | Where to look ***\<TODO>***                      |
| -------------- | ------------------------------------------------ |
| Device console | `ssh distiller@<DEVICE_IP>`                      |
| Community chat | #distiller‑devs channel on Discord ***\<TODO>*** |
|                |                                                  |

Happy building! 🛠️
— *The PamirAI Team*
