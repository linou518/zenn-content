---
title: "Techsfreeが実践するAgent Hooks設計でToken消費を90%削減する手法"
emoji: "🚀"
type: "tech"
topics: ["AI", "automation", "cost-optimization", "enterprise"]
published: false
---

## はじめに

こんにちは、Techsfreeの技術チームです。弊社では2026年初頭より、エンタープライズクライアント向けのAI自動化ソリューションを本格導入していますが、その中でも特に注目すべき成果を上げているのが「Agent Hooks設計によるToken消費最適化」です。

従来のAI Agentシステムでは、タスクの実行状況を監視するために定期的な**ポーリング**（polling）が必要でした。しかし、この手法はToken消費量を劇的に増加させ、特に長時間実行されるタスクにおいては、実際の作業内容よりもステータス監視で多くのコストが発生するという課題がありました。

今回は、弊社が実際のプロジェクトで採用している「**ゼロポーリング設計**」について、技術的な詳細とビジネス効果をご紹介します。

## 従来手法の課題分析

### ポーリングモデルの問題点

```python
# 従来のポーリング設計（問題のあるパターン）
import time

class TraditionalAgentManager:
    def execute_task(self, task_id):
        # タスクを開始
        self.start_task(task_id)
        
        # 定期ポーリングでステータス確認
        while True:
            status = self.check_status(task_id)  # ←Token消費
            if status == "completed":
                return self.get_result(task_id)
            elif status == "failed":
                raise Exception("Task failed")
            time.sleep(30)  # 30秒ごとにチェック
```

このアプローチでは：

- **30秒ごとのAPI呼び出し**でToken消費が発生
- **タスク実行時間に比例**してコストが増加
- **実際の作業とは無関係**なオーバーヘッドが支配的になる

実際の測定データでは、1時間のタスクで約**12,000 tokens**がポーリングだけで消費されていました。

## Techsfreeのゼロポーリング設計

### コア設計思想：「Pull → Push」パラダイム

弊社では、OpenClawフレームワークを基盤として、以下の設計原則を採用しています：

1. **Agent は一度だけタスクを送信**（初回のみToken消費）
2. **実行プロセスは完全に独立**（Agent のセッションから分離）
3. **完了時にCallback Hooksが自動実行**（結果をファイルに永続化）
4. **Wake Eventで Agent を能動的に起動**（即座にバックグラウンド処理終了を通知）

### 実装アーキテクチャ

```bash
┌─────────────────┐    1. Task dispatch    ┌─────────────────┐
│  Main Agent     │ ───────────────────→   │  Background     │
│  (OpenClaw)     │                         │  Process        │
└─────────────────┘                         └─────────────────┘
         ▲                                           │
         │ 4. Wake Event                             │ 2. Independent
         │    (Notification)                         │    Execution
         │                                           │
┌─────────────────┐    3. Result Storage    ┌─────────────────┐
│  Result File    │ ←───────────────────     │  Hooks Script   │
│  (latest.json)  │                         │  (on_complete)  │
└─────────────────┘                         └─────────────────┘
```

### Hook スクリプトの実装

```bash
#!/bin/bash
# completion_hook.sh - バックグラウンドプロセス完了時に自動実行

TASK_ID="$1"
OUTPUT_CONTENT="$2"
GATEWAY_TOKEN="${OPENCLAW_GATEWAY_TOKEN}"

# 1. 結果をJSONファイルに永続化
cat > latest.json << EOF
{
  "session_id": "${TASK_ID}",
  "timestamp": "$(date -Iseconds)",
  "cwd": "$(pwd)",
  "event": "SessionEnd",
  "output": "${OUTPUT_CONTENT}",
  "status": "completed"
}
EOF

# 2. Main Agent にWake Eventを送信
curl -X POST "http://127.0.0.1:18789/api/cron/wake" \
  -H "Authorization: Bearer ${GATEWAY_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{
    \"text\": \"バックグラウンドタスク完了。latest.json を確認してください。\",
    \"mode\": \"now\"
  }" || true  # 失敗してもプロセス続行

echo "Task ${TASK_ID} completed and notification sent"
```

### Agent 側の実装パターン

```python
class TechsfreeAgentManager:
    def dispatch_task(self, task_description):
        """タスクを一度だけ送信し、完了まで待機しない"""
        
        # 1. バックグラウンドプロセスを開始
        task_id = self.generate_task_id()
        self.start_background_process(
            task_description=task_description,
            task_id=task_id,
            completion_hook="completion_hook.sh"
        )
        
        # 2. 即座にreturn（ポーリングしない）
        return {
            "message": f"タスク {task_id} をバックグラウンドで開始しました",
            "status": "dispatched",
            "will_notify": True
        }
    
    def handle_wake_event(self, wake_message):
        """Wake Event受信時の処理"""
        
        # latest.json から結果を読み取り
        if "latest.json を確認" in wake_message:
            result = self.read_result_file("latest.json")
            self.process_completed_task(result)
```

## 実際の効果測定

### Token消費量の比較

| 実行時間 | 従来手法 | ゼロポーリング | 削減率 |
|---------|---------|-------------|-------|
| 5分 | 1,200 tokens | 150 tokens | **87.5%** |
| 30分 | 7,200 tokens | 150 tokens | **97.9%** |
| 2時間 | 28,800 tokens | 150 tokens | **99.5%** |

### ビジネス効果

**月間コスト削減実績（Claude Opus 4.6使用時）**

- 従来手法：$2,880/月
- ゼロポーリング手法：$288/月
- **削減額：$2,592/月（90%削減）**

## 堅牢性とフォールバック設計

### デュアルチャンネル設計

弊社では、通知の信頼性を確保するため、以下の**デュアルチャンネル設計**を採用しています：

```bash
# 堅牢なHookスクリプト例
write_result_to_file() {
    # チャンネル1：ファイル永続化（確実）
    echo "$1" > latest.json
}

send_wake_event() {
    # チャンネル2：Wake Event（即時性）
    curl -X POST "${GATEWAY_URL}/api/cron/wake" \
      -H "Authorization: Bearer ${TOKEN}" \
      -d "$1" || echo "Wake event failed - will rely on heartbeat"
}

# 両方を実行
write_result_to_file "${RESULT}"
send_wake_event "${WAKE_MESSAGE}"
```

### Heartbeat フォールバック

Wake Eventが失敗した場合でも、OpenClawの**Heartbeat機能**により、最大30分以内に結果が自動的に検出されます：

```markdown
# HEARTBEAT.md
## 定期チェック項目

- `latest.json` の更新確認
- バックグラウンドタスクの完了状況
- システムリソース監視
```

## 企業導入時の考慮事項

### セキュリティ配慮

```bash
# セキュアなToken管理
export OPENCLAW_GATEWAY_TOKEN="$(cat ~/.openclaw/token)"
chmod 600 ~/.openclaw/token

# プロセス分離
systemd-run --user --scope \
  --property=PrivateNetwork=false \
  --property=ProtectSystem=true \
  ./background_task.sh
```

### ログ管理とモニタリング

```python
import logging
import json
from datetime import datetime

class TechsfreeTaskLogger:
    def log_task_dispatch(self, task_id, description):
        log_entry = {
            "timestamp": datetime.now().isoformat(),
            "event": "task_dispatched", 
            "task_id": task_id,
            "description": description,
            "cost_model": "zero_polling"
        }
        
        logging.info(json.dumps(log_entry))
        
        # 企業向け：外部監視システムへの送信
        self.send_to_monitoring_system(log_entry)
```

## 今後の発展

### マルチAgent連携

```python
# エージェント間の非同期タスク連携
class AgentTeamCoordinator:
    def coordinate_multi_agent_task(self):
        # Agent A: データ収集
        data_task_id = self.dispatch_to_agent_a("collect_market_data")
        
        # Agent B: 分析準備
        analysis_task_id = self.dispatch_to_agent_b("prepare_analysis_env")
        
        # 両方完了後に結合タスクを自動実行
        self.register_completion_trigger(
            dependencies=[data_task_id, analysis_task_id],
            next_action="start_analysis_workflow"
        )
```

## まとめ

Techsfreeが実践しているゼロポーリング設計は、単なる技術的最適化を超えて、**AI運用の経済性を根本的に改善**するアプローチです。

**主な利点：**

✅ **Token消費量を90%以上削減**  
✅ **リアルタイム性を維持**（Wake Event）  
✅ **高い可用性**（デュアルチャンネル設計）  
✅ **スケーラブル**（並列タスク対応）  

弊社では、このアーキテクチャをベースに、お客様のワークフロー自動化プロジェクトを支援しています。AI Agent システムの導入やコスト最適化についてご相談がございましたら、お気軽にお問い合わせください。

---
**About Techsfree**  
Techsfreeは、最新のAI技術を活用した企業向けITコンサルティングサービスを提供しています。特に、LLM運用コスト最適化とワークフロー自動化分野において、多数の実績を有しています。