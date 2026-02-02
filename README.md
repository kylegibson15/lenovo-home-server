# lenovo-home-server

Repurposing a ~2010 Lenovo ThinkPad T410 as a headless Linux home server for learning local LLM infrastructure.

## What This Is

A learning project to understand how to:

- Set up a Linux server from scratch (Ubuntu Server 24.04 LTS)
- Run a local LLM (large language model) using [Ollama](https://ollama.com)
- Expose a model as a REST API on a local network
- Query the model from other devices via API or browser chat UI

This is a documentation-first repo — the primary artifact is the build guide, not application code.

## Repository Contents

| File | Description |
|---|---|
| [SERVER_BUILD_GUIDE.md](SERVER_BUILD_GUIDE.md) | Step-by-step guide covering hardware audit, OS install, Ollama setup, network config, and chat UI |
| [chat.html](chat.html) | Lightweight browser-based chat interface that connects to the Ollama API |

## Hardware

- **Laptop:** Lenovo ThinkPad T410 (2010)
- **CPU:** Intel Core i5-520M (2 cores / 4 threads)
- **RAM:** 4 GB DDR3
- **Storage:** 160 GB HDD
- **GPU:** None (CPU-only inference)

## Quick Start

If you've already completed the server setup:

```bash
# SSH into the server
ssh your-username@your-server-ip

# Check Ollama is running
sudo systemctl status ollama

# Chat with the model
ollama run tinyllama
```

To use the browser chat UI, serve `chat.html` locally and open it in Safari:

```bash
cd /path/to/lenovo-home-server
python3 -m http.server 8080
# Open http://localhost:8080/chat.html in Safari
```

## Future Plans

- Security hardening (SSH keys, firewall, fail2ban)
- LLM request orchestration and routing across multiple models
- Upgrade to more capable hardware for larger models
