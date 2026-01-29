# 🚀 Moltbot + Ollama（GPU）Docker 部署紀錄（Windows）

本文件紀錄如何在 Windows 主機上，
使用 Docker Compose 啟動 Moltbot Gateway 與 CLI，
並成功透過本地 Ollama（GPU）模型執行 Moltbot Agent。

---

## 環境前置條件

主機環境需求：

- Windows 10 / 11
- NVIDIA GPU（已安裝官方顯示卡驅動）
- Docker Desktop（啟用 WSL2）
- Ollama（Windows 版）
- 本次使用的 模型是  qwen2.5:7b-instruct-q4_K_M 這個 可以自行替換

---

## 確認 Ollama 正常運作（主機 PowerShell）

執行以下指令：

```powershell
ollama list
ollama run qwen2.5:7b-instruct-q4_K_M "hello"
```

確認 Ollama API 是否可存取：

```powershell
curl http://localhost:11434/api/tags
```

若能正常回傳模型清單，代表 Ollama 已就緒。

---

## 專案結構

```
Moltbot/
├─ docker-compose.yml
├─ Dockerfile
└─ README.txt
```

---


ps ports 建議改成固定地址即可  原始代碼不給ip 等價全局監聽 偏危險
```
ports:
      - "192.168.1.xx:18789:18789" 
      - "192.168.1.xx:18790:18790"
## docker-compose.yml
```

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

## 啟動 Moltbot Gateway

```powershell
docker compose up -d
docker ps
```

---

## Moltbot 與 Ollama 整合重點

重要觀念：

- Ollama 不使用 `auth-profiles.json`
- 必須設定在 `~/.clawdbot/moltbot.json`
- Moltbot 將 Ollama 視為 OpenAI-compatible provider

---

## 進入 Gateway Container

```powershell
docker exec -it moltbot-moltbot-gateway-1 /bin/sh
```

---

## 建立 / 編輯 moltbot.json

以下範例模型為：
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

## 重啟 Gateway

```powershell
docker compose restart moltbot-gateway
```

---

## 確認 Agent 是否存在

```powershell
docker compose run --rm moltbot-cli agents list
```

預期結果包含：

```
main (default)
Model: ollama/qwen2.5:7b-instruct-q4_K_M
```

---

## 本地 Agent 推理測試

```powershell
docker compose run --rm moltbot-cli agent --local --agent main --message "你好，請用一句話介紹你自己"
```

若 CLI 能輸出中文回覆，代表 Agent 正常。

同時在主機執行：

```powershell
ollama ps
```

若看到模型正在執行且 GPU 使用率為 100%，代表成功使用 GPU 推理。

---

## 常見現象與踩雷

GPU 100%：

- 第一次推理會載入模型與建立 KV cache
- 屬正常現象，後續會加快

low context window 警告：

- 僅為提醒
- Agent 最低需求為 16k context

常見錯誤：

- 在 `auth-profiles.json` 設定 Ollama（無效）
- 未指定 `agents.defaults.model.primary`
- `contextWindow < 16000`
- 混用 anthropic / synthetic provider

---

## 最終成果

- Moltbot Gateway + CLI Docker 化完成
- 使用本地 Ollama（GPU）推理
- Agent pipeline 正常運作
- 可於離線環境使用（Docker / Ollama 除外）
