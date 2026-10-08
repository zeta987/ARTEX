# CLAUDE.md

本檔給在此 repo 根目錄工作的 coding agent 使用，記錄這個 fork 的實際建置、部署與更新流程。README 的安裝／更新章節多數已失效，以本檔為準。

## 專案背景

- ARTEX 是 AI 自主滲透測試系統（Go 後端 + Next.js 前端），會主動對目標發動掃描與攻擊。只能對已取得授權的目標使用。
- 原作者的 `Autumn-27/ARTEX` 已刪除。本 repo fork 自完整鏡像 `mhtsec/ARTEX`，同步到 v0.3.15（`e6ec569`）。
- Remotes：`origin` = 本 fork，`upstream` = `https://github.com/mhtsec/ARTEX.git`。新 clone 時自行補上 `upstream`。

## 已失效的 README 流程（不要照做）

| README 內容 | 現況 |
|---|---|
| `docker compose pull`／`autumn27/artex` 映像 | Docker Hub 已 404，只能本機建置 `artex:local` |
| Releases 預編譯包 | 原 repo 已刪除，沒有 Releases |
| 頁面「一鍵更新」、`update.sh` | `selfupdate/github.go` 寫死 `Autumn-27/artex`，必定失敗 |
| `npm run build:static`、`./build.sh` | POSIX 語法，Windows PowerShell 不能直接執行 |

## 本機部署結構（Docker Compose）

以下檔案都不進 git（`.env`、`jwt.key`、`dist/`、`server/webui/dist/` 已在 `.gitignore`；override 放在 `.git/info/exclude`）：

| 檔案 | 用途 |
|---|---|
| `.env` | 由 `.env.example` 複製，`POSTGRES_PASSWORD` 為隨機值。不要印出、不要提交 |
| `docker-compose.override.yml` | 改用 `artex:local`、禁止 pull、port 只綁 loopback、掛載 `jwt.key` |
| `jwt.key` | JWT 簽章金鑰，掛到容器 `/app/jwt.key`，容器重建後登入狀態不會失效 |
| `data/` | 工作區（SQLite、session 檔案），網頁檔案瀏覽器和 agent 工具都讀得到 |

- `jwt.key` 絕對不要放進 `data/`：那裡可被瀏覽下載，而且程式啟動時會把 `data/jwt.key` 遷移走並刪除。
- 不要把 override 的 port 改回 `0.0.0.0`：首次進入 `/setup` 的人就能設定管理員密碼。

`docker-compose.override.yml` 內容（新機器需手動建立，並把檔名加進 `.git/info/exclude`）：

```yaml
services:
  artex:
    image: artex:local
    pull_policy: never
    ports: !override
      - "127.0.0.1:8787:8787"
      - "127.0.0.1:8788:8788"
    volumes:
      - ./jwt.key:/app/jwt.key
```

## 新機器初始化（PowerShell）

`jwt.key` 必須在 `docker compose up` 之前就存在且是檔案；否則 Docker Desktop 會在該路徑建立資料夾，artex 無法啟動。

```powershell
# .env：隨機 Postgres 密碼
$pw = -join ((1..32) | ForEach-Object { '{0:x2}' -f (Get-Random -Maximum 256) })
(Get-Content .env.example) -replace '^POSTGRES_PASSWORD=.*', "POSTGRES_PASSWORD=$pw" | Set-Content .env

# jwt.key：至少 32 個英數字元
$chars = [char[]]'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789'
-join ((1..48) | ForEach-Object { $chars | Get-Random }) | Set-Content -NoNewline jwt.key
```

接著依下節建置映像，再 `docker compose up -d`。

## 建置 `artex:local`（PowerShell）

```powershell
# 1) 前端靜態匯出（等同 npm run build:static）
Set-Location web
npm ci
$env:NEXT_EXPORT = '1'; npx next build
Set-Location ..

# 2) 嵌入前端
if (Test-Path server\webui\dist) { Remove-Item -Recurse -Force server\webui\dist }
New-Item -ItemType Directory -Force server\webui | Out-Null
Copy-Item -Recurse web\out server\webui\dist

# 3) 交叉編譯 Linux 二進位（Dockerfile 從 dist/<TARGETARCH>/artex 複製）
$env:CGO_ENABLED = '0'; $env:GOOS = 'linux'; $env:GOARCH = 'amd64'
go build -tags embedui -trimpath -ldflags '-s -w' -o dist/amd64/artex ./cmd/artex

# 4) 建映像（會下載 Playwright Chromium，耗時數分鐘）
docker build --platform linux/amd64 -t artex:local .
```

- `npm ci` 會提示 `sharp` 的 install script 被擋；static export 用不到，可忽略。
- 未注入版本號，介面顯示版本 `dev`，不影響功能。

## 啟動與檢查

```powershell
docker compose up -d
Invoke-WebRequest http://127.0.0.1:8787/api/auth/status   # 預期 200 + {"initialized":...}
docker compose logs artex --tail 30
```

- Web UI：`http://localhost:8787`。首次進入 `/setup` 設管理員密碼，再到 `/system/llm` 設定 LLM provider。
- 流量錄製代理：`127.0.0.1:8788`，需信任 ARTEX 產生的 CA。
- 容器重建後若 log 出現「新 JWT key 已写入」，代表 `jwt.key` 掛載失效。

## 更新流程

```powershell
git fetch upstream
git merge upstream/main        # 或在 feature 分支上處理衝突
# 重跑「建置 artex:local」四個步驟
docker compose up -d           # 換新映像重啟會自動遷移 schema
```

- 不要執行 `docker compose pull`、`update.sh`，也不要按頁面上的「一鍵更新」。
- `upstream` 只是鏡像。原作者已不再維護，之後的修補需要自己追蹤或從其他 fork 取得。

## 開發檢查

- 後端：`go test ./...`
- 前端：`cd web` 後執行 `npm run check`（Biome）
- 程式註解與 commit 慣例沿用上游（中文註解、Conventional Commits）。
