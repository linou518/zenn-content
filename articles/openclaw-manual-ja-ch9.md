---
title: "OpenClaw完全ガイド第9章：マルチサーバークラスターデプロイ - エンタープライズ構成"
emoji: "🌐"
type: "tech"
topics: ["openclaw", "ai-agent", "automation", "llm"]
published: true
---

# 第9章：マルチサーバークラスターデプロイ

> 🎯 学習目標：OpenClawのマルチサーバーデプロイアーキテクチャを習得し、高可用性と水平スケーリングを実現する

## 🏗️ **なぜマルチサーバーデプロイが必要か？**

### **シングルサーバーの限界**
- 🔥 **パフォーマンスのボトルネック**: 単一マシンのリソースに限界があり、大量の同時接続に対応不可
- 💥 **単一障害点**: サーバー障害でシステム全体が利用不能に
- 📈 **スケーリングの困難**: 垂直スケーリングはコストが高く、水平スケーリングが不可能
- 🌍 **地理的制約**: グローバルユーザーへの近接サービスが不可能
- 🔒 **セキュリティリスク**: すべてのサービスが1台のマシンに集中

### **マルチサーバーの価値**
- ⚡ **高パフォーマンス**: 分散コンピューティング、大量リクエストの並列処理
- 🛡️ **高可用性**: サーバー障害が全体の可用性に影響しない
- 📊 **水平スケーリング**: 負荷に応じてサーバーを動的に追加
- 🌐 **地理分散**: グローバルデプロイ、ユーザーに近い場所でサービス
- 🔐 **セキュリティ分離**: 異なるサービスを分離された環境で実行

---

## 🏛️ **マルチサーバーアーキテクチャ設計**

### **アーキテクチャパターン1: 機能分離型**
```
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   Gateway       │  │   Agent Farm    │  │   Service       │
│   Server        │  │   Server        │  │   Server        │
├─────────────────┤  ├─────────────────┤  ├─────────────────┤
│ • Load Balancer │  │ • Agent-1       │  │ • Database      │
│ • API Gateway   │  │ • Agent-2       │  │ • File Storage  │
│ • Auth Service  │  │ • Agent-3       │  │ • Message Queue │
│ • Rate Limiting │  │ • Agent-4       │  │ • Cache         │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

### **アーキテクチャパターン2: 地理分散型**
```
                    ┌─────────────────┐
                    │   Global CDN    │
                    │  & Load Balancer│
                    └─────────┬───────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
┌───────▼──────┐     ┌────────▼────────┐     ┌──────▼───────┐
│  US-East     │     │    EU-Central   │     │  Asia-Pacific│
│  Region      │     │    Region       │     │   Region     │
├──────────────┤     ├─────────────────┤     ├──────────────┤
│ Gateway      │     │ Gateway         │     │ Gateway      │
│ Agent Farm   │     │ Agent Farm      │     │ Agent Farm   │
│ Services     │     │ Services        │     │ Services     │
└──────────────┘     └─────────────────┘     └──────────────┘
```

---

## 🛠️ **実践：3サーバークラスターデプロイ**

### **サーバー計画**

| サーバー | 役割 | スペック | コンポーネント |
|--------|------|------|------|
| **Server-1** | Gateway + LB | 4コア8GB | ゲートウェイ、ロードバランサー、認証 |
| **Server-2** | Agent Farm | 8コア16GB | メインAgentインスタンス |
| **Server-3** | Services | 4コア8GB | データベース、キャッシュ、ストレージ |

### **ネットワークアーキテクチャ**
```
Internet
    │
    ▼
┌─────────────┐
│ Load        │
│ Balancer    │  ← Nginx/HAProxy
│ (Server-1)  │
└──────┬──────┘
       │
   ┌───┴───┐
   ▼       ▼
┌─────────────┐    ┌─────────────┐
│ OpenClaw    │    │ OpenClaw    │
│ Gateway     │    │ Agents      │
│ (Server-1)  │◄──►│ (Server-2)  │
└─────────────┘    └──────┬──────┘
       │                  │
       └──────────────────┼────────┐
                          │        │
                    ┌─────▼────────▼─┐
                    │   Services     │
                    │  • PostgreSQL  │
                    │  • Redis       │
                    │  • File Store  │
                    │  (Server-3)    │
                    └────────────────┘
```

---

## 🚀 **デプロイ実施ガイド**

### **ステップ1: サーバー準備**

#### **基本環境設定（全サーバー共通）**
```bash
#!/bin/bash
# setup-base.sh - 基本環境設定スクリプト

# システムの更新
sudo apt update && sudo apt upgrade -y

# 基本ソフトウェアのインストール
sudo apt install -y \
    curl wget git vim htop \
    docker.io docker-compose \
    nginx certbot \
    postgresql-client redis-tools

# Dockerの設定
sudo usermod -aG docker $USER
sudo systemctl enable docker
sudo systemctl start docker

# Node.jsのインストール
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# OpenClawのインストール
sudo npm install -g @openclaw/openclaw

# ファイアウォールの設定
sudo ufw allow ssh
sudo ufw allow 80
sudo ufw allow 443
sudo ufw allow 18789  # OpenClaw Gateway
sudo ufw --force enable

echo "基本環境設定完了"
```

### **ステップ2: Server-1 (Gateway) の設定**

#### **ロードバランサー設定**
```nginx
# /etc/nginx/sites-available/openclaw
upstream openclaw_gateway {
    least_conn;
    server 127.0.0.1:18789 max_fails=3 fail_timeout=30s;
    server SERVER-2-IP:18789 max_fails=3 fail_timeout=30s backup;
}

server {
    listen 80;
    server_name your-domain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name your-domain.com;

    ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;

    add_header Strict-Transport-Security "max-age=63072000" always;
    add_header X-Frame-Options DENY always;
    add_header X-Content-Type-Options nosniff always;

    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    limit_req zone=api burst=20 nodelay;

    location / {
        proxy_pass http://openclaw_gateway;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 300s;
    }

    location /health {
        access_log off;
        proxy_pass http://openclaw_gateway/health;
    }
}
```

### **ステップ3: Server-2 (Agent Farm) の設定**

#### **Agent専用設定**
```json
{
  "gateway": {
    "port": 18789,
    "bind": "all",
    "cluster": {
      "enabled": true,
      "mode": "agent",
      "primary": "SERVER-1-IP:18789"
    }
  },
  "agents": [
    {
      "id": "email-manager",
      "name": "メール管理者",
      "model": "anthropic/claude-sonnet-4-20250514",
      "workspace": {"root": "./agents/email-manager"},
      "resources": {"memory": "2GB", "cpu": "2"}
    },
    {
      "id": "calendar-manager",
      "name": "スケジュール管理者",
      "model": "anthropic/claude-sonnet-4-20250514",
      "workspace": {"root": "./agents/calendar-manager"},
      "resources": {"memory": "1.5GB", "cpu": "1.5"}
    },
    {
      "id": "doc-processor",
      "name": "ドキュメントプロセッサ",
      "model": "anthropic/claude-sonnet-4-20250514",
      "workspace": {"root": "./agents/doc-processor"},
      "resources": {"memory": "3GB", "cpu": "2.5"}
    },
    {
      "id": "data-analyst",
      "name": "データアナリスト",
      "model": "anthropic/claude-sonnet-4-20250514",
      "workspace": {"root": "./agents/data-analyst"},
      "resources": {"memory": "4GB", "cpu": "3"}
    }
  ]
}
```

### **ステップ4: Server-3 (Services) の設定**

#### **データベース設定**
```sql
-- setup-database.sql
CREATE DATABASE openclaw;
CREATE USER openclaw WITH PASSWORD 'SECURE_PASSWORD';
GRANT ALL PRIVILEGES ON DATABASE openclaw TO openclaw;

\c openclaw

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE IF NOT EXISTS agents (
    id VARCHAR(50) PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    config JSONB NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS sessions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    agent_id VARCHAR(50) NOT NULL,
    user_id VARCHAR(100),
    channel VARCHAR(50),
    context JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS messages (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    session_id UUID NOT NULL,
    role VARCHAR(20) NOT NULL,
    content TEXT NOT NULL,
    metadata JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_sessions_agent_id ON sessions(agent_id);
CREATE INDEX idx_messages_session_id ON messages(session_id);
CREATE INDEX idx_messages_created_at ON messages(created_at);
```

#### **Redis設定**
```redis
# /etc/redis/redis.conf
port 6379
bind 0.0.0.0
protected-mode yes
requirepass REDIS_PASSWORD

save 900 1
save 300 10
save 60 10000

maxmemory 2gb
maxmemory-policy allkeys-lru

tcp-keepalive 300
```

---

## 🔄 **自動化デプロイスクリプト**

```bash
#!/bin/bash
# deploy.sh - 自動化デプロイスクリプト

set -e

SERVERS=("SERVER-1-IP" "SERVER-2-IP" "SERVER-3-IP")
SSH_USER="openclaw"
SSH_KEY="~/.ssh/openclaw-deploy"

GREEN='\033[0;32m'
RED='\033[0;31m'
NC='\033[0m'

log_info() { echo -e "${GREEN}[INFO]${NC} $1"; }
log_error() { echo -e "${RED}[ERROR]${NC} $1"; }

# 前提条件の確認
check_prerequisites() {
    log_info "デプロイの前提条件を確認中..."
    for server in "${SERVERS[@]}"; do
        if ! ssh -i $SSH_KEY -o ConnectTimeout=5 $SSH_USER@$server "echo 'SSH OK'" &>/dev/null; then
            log_error "$server にSSH接続できません"
            exit 1
        fi
    done
    log_info "前提条件の確認完了"
}

# Servicesサーバーのデプロイ
deploy_services() {
    log_info "Servicesサーバーをデプロイ中..."
    ssh -i $SSH_KEY $SSH_USER@${SERVERS[2]} << 'EOF'
        cd ~/openclaw
        docker-compose -f docker-compose.services.yml down
        docker-compose -f docker-compose.services.yml pull
        docker-compose -f docker-compose.services.yml up -d
EOF
    log_info "Servicesサーバーのデプロイ完了"
}

# Agentサーバーのデプロイ
deploy_agents() {
    log_info "Agentサーバーをデプロイ中..."
    ssh -i $SSH_KEY $SSH_USER@${SERVERS[1]} << 'EOF'
        cd ~/openclaw
        openclaw gateway stop 2>/dev/null || true
        sudo npm update -g @openclaw/openclaw
        openclaw gateway start --config agents/config.json --daemon
        sleep 10
        openclaw status
EOF
    log_info "Agentサーバーのデプロイ完了"
}

# Gatewayサーバーのデプロイ
deploy_gateway() {
    log_info "Gatewayサーバーをデプロイ中..."
    ssh -i $SSH_KEY $SSH_USER@${SERVERS[0]} << 'EOF'
        cd ~/openclaw
        sudo cp nginx/openclaw.conf /etc/nginx/sites-available/
        sudo nginx -t
        sudo systemctl reload nginx
        openclaw gateway stop 2>/dev/null || true
        sudo npm update -g @openclaw/openclaw
        openclaw gateway start --config gateway/config.json --daemon
        sleep 10
        openclaw status
        curl -f http://localhost:18789/health || exit 1
EOF
    log_info "Gatewayサーバーのデプロイ完了"
}

# ヘルスチェック
health_check() {
    log_info "ヘルスチェックを実行中..."
    for server in "${SERVERS[@]}"; do
        ssh -i $SSH_KEY $SSH_USER@$server "openclaw status" || log_error "$server のステータスが異常"
    done
    log_info "ヘルスチェック完了"
}

# メインデプロイフロー
main() {
    log_info "OpenClawクラスターデプロイを開始..."
    check_prerequisites
    deploy_services
    sleep 60
    deploy_agents
    sleep 30
    deploy_gateway
    sleep 30
    health_check
    log_info "🎉 OpenClawクラスターデプロイ完了！"
}

main "$@"
```

---

## 📊 **監視と運用**

### **Prometheus監視設定**
```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'openclaw-gateway'
    static_configs:
      - targets: ['SERVER-1-IP:9090', 'SERVER-2-IP:9090']

  - job_name: 'system'
    static_configs:
      - targets: ['SERVER-1-IP:9100', 'SERVER-2-IP:9100', 'SERVER-3-IP:9100']

  - job_name: 'postgres'
    static_configs:
      - targets: ['SERVER-3-IP:9187']

  - job_name: 'redis'
    static_configs:
      - targets: ['SERVER-3-IP:9121']
```

### **アラートルール**
```yaml
# rules/openclaw.yml
groups:
  - name: openclaw
    rules:
      - alert: GatewayDown
        expr: up{job="openclaw-gateway"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "OpenClaw Gatewayがダウンしています"

      - alert: HighResponseTime
        expr: openclaw_request_duration_seconds{quantile="0.95"} > 5
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "レスポンス時間が長いことを検出"

      - alert: HighErrorRate
        expr: rate(openclaw_requests_total{status=~"5.."}[5m]) > 0.1
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "高いエラー率を検出"
```

---

## ✅ **本章のまとめ**

本章の学習を通じて、以下を習得しました：

- [x] マルチサーバーアーキテクチャ設計のコア概念とパターン
- [x] 3サーバークラスターの完全なデプロイ方案
- [x] ロードバランシングと高可用性の設定
- [x] データベースクラスターとキャッシュの設定
- [x] 自動化デプロイと設定管理
- [x] 監視アラート体系の構築
- [x] 運用と障害対応のフロー

---

## 🚀 **次のステップ**

マルチサーバーデプロイを習得したら、さらに学習を進められます：

**[次の章：コンテナ化とオーケストレーション →](../chapter10/README.md)**

Kubernetesクラスターデプロイとコンテナオーケストレーション技術を深く学びましょう。

---

## 📝 **発展演習**

1. **高可用性の実践**: サーバー障害をシミュレートし、自動フェイルオーバーをテスト
2. **パフォーマンスチューニング**: 負荷テストツールを使用してクラスターの性能限界をテスト
3. **災害復旧訓練**: 完全なバックアップとリストアプロセスを実施
4. **マルチリージョンデプロイ**: クロスリージョンのグローバルデプロイに拡張
5. **監視の最適化**: 完全なSRE監視体系を構築

**さらに大規模なエンタープライズデプロイに挑戦する準備はできましたか？** 🎯
