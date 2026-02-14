---
title: "OpenClaw完全ガイド第8章：監視とデバッグ技術 - 本番運用のオブザーバビリティ"
emoji: "📊"
type: "tech"
topics: ["openclaw", "ai-agent", "automation", "llm"]
published: true
---

# 第8章：監視とデバッグ技術

> 🎯 学習目標：包括的なOpenClaw監視体制を確立し、パフォーマンスデバッグ手法を習得し、障害の迅速な特定と自動化運用を実現する

## 📊 **監視体系の概要**

本番レベルのOpenClawデプロイには、全方位の監視能力が必要です：
- 🔍 **リアルタイム監視**: システムステータス、パフォーマンス指標、エラー率
- 📝 **ログ管理**: 構造化ログ、集中収集、インテリジェント分析
- ⚠️ **アラートメカニズム**: 異常検知、段階的アラート、自動対応
- 📈 **可視化**: ダッシュボード、トレンド分析、キャパシティプランニング

---

## 🏗️ **監視アーキテクチャ設計**

### **8.1 監視階層モデル**

```
┌─────────────────────────────────────────┐
│            アプリケーション層監視            │
│   Agentパフォーマンス | Sessionステータス    │
├─────────────────────────────────────────┤
│              Gateway監視                  │
│    接続数 | レスポンス時間 | スループット      │
├─────────────────────────────────────────┤
│             システム層監視                  │
│    CPU | Memory | Disk | Network        │
├─────────────────────────────────────────┤
│           インフラストラクチャ監視            │
│    サーバー | ネットワーク | ストレージ        │
└─────────────────────────────────────────┘
```

---

## 📋 **OpenClaw組み込み監視**

### **8.3 Gatewayステータス監視**

#### **基本ステータスクエリ**
```bash
# 包括的なシステムステータス
openclaw status

# 詳細な監視情報
openclaw status --all --deep

# JSON形式出力（スクリプト処理に便利）
openclaw status --json
```

#### **重要な監視指標**
```json
{
  "gateway": {
    "uptime": "72h 15m",
    "version": "2026.2.9",
    "connections": 42,
    "requests_total": 15847,
    "requests_per_minute": 23.4,
    "memory_usage": "512MB",
    "cpu_usage": "15%"
  },
  "agents": {
    "total": 8,
    "active": 6,
    "sessions": 299,
    "avg_response_time": "1.2s"
  },
  "channels": {
    "telegram": {"status": "OK", "latency": "45ms"},
    "slack": {"status": "OK", "latency": "32ms"},
    "whatsapp": {"status": "WARN", "error": "Not linked"}
  }
}
```

### **8.4 ヘルスチェック設定**

#### **自動化ヘルスチェック**
```bash
# 完全なヘルスチェックを実行
openclaw doctor --non-interactive

# 特定コンポーネントの確認
openclaw doctor --check-channels
openclaw doctor --check-models
openclaw doctor --check-security
```

---

## 📝 **ログ管理システム**

### **8.5 OpenClawログ設定**

#### **ログレベル設定**
```json
{
  "logging": {
    "level": "info",
    "format": "structured",
    "outputs": [
      {
        "type": "file",
        "path": "/var/log/openclaw/gateway.log",
        "rotation": {
          "maxSize": "100MB",
          "maxFiles": 10,
          "compress": true
        }
      },
      {
        "type": "syslog",
        "facility": "local0"
      }
    ]
  }
}
```

#### **ログ確認コマンド**
```bash
# リアルタイムログストリーム
openclaw logs --follow

# エラーログのフィルタリング
openclaw logs --level error --since "1h"

# 特定Agentのログ
openclaw logs --agent main --limit 100

# チャネル別フィルタリング
openclaw logs --channel telegram --since "2026-02-13"
```

### **8.6 ELK Stack統合**

```yaml
# docker-compose-logging.yml
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ports:
      - "9200:9200"

  logstash:
    image: docker.elastic.co/logstash/logstash:8.11.0
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf
      - /var/log/openclaw:/logs:ro

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    ports:
      - "5601:5601"
```

### **8.7 構造化ログのベストプラクティス**

#### **ログフォーマットの標準化**
```json
{
  "timestamp": "2026-02-13T11:02:45.123Z",
  "level": "INFO",
  "component": "gateway",
  "agent_id": "main",
  "session_id": "abc123",
  "channel": "telegram",
  "user_id": "7996447774",
  "action": "message_received",
  "message": "User message processed successfully",
  "duration_ms": 1250,
  "metadata": {
    "tool_calls": 3,
    "tokens_used": 1847,
    "model": "claude-sonnet-4"
  }
}
```

---

## 📈 **パフォーマンス監視とデバッグ**

### **8.8 Prometheus統合**

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'openclaw-gateway'
    static_configs:
      - targets: ['localhost:18789']
    metrics_path: '/metrics'
    scrape_interval: 10s

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['localhost:9100']
```

#### **カスタムメトリクスのエクスポート**
```bash
# OpenClaw Prometheusメトリクスエンドポイント
curl http://localhost:18789/metrics

# メトリクス出力例
openclaw_gateway_requests_total{channel="telegram"} 15847
openclaw_gateway_response_time_seconds{quantile="0.5"} 1.2
openclaw_gateway_response_time_seconds{quantile="0.95"} 3.5
openclaw_agent_sessions_active{agent="main"} 12
```

---

## 🚨 **アラートと通知システム**

### **8.10 アラートルール設定**

#### **Prometheusアラートルール**
```yaml
# openclaw-alerts.yml
groups:
  - name: openclaw.rules
    rules:
      - alert: HighErrorRate
        expr: rate(openclaw_gateway_errors_total[5m]) > 0.1
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "高いエラー率を検出"
          description: "エラー率が2分間にわたって10%を超えています"

      - alert: HighResponseTime
        expr: openclaw_gateway_response_time_seconds{quantile="0.95"} > 5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "レスポンス時間が長い"
          description: "95パーセンタイルのレスポンス時間が5秒を超えています"

      - alert: AgentDown
        expr: openclaw_agent_status == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Agentがダウン"
          description: "Agent {{ $labels.agent_id }} が1分間ダウンしています"
```

### **8.11 インテリジェントアラート戦略**

#### **アラートの段階的エスカレーション**
```json
{
  "alerting": {
    "levels": [
      {
        "name": "info",
        "channels": ["log"],
        "escalation": false
      },
      {
        "name": "warning",
        "channels": ["slack", "email"],
        "escalation": {
          "after": "15m",
          "to": "critical"
        }
      },
      {
        "name": "critical",
        "channels": ["telegram", "phone", "pager"],
        "escalation": {
          "after": "5m",
          "to": "emergency"
        }
      }
    ]
  }
}
```

---

## 📊 **可視化監視パネル**

### **8.12 Grafana Dashboard**

```json
{
  "dashboard": {
    "title": "OpenClaw System Overview",
    "panels": [
      {
        "title": "Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(openclaw_gateway_requests_total[5m])",
            "legendFormat": "{{channel}}"
          }
        ]
      },
      {
        "title": "Response Time Percentiles",
        "type": "graph",
        "targets": [
          {
            "expr": "openclaw_gateway_response_time_seconds{quantile=\"0.5\"}",
            "legendFormat": "p50"
          },
          {
            "expr": "openclaw_gateway_response_time_seconds{quantile=\"0.95\"}",
            "legendFormat": "p95"
          }
        ]
      },
      {
        "title": "Active Sessions",
        "type": "singlestat",
        "targets": [
          {
            "expr": "sum(openclaw_agent_sessions_active)"
          }
        ]
      }
    ]
  }
}
```

---

## 🔧 **障害診断とデバッグ**

### **8.14 障害診断フロー**

#### **障害分類デシジョンツリー**
```
障害報告 →
├── ユーザーがアクセスできない？
│   ├── Gatewayステータスを確認 → openclaw status
│   ├── チャネル接続を確認 → openclaw status --channels
│   └── ネットワーク接続を確認 → ping/traceroute
│
├── レスポンスが遅い？
│   ├── システム負荷を確認 → top/htop
│   ├── Agentパフォーマンスを確認 → openclaw logs --level performance
│   └── メモリ使用量を確認 → openclaw memory stats
│
└── 機能が異常？
    ├── エラーログを確認 → openclaw logs --level error
    ├── 設定の問題を確認 → openclaw doctor
    └── モデルステータスを確認 → openclaw status --models
```

#### **自動化障害診断スクリプト**
```bash
#!/bin/bash
# openclaw-troubleshoot.sh

echo "🔍 OpenClaw自動障害診断"
echo "========================="

# 1. 基本的な接続性チェック
echo "📡 Gateway接続性を確認中..."
if ! curl -s http://localhost:18789/health > /dev/null; then
    echo "❌ Gatewayが応答しません。サービスステータスを確認してください"
    systemctl --user status openclaw-gateway
    exit 1
fi
echo "✅ Gatewayは正常に稼働中"

# 2. チャネルステータスチェック
echo "📱 チャネルステータスを確認中..."
CHANNELS=$(openclaw status --json | jq -r '.channels | to_entries[] | select(.value.status != "OK") | .key')
if [[ -n "$CHANNELS" ]]; then
    echo "⚠️ チャネルの問題を検出: $CHANNELS"
else
    echo "✅ すべてのチャネルが正常"
fi

# 3. エラーログ分析
echo "📋 最近のエラーを確認中..."
ERROR_COUNT=$(openclaw logs --level error --since "1h" --json | jq '. | length')
if [[ "$ERROR_COUNT" -gt 10 ]]; then
    echo "⚠️ ${ERROR_COUNT} 件のエラーを検出"
    openclaw logs --level error --limit 5
fi

echo ""
echo "🛠️ 自動修復の提案:"
echo "  1. Gatewayの再起動: openclaw gateway restart"
echo "  2. キャッシュのクリア: openclaw memory cache clear"
echo "  3. 設定の確認: openclaw doctor --non-interactive"
echo "  4. 詳細ログの確認: openclaw logs --follow"
```

---

## 🤖 **自動化運用**

### **8.16 自己修復システム設計**

```bash
#!/bin/bash
# auto-heal.sh - OpenClaw自己修復スクリプト

HEALTH_CHECK_URL="http://localhost:18789/health"
MAX_RETRIES=3
RESTART_COOLDOWN=300  # 5分のクールダウン

check_health() {
    local response=$(curl -s -w "%{http_code}" -o /dev/null "$HEALTH_CHECK_URL")
    [[ "$response" == "200" ]]
}

restart_gateway() {
    echo "$(date): Gateway異常を検出、再起動準備中..."
    openclaw gateway stop --graceful --timeout 30s
    sleep 5
    openclaw gateway start --background
    sleep 10

    if check_health; then
        echo "$(date): Gateway再起動成功"
        return 0
    else
        echo "$(date): Gateway再起動失敗"
        return 1
    fi
}

# メインループ
while true; do
    if ! check_health; then
        echo "$(date): Gatewayヘルスチェック失敗"

        for i in $(seq 1 $MAX_RETRIES); do
            echo "$(date): 再起動を試行 ($i/$MAX_RETRIES)"
            if restart_gateway; then
                break
            fi

            if [[ $i -eq $MAX_RETRIES ]]; then
                echo "$(date): 再起動失敗、アラートを送信"
                curl -X POST "$ALERT_WEBHOOK" \
                     -d '{"level":"critical","message":"OpenClaw Gatewayの再起動に失敗しました"}'
            fi
            sleep $RESTART_COOLDOWN
        done
    fi
    sleep 60  # 1分ごとにチェック
done
```

### **8.17 バックアップとリストアの自動化**

```bash
#!/bin/bash
# backup-automation.sh

BACKUP_DIR="/backup/openclaw/$(date +%Y%m%d)"
RETENTION_DAYS=30

mkdir -p "$BACKUP_DIR"

# 設定ファイルのバックアップ
echo "設定ファイルをバックアップ中..."
tar -czf "$BACKUP_DIR/config-$(date +%H%M).tar.gz" \
    ~/.openclaw/openclaw.json \
    ~/.openclaw/auth-profiles.json

# メモリファイルのバックアップ
echo "メモリファイルをバックアップ中..."
tar -czf "$BACKUP_DIR/memory-$(date +%H%M).tar.gz" \
    ~/.openclaw/workspace-main/memory/

# 古いバックアップのクリーンアップ
find /backup/openclaw -type d -mtime +$RETENTION_DAYS -exec rm -rf {} +

echo "✅ バックアップ完了: $BACKUP_DIR"
```

---

## 📋 **本章のまとめ**

### **コアポイント**
1. **階層型監視**: アプリケーション層、Gateway層、システム層、インフラストラクチャ層
2. **包括的なオブザーバビリティ**: メトリクス、ログ、トレースの3本柱
3. **インテリジェントアラート**: 段階的アラート、エスカレーション戦略、サイレンスメカニズム
4. **自動化運用**: 自己修復システム、動的スケーリング、バックアップリストア

### **監視チェックリスト**
- ✅ **基本メトリクス**: CPU、メモリ、ディスク、ネットワーク
- ✅ **アプリケーションメトリクス**: リクエスト率、レスポンス時間、エラー率、同時接続数
- ✅ **ビジネスメトリクス**: ユーザーアクティビティ、Token使用量、チャネル分布
- ✅ **セキュリティメトリクス**: 認証失敗、異常アクセス、権限変更

### **次の学習ステップ**
- 第9章: マルチサーバーデプロイアーキテクチャ

---

**🔗 関連リソース:**
- [OpenClaw監視ドキュメント](https://docs.openclaw.ai/monitoring)
- [Prometheus OpenClaw Exporter](https://github.com/openclaw/prometheus-exporter)
- [Grafana Dashboardテンプレート](https://grafana.com/dashboards/openclaw)
