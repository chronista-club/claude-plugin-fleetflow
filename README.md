# FleetFlow Claude Code Plugin

Claude Code plugin for FleetFlow container orchestration.

**環境構築は、対話になった。伝えれば、動く。**

## Features

- **Project Inspection** - Analyze FleetFlow projects and view service/stage configurations
- **Container Management** - Start, stop, restart containers for your project
- **Build Support** - Build Docker images from Dockerfiles
- **Log Viewing** - View container logs directly from Claude Code
- **Configuration Validation** - Validate fleet.kdl syntax and structure
- **Cloud Infrastructure** - Manage cloud servers (Sakura Cloud) and DNS (Cloudflare)
- **Fleet Registry** - Manage multiple FleetFlow projects and SSH remote deploy

## Installation

```bash
# Install FleetFlow CLI (includes MCP server)
cargo install --git https://github.com/chronista-club/fleetflow fleetflow

# Copy plugin config to Claude Code
cp .mcp.json ~/.claude/
```

## MCP Tools

### v1: ローカル操作

| Tool | Description | Required Args |
|------|-------------|---------------|
| `fleetflow_inspect_project` | Analyze fleet.kdl and show project structure | - |
| `fleetflow_ps` | Show container status for project | - |
| `fleetflow_up` | Start containers for a stage | `stage` |
| `fleetflow_down` | Stop containers for a stage | `stage` |
| `fleetflow_deploy` | Deploy stage (CI/CD: stop → pull → start) | `stage` |
| `fleetflow_logs` | View container logs | `stage` |
| `fleetflow_restart` | Restart a specific service | `stage`, `service` |
| `fleetflow_validate` | Validate configuration files | - |
| `fleetflow_validate_secrets` | Validate 1Password secret references | - |
| `fleetflow_build` | Build Docker images | `stage` |
| `fleetflow_setup` | Setup stage environment (idempotent) | `stage` |
| `fleetflow_env` | Show environment variables (masked) | `stage` |

### v2: Control Plane 経由

| Tool | Description | Required Args |
|------|-------------|---------------|
| `fleetflow_cp_tenant` | テナント情報の取得・更新 | - |
| `fleetflow_cp_project` | プロジェクトの CRUD 操作 | `slug` |
| `fleetflow_cp_stage` | ステージ概要の取得 | `project`, `stage` |
| `fleetflow_cp_service` | サービス詳細の取得 | `project`, `stage` |
| `fleetflow_cp_container` | コンテナ情報の取得 | `project`, `stage` |
| `fleetflow_cp_server` | サーバーの CRUD + 電源操作 | `slug` |
| `fleetflow_cp_health` | ヘルスチェック状態の取得 | `project`, `stage` |
| `fleetflow_cp_cost` | コスト情報の取得 | - |
| `fleetflow_cp_dns` | DNS レコード管理 | `project`, `stage` |

## Usage Examples

Once installed, you can use FleetFlow tools directly in Claude Code:

```
User: Show me the FleetFlow project structure
Claude: [Uses fleetflow_inspect_project]

User: Validate the configuration
Claude: [Uses fleetflow_validate]

User: Start the local environment
Claude: [Uses fleetflow_up with stage="local"]

User: Check what's running
Claude: [Uses fleetflow_ps]

User: Show me the logs for the web service
Claude: [Uses fleetflow_logs with stage="local", service="web"]

User: Restart the api service
Claude: [Uses fleetflow_restart with stage="local", service="api"]

User: Build images for local
Claude: [Uses fleetflow_build with stage="local"]

User: Deploy to live
Claude: [Uses fleetflow_deploy with stage="live"]

User: Stop everything
Claude: [Uses fleetflow_down with stage="local", remove=true]

User: Show me the environment variables for local stage
Claude: [Uses fleetflow_env with stage="local"]

User: Validate the 1Password secret references
Claude: [Uses fleetflow_validate_secrets]
```

## ステージ管理

FleetFlowは `local` / `dev` / `pre` / `live` の4ステージでインフラを統一管理する。MCPツールの `stage` 引数でステージを指定する。

| ステージ | 用途 |
|----------|------|
| `local` | ローカル開発環境（OrbStack/Docker Desktop） |
| `dev` | 開発サーバー |
| `pre` | ステージング・検証環境 |
| `live` | 本番環境 |

環境変数 `FLEET_STAGE` を設定すると、MCPツールのデフォルトステージとして使用される。

## fleet.kdl 設定の基本

プロジェクトの `.fleetflow/fleet.kdl` に設定を記述する:

```kdl
project "myapp"

stage "local" {
    service "db"
    service "api"
}

service "db" {
    image "postgres:16"          // image は必須
    restart "unless-stopped"
    ports {
        port host=5432 container=5432
    }
    env {
        POSTGRES_PASSWORD "postgres"
    }
}

service "api" {
    image "myapp/api:latest"
    depends_on "db"
    ports {
        port host=3000 container=3000
    }
}
```

詳細なKDL構文はスキルの `reference/kdl-syntax.md` を参照。

## Environment Variables

| Variable | Description |
|----------|-------------|
| `FLEET_STAGE` | Default stage name (local/dev/pre/live) |
| `FLEETFLOW_CONFIG_PATH` | Direct path to config file |
| `OP_SERVICE_ACCOUNT_TOKEN` | 1Password Service Account token (CI/CD) |
| `CLOUDFLARE_API_TOKEN` | Cloudflare API token (for DNS) |
| `CLOUDFLARE_ZONE_ID` | Cloudflare Zone ID |
| `CLOUDFLARE_DOMAIN` | Managed domain |

## Requirements

- Docker or OrbStack running
- FleetFlow project with `.fleetflow/fleet.kdl`
- `fleet` binary in your PATH
- 1Password CLI (`op`) v2.x+ for secret management (optional)

## Fleet Agent

Fleet Agent はリモートサーバー上で動作する常駐エージェント。Control Plane と双方向通信し、以下を実行する:

- **デプロイ実行**: CP からの指示で `docker compose up` を実行
- **ヘルスモニタリング**: コンテナの状態を定期監視し、異常（再起動ループ、予期しない停止、unhealthy）を検出してアラート送信
- **アクション応答**: Dashboard からの再デプロイ・再起動リクエストに応答

## Dashboard API

Control Plane の WebUI Dashboard（`127.0.0.1:32080`）から利用可能な API:

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/tenant` | テナント情報 |
| `GET` | `/api/tenant/users` | テナントユーザー一覧 |
| `GET` | `/api/projects` | プロジェクト一覧 |
| `GET` | `/api/stages/:project/:stage` | ステージ概要 |
| `POST` | `/api/stages/:project/:stage/redeploy` | 再デプロイ（CP → Agent → docker compose up） |
| `POST` | `/api/stages/:project/:stage/restart/:service` | サービス再起動（CP → Agent → docker restart） |
| `GET` | `/api/stages/:project/:stage/alerts` | アクティブアラート一覧 |
| `GET` | `/api/stages/:project/:stage/logs/:container` | コンテナログ取得 |
| `GET` | `/api/agents` | 接続中 Agent 一覧 |

## Architecture

This plugin is a lightweight configuration package. The MCP server is built into the main FleetFlow CLI:

```
fleetflow/                          # Main repo
├── crates/
│   ├── fleetflow/                  # CLI with MCP server
│   ├── fleetflow-mcp/              # MCP server library (v1 + v2)
│   ├── fleetflow-registry/         # Fleet Registry library
│   ├── fleetflow-controlplane/     # Control Plane library
│   ├── fleetflowd/                 # Control Plane daemon
│   └── fleet-agent/                # Fleet Agent (server-side)

claude-plugin-fleetflow/            # This plugin repo
├── .claude-plugin/plugin.json      # Plugin metadata
├── .mcp.json                       # MCP server configuration
└── README.md
```

## Related Projects

- [FleetFlow](https://github.com/chronista-club/fleetflow) - Main container orchestration CLI
- [Creo Memories](https://github.com/chronista-club/claude-plugin-creo-memories) - Persistent memory plugin

## License

MIT OR Apache-2.0
