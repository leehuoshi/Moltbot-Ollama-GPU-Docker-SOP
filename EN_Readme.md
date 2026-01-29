# 🚀 Moltbot + Ollama (GPU) Docker Deployment Notes (Windows)

This document records how to run Moltbot Gateway and CLI on a Windows host using Docker Compose, and successfully execute a Moltbot Agent with a local Ollama (GPU) model.

---

## Prerequisites

Host requirements:

- Windows 10 / 11
- NVIDIA GPU (official drivers installed)
- Docker Desktop (WSL2 enabled)
- Ollama (Windows version)
- Model used in this example: `qwen2.5:7b-instruct-q4_K_M` (you can replace it)

---

## Verify Ollama (Host PowerShell)

Run the following commands:

```powershell
ollama list
ollama run qwen2.5:7b-instruct-q4_K_M "hello"
```

Check that the Ollama API is reachable:

```powershell
curl http://localhost:11434/api/tags
```

If the model list returns normally, Ollama is ready.

---

## Project Structure

```
Moltbot/
├─ docker-compose.yml
├─ Dockerfile
└─ README.txt
```

---

## Port Binding Note

It is recommended to bind to a fixed IP address. The original config without an IP binds globally, which is less secure.

```yaml
ports:
  - "192.168.1.xx:18789:18789"
  - "192.168.1.xx:18790:18790"
```

---

## docker-compose.yml

```yaml
services:
  moltbot-gateway:
    image: moltbot:local
    build: .
    environment:
      HOME: /home/node
      TERM: xterm-256color
    volumes:
      - clawdbot-config:/home/node/.clawdbot
      - clawdbot-workspace:/home/node/clawd
    ports:
      - "192.168.1.11:18789:18789"
      - "192.168.1.11:18790:18790"
    restart: unless-stopped
    command:
      [
        "node",
        "dist/index.js",
        "gateway",
        "--allow-unconfigured",
        "--bind",
        "lan",
        "--port",
        "18789"
      ]

  moltbot-cli:
    image: moltbot:local
    build: .
    environment:
      HOME: /home/node
      TERM: xterm-256color
    volumes:
      - clawdbot-config:/home/node/.clawdbot
      - clawdbot-workspace:/home/node/clawd
    stdin_open: true
    tty: true
    entrypoint: ["node", "dist/index.js"]

volumes:
  clawdbot-config:
  clawdbot-workspace:
```

---

## Start Moltbot Gateway

```powershell
docker compose up -d
docker ps
```

---

## Moltbot + Ollama Integration Notes

Key points:

- Ollama does not use `auth-profiles.json`
- Configure it in `~/.clawdbot/moltbot.json`
- Moltbot treats Ollama as an OpenAI-compatible provider

---

## Enter the Gateway Container

```powershell
docker exec -it moltbot-moltbot-gateway-1 /bin/sh
```

---

## Create / Edit moltbot.json

Example model:
`qwen2.5:7b-instruct-q4_K_M`

```sh
cat > /home/node/.clawdbot/moltbot.json << 'EOF'
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "ollama/qwen2.5:7b-instruct-q4_K_M"
      }
    }
  },
  "models": {
    "mode": "merge",
    "providers": {
      "ollama": {
        "baseUrl": "http://host.docker.internal:11434/v1",
        "apiKey": "ollama-local",
        "api": "openai-responses",
        "models": [
          {
            "id": "qwen2.5:7b-instruct-q4_K_M",
            "name": "Qwen 2.5 7B Instruct",
            "reasoning": false,
            "input": ["text"],
            "contextWindow": 16384,
            "maxTokens": 512
          }
        ]
      }
    }
  }
}
EOF
```

```sh
exit
```

---

## Restart Gateway

```powershell
docker compose restart moltbot-gateway
```

---

## Confirm the Agent

```powershell
docker compose run --rm moltbot-cli agents list
```

Expected output contains:

```
main (default)
Model: ollama/qwen2.5:7b-instruct-q4_K_M
```

---

## Local Agent Inference Test

```powershell
docker compose run --rm moltbot-cli agent --local --agent main --message "Hello, introduce yourself in one sentence."
```

If the CLI returns a response, the agent is working.

Also run on the host:

```powershell
ollama ps
```

If you see the model running and GPU usage at 100%, GPU inference is working.

---

## Common Observations and Pitfalls

GPU 100%:

- The first inference loads the model and builds the KV cache
- This is normal; subsequent runs are faster

Low context window warning:

- Informational only
- Agent minimum requirement is a 16k context window

Common mistakes:

- Configuring Ollama in `auth-profiles.json` (ignored)
- Not setting `agents.defaults.model.primary`
- `contextWindow < 16000`
- Mixing anthropic / synthetic providers

---

## Final Result

- Moltbot Gateway + CLI dockerized
- Local Ollama (GPU) inference working
- Agent pipeline functioning
- Usable offline (except for Docker / Ollama dependencies)
