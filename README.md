# Moltbot-Ollama-GPU-Docker-SOP
在 Windows 主機上，使用 Docker Compose 啟動 Moltbot Gateway 與 CLI， 並成功透過 本地 Ollama（GPU）模型執行 Moltbot Agent。

0️⃣ 環境前置條件
主機環境
Windows 10 / 11
NVIDIA GPU（已安裝驅動）
Docker Desktop（啟用 WSL2）
Ollama（Windows 版）

1.確認 Ollama 正常（在主機 PowerShell）
ollama list
ollama run qwen2.5:7b-instruct-q4_K_M "hello"


並確認 API 可存取：
curl http://localhost:11434/api/tags

1️⃣ 建立專案結構
Moltbot/
```text
Moltbot/
├─ docker-compose.yml
├─ Dockerfile
└─ README.md
```

## 2️⃣ docker-compose.yml

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
      - "18789:18789"
      - "18790:18790"
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


3️⃣ 啟動 Moltbot Gateway
docker compose up -d


確認容器狀態：

docker ps

4️⃣ 設定 Moltbot 使用 Ollama

Ollama 不走 agent auth-profiles

必須設定在 ~/.clawdbot/moltbot.json → models.providers.ollama

Moltbot 把 Ollama 視為 OpenAI-compatible provider

4.1 編輯 container 內的 moltbot.json
範例使用模型是:qwen2.5:7b-instruct-q4_K_M 模型有換請自己改


docker exec -it moltbot-moltbot-gateway-1 /bin/sh

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




離開 container：

exit

5️⃣ 重啟 Gateway 讓設定生效
docker compose restart moltbot-gateway

6️⃣ 確認 Agent 已建立
docker compose run --rm moltbot-cli agents list


預期看到：

Agents:
- main (default)
  Model: ollama/qwen2.5:7b-instruct-q4_K_M

7️⃣ 實際測試（Agent 本地推理）
docker compose run --rm moltbot-cli agent \
  --local \
  --agent main \
  --message "你好，請用一句話介紹你自己"

成功判斷方式

CLI 出現中文回答

同時在主機執行：

ollama ps


可看到：

qwen2.5:7b-instruct-q4_K_M   running   100% GPU


👉 代表 Moltbot 已透過 Ollama 使用 GPU 推理
8️⃣ 常見現象與說明
🔥 GPU 100%

第一次推理會：

載入模型

建立 KV cache
屬正常現象，後續會快很多
⚠️ low context window warning
warn<32000


只是提醒，不影響執行
Agent 模式最低需求為 16k context
9️⃣ 關鍵踩雷紀錄（血淚）

❌ 在 auth-profiles.json 設 Ollama（無效）

❌ 未指定 agents.defaults.model.primary

❌ contextWindow < 16000（agent 直接拒絕）

❌ 混用 anthropic / synthetic provider

✅ 最終成果

Docker 化 Moltbot Gateway + CLI

使用本地 Ollama（GPU）

Agent pipeline 正常運作

完全離線可用（除 Docker 本身）
