---
title: "Multi-Agent Clusterを落とさないために — 実運用で踏んだ2つの地雷と対策"
emoji: "🛡️"
type: "tech"
topics: ["OpenClaw", "infrastructure", "multiagent", "macOS", "selfhosted"]
published: true
---

> 9ノード・20+エージェントのOpenClawクラスタを運用して学んだ、耐障害性の実践記録。

## はじめに

自宅で9台のGMK Mini PCとMac Miniを使い、20以上のAIエージェントを24時間稼働させている。業務自動化、学習、家庭サポートまで、用途はバラバラだが共通の課題がある：**落ちたら困る**。

今日だけで2つの障害に遭遇し、対策を打った。どちらも「あるある」だが、事前に防げたはずのものだった。

## 地雷1: Anthropic API過負荷 — 全エージェント一斉沈黙

### 何が起きたか

Claude Opus APIが過負荷状態に。OpenClawのGatewayはバックオフリトライするが、連続失敗するとセッションが中断する。fallbackモデルが未設定だったため、**全9ノードの全エージェントが同時に応答不能**になった。

単一障害点（SPOF）の教科書的な例。APIプロバイダーに依存する以上、避けられないリスクだ。

### 対策: Codex Fallbackの全クラスタ一括配置

OpenClawは `model.fallbacks` でフォールバックモデルを指定できる。今回はOpenAI Codexをfallbackモデルとして選択した。

**手順:**

1. **認証プロファイルの伝播** — メインノードの認証情報からOAuthトークンを抽出し、全ノードの `auth-profiles.json` に注入

2. **設定の一括変更** — Pythonスクリプトで全ノードの `openclaw.json` を更新:
```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "anthropic/claude-opus-4-6",
        "fallbacks": ["openai-codex/gpt-5.3-codex"]
      }
    }
  }
}
```

3. **全ノードGateway再起動＋検証** — 9/9ノード成功

### 効果

Anthropic APIが落ちても、エージェントは自動的にCodexに切り替わる。ユーザーからはエラーが見えない。品質は若干落ちるが、**沈黙よりはるかにマシ**。

### 学び

- **fallbackなしの本番運用は裸で走っているのと同じ。** APIプロバイダーの可用性を自分でコントロールできない以上、代替経路は必須。
- 認証プロファイルをテンプレート化しておくと、新ノード追加時にもすぐ適用できる。共有ストレージにテンプレートとして配置済み。

## 地雷2: macOS sleep=1 — Mac Mini M1が1分ごとに死ぬ

### 何が起きたか

Mac Mini M1ノード上のエージェントが応答しなくなった。ログを調べると：

- Slack WebSocketが約30分ごとにstale（切断）
- 今日だけで7回断線（07:18〜10:21）
- health-monitorが検知して自動再起動するが、断線中のメッセージは消失

さらに、旧ノードから移行した際に古い設定が残っており、**2つのGatewayが同じSlackトークンで接続**していた。

### 根本原因

```bash
$ pmset -g | grep sleep
 sleep           1  # ← macOSが1分で寝る
```

Mac Miniをサーバーとして使っているのに、デフォルトのスリープ設定のまま。ネットワークが切れ、WebSocketが死に、Gatewayが断線する。

### 対策

```bash
# スリープ完全無効化
sudo pmset -a sleep 0 displaysleep 0 disksleep 0

# 重複launchd service削除
sudo launchctl bootout system/com.openclaw.gateway
sudo rm /Library/LaunchDaemons/com.openclaw.gateway.plist

# Gateway再起動
launchctl kickstart -k gui/501/ai.openclaw.gateway
```

### 学び

- **macOSをサーバー用途で使うなら `pmset -a sleep 0` は初日にやれ。** これは「常識」のはずだが、セットアップ時に忘れた。
- launchdサービスの新旧plistが共存すると、片方がcrash loopを起こしてリソースを食う。移行後は必ず旧サービスを掃除する。
- macOSのGatewayログは `~/.openclaw/logs/gateway.log` にある。トラブル時に最初に見る場所を間違えると時間を浪費する。

## まとめ: クラスタ耐障害性チェックリスト

| チェック項目 | 対策 |
|---|---|
| APIプロバイダー障害 | fallbackモデル設定（別プロバイダー） |
| ノードのスリープ/電源管理 | サーバー用途なら初日に無効化 |
| 旧設定の残留 | 移行後に旧サービス・旧設定を掃除 |
| 認証情報の伝播 | テンプレート化して共有ストレージに配置 |
| WebSocket断線検知 | health-monitorの設定確認 |

派手な技術ではないが、**これらを全部やっておくだけで深夜3時の緊急対応が激減する**。自動化の仕組みを作る前に、まず「落ちない」基盤を作るほうが先だ。

<!-- 本文已脱敏処理: IP/密码/Token等敏感信息已替換 -->
