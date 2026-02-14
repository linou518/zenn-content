---
title: "OpenClaw完全ガイド第5章：メッセージチャネル統合 - Telegram/Discord/WhatsApp連携"
emoji: "📱"
type: "tech"
topics: ["openclaw", "ai-agent", "automation", "llm"]
published: true
---

# 第5章：メッセージチャネル統合

> 🎯 学習目標：Telegram/Discord/WhatsAppなどのチャネルを設定し、マルチチャネルルーティングメカニズムを理解し、クロスプラットフォームメッセージ管理を実現する

## 🌐 **チャネルアーキテクチャ概要**

OpenClawは複数のメッセージチャネルをサポートしており、AIアシスタントがユーザーが普段使用するプラットフォーム上でサービスを提供できます。

### **対応チャネルタイプ**
- 📱 **インスタントメッセージ**: Telegram, WhatsApp, Discord, Slack
- 💬 **企業向けコミュニケーション**: Microsoft Teams, Feishu/Lark, 企業WeChat
- 🌍 **その他のプラットフォーム**: iMessage, Signal, Matrix, Web UI

---

## 🏗️ **Gatewayルーティングメカニズム**

### **5.1 アーキテクチャ設計原理**

```
┌─────────────────────────────────────────┐
│           ユーザーメッセージソース           │
├─────────────┬─────────────┬─────────────┤
│  Telegram   │   Discord   │  WhatsApp   │
│   Bot API   │   Bot API   │ Business API │
└─────────────┴─────────────┴─────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│            OpenClaw Gateway             │
├─────────────────────────────────────────┤
│  • アカウント管理 (Accounts)              │
│  • メッセージルーティング (Routing Rules)   │
│  • Agentバインディング (Bindings)         │
│  • セキュリティ検証 (Auth & Permissions)   │
└─────────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│              Agent配信                   │
├─────────────┬─────────────┬─────────────┤
│   Agent-1   │   Agent-2   │   Agent-3   │
│ (パーソナル)  │ (カスタマー)  │ (テクサポ)   │
└─────────────┴─────────────┴─────────────┘
```

### **5.2 コア概念**

#### **Account（アカウント）**
- 特定のBotアカウント（Telegram Bot Tokenなど）を表す
- 各Accountは1つのチャネルタイプにのみ接続可能
- メッセージの受信と送信を担当

#### **Agent（インテリジェントエージェント）**
- メッセージを処理するAIエンティティ
- 1つのAgentが複数のAccountにサービスを提供可能
- 独立したメモリ、スキル、設定を含む

#### **Binding（バインディング）**
- どのメッセージをどのAgentにルーティングするかを定義
- ユーザーID、グループID、キーワードなどのルーティングルールをサポート
- 優先度とフォールバックルールを設定可能

---

## 📱 **Telegram統合の詳細**

### **5.3 Telegram Botの作成**

#### **ステップ1: BotFatherでBotを作成**
```
1. Telegramで @BotFather を検索
2. /newbot を送信
3. Bot名を入力: "マイAIアシスタント"
4. Botユーザー名を入力: "my_ai_assistant_bot"
5. Bot Tokenを取得: 123456789:ABCdefGHIjklMNOpqrSTUvwxYZ
```

#### **ステップ2: Bot権限の設定**
```
@BotFather に送信:
/setprivacy - Disable (グループの全メッセージを読めるように)
/setjoingroups - Enable (グループへの追加を許可)
/setcommands - Botコマンドを設定
```

#### **ステップ3: OpenClawの設定**
```json
{
  "telegram": {
    "accounts": [
      {
        "accountId": "main-assistant",
        "botToken": "123456789:ABCdefGHIjklMNOpqrSTUvwxYZ",
        "name": "メインアシスタント",
        "enabled": true,
        "pollInterval": 2000
      }
    ]
  },
  "bindings": [
    {
      "accountId": "main-assistant",
      "agentId": "main",
      "dmPolicy": "allowlist",
      "allowFrom": ["7996447774"],  // ユーザーIDホワイトリスト
      "priority": 1
    }
  ]
}
```

### **5.4 高度なTelegram機能**

#### **インラインボタンサポート**
```bash
# ボタン付きメッセージを送信
message action="send"
        target="@user123"
        message="操作を選択してください："
        buttons='[
          [{"text": "ステータス確認", "callback_data": "status"}],
          [{"text": "サービス再起動", "callback_data": "restart"}]
        ]'
```

#### **ファイル処理**
```bash
# ファイルを送信
message action="send"
        target="@user123"
        message="レポートが生成されました"
        media="/path/to/report.pdf"
        filename="システムレポート.pdf"

# 受信したファイルの処理
# OpenClawはユーザーが送信したファイルを自動的に一時ディレクトリにダウンロード
```

#### **グループ管理**
```bash
# グループ情報を取得
message action="member-info" target="-1001234567890"

# グループにメッセージを送信
message action="send"
        target="-1001234567890"  # グループID（負の数）
        message="グループステータス更新"

# 特定のメッセージに返信
message action="send"
        target="@user123"
        replyTo="message_id_123"
        message="こちらが返信です"
```

---

## 💬 **Discord統合**

### **5.5 Discord Bot設定**

#### **Discordアプリケーションの作成**
```
1. https://discord.com/developers/applications にアクセス
2. "New Application" をクリック
3. アプリケーション名を入力
4. "Bot" ページでBotを作成
5. Bot Tokenをコピー
6. 権限設定: Send Messages, Read Message History, Use Slash Commands
```

#### **OpenClawの設定**
```json
{
  "discord": {
    "accounts": [
      {
        "accountId": "discord-bot",
        "botToken": "your_discord_bot_token_here",
        "name": "Discordアシスタント",
        "enabled": true,
        "guildId": "your_guild_id"  // オプション、特定サーバーに限定
      }
    ]
  },
  "bindings": [
    {
      "accountId": "discord-bot",
      "agentId": "main",
      "dmPolicy": "allowlist",
      "allowFrom": ["user_id_1", "user_id_2"]
    }
  ]
}
```

### **5.6 Discord特有の機能**

#### **スラッシュコマンドサポート**
```bash
# スラッシュコマンドを登録
message action="send"
        channel="discord"
        target="guild_id"
        type="slash_command"
        name="status"
        description="システムステータスを確認"
```

#### **Embedメッセージ**
```bash
message action="send"
        channel="discord"
        target="channel_id"
        embed='{
          "title": "システムステータスレポート",
          "description": "すべてのサービスが正常に稼働中",
          "color": 3066993,
          "fields": [
            {"name": "CPU", "value": "45%", "inline": true},
            {"name": "メモリ", "value": "2.1GB", "inline": true}
          ]
        }'
```

---

## 📲 **WhatsApp Business統合**

### **5.7 WhatsApp Business API**

#### **Businessアカウントの申請**
```
1. WhatsApp Business APIアクセス権限を申請
2. Phone Number IDとAccess Tokenを取得
3. Webhookを設定してメッセージを受信
```

#### **OpenClawの設定**
```json
{
  "whatsapp": {
    "accounts": [
      {
        "accountId": "business-wa",
        "phoneNumberId": "your_phone_number_id",
        "accessToken": "your_access_token",
        "verifyToken": "your_verify_token",
        "name": "ビジネスWhatsApp",
        "enabled": true
      }
    ]
  }
}
```

### **5.8 WhatsApp特有の機能**

#### **テンプレートメッセージ**
```bash
# テンプレートメッセージを送信（24時間制限を回避）
message action="send"
        channel="whatsapp"
        target="+1234567890"
        template="order_confirmation"
        templateParams='["注文123", "2024-02-13"]'
```

#### **メディアメッセージ**
```bash
# 画像を送信
message action="send"
        channel="whatsapp"
        target="+1234567890"
        media="https://example.com/image.jpg"
        caption="画像の説明です"
```

---

## 🏢 **エンタープライズチャネル統合**

### **5.9 Slack統合**

#### **Slack Appの作成**
```
1. https://api.slack.com/apps にアクセス
2. 新しいアプリケーションを作成
3. OAuth権限を設定: chat:write, channels:read, users:read
4. ワークスペースにインストールしてTokenを取得
```

#### **Socket Mode設定**
```json
{
  "slack": {
    "accounts": [
      {
        "accountId": "company-slack",
        "botToken": "xoxb-your-bot-token",
        "appToken": "xapp-your-app-token",
        "socketMode": true,
        "name": "社内Slackアシスタント",
        "enabled": true
      }
    ]
  }
}
```

### **5.10 Microsoft Teams**

```json
{
  "teams": {
    "accounts": [
      {
        "accountId": "teams-bot",
        "appId": "your_app_id",
        "appPassword": "your_app_password",
        "name": "Teamsアシスタント",
        "enabled": true
      }
    ]
  }
}
```

---

## 🔀 **マルチチャネルルーティング戦略**

### **5.11 高度なルーティング設定**

#### **ユーザーベースのインテリジェントルーティング**
```json
{
  "bindings": [
    {
      "accountId": "telegram-main",
      "agentId": "personal-assistant",
      "dmPolicy": "allowlist",
      "allowFrom": ["owner_user_id"],
      "priority": 1,
      "description": "オーナー専用アシスタント"
    },
    {
      "accountId": "telegram-support",
      "agentId": "customer-service",
      "dmPolicy": "open",
      "keywords": ["help", "support", "issue"],
      "priority": 2,
      "description": "カスタマーサービスBot"
    },
    {
      "accountId": "*",  // ワイルドカード: 全アカウント
      "agentId": "fallback",
      "priority": 999,
      "description": "フォールバック処理"
    }
  ]
}
```

#### **コンテンツベースのルーティング**
```json
{
  "bindings": [
    {
      "accountId": "discord-bot",
      "agentId": "tech-support",
      "channels": ["tech-help"],  // 特定チャネルに限定
      "keywords": ["bug", "error", "crash"],
      "priority": 1
    },
    {
      "accountId": "discord-bot",
      "agentId": "general-chat",
      "channels": ["general", "random"],
      "priority": 2
    }
  ]
}
```

### **5.12 クロスチャネルメッセージ同期**

#### **メッセージブリッジ設定**
```json
{
  "bridges": [
    {
      "name": "admin-sync",
      "enabled": true,
      "sources": [
        {"account": "telegram-main", "userId": "admin_id"},
        {"account": "discord-bot", "userId": "admin_discord_id"}
      ],
      "targets": [
        {"account": "slack-company", "channel": "admin-alerts"}
      ],
      "filters": ["urgent", "alert", "critical"]
    }
  ]
}
```

#### **ユーザーアイデンティティの統一**
```json
{
  "userMapping": {
    "user123": {
      "telegram": "telegram_user_id",
      "discord": "discord_user_id",
      "slack": "slack_user_id",
      "whatsapp": "+1234567890",
      "preferences": {
        "primaryChannel": "telegram",
        "notifications": ["urgent", "daily-summary"]
      }
    }
  }
}
```

---

## 🛡️ **セキュリティと権限管理**

### **5.13 セキュリティのベストプラクティス**

#### **Tokenの安全な保管**
```bash
# 環境変数方式
export TELEGRAM_BOT_TOKEN="123456789:ABCdef..."
export DISCORD_BOT_TOKEN="your_discord_token"

# 設定ファイルでの参照
{
  "telegram": {
    "accounts": [
      {
        "accountId": "main",
        "botToken": "${TELEGRAM_BOT_TOKEN}",  // 環境変数参照
        "name": "メインアシスタント"
      }
    ]
  }
}
```

#### **段階的な権限制御**
```json
{
  "security": {
    "userRoles": {
      "admin": {
        "users": ["admin_user_id"],
        "permissions": ["all"]
      },
      "moderator": {
        "users": ["mod_user_id_1", "mod_user_id_2"],
        "permissions": ["read", "moderate", "basic_commands"]
      },
      "user": {
        "users": ["*"],  // 全ユーザー
        "permissions": ["read", "basic_commands"]
      }
    },
    "rateLimiting": {
      "maxMessagesPerMinute": 30,
      "maxCommandsPerHour": 100
    }
  }
}
```

### **5.14 監視とログ**

#### **メッセージフロー監視**
```bash
# メッセージ統計を確認
openclaw status --channels

# リアルタイムログ監視
openclaw logs --follow --filter "channel=telegram"

# エラーログのフィルタリング
openclaw logs --level error --since "1h"
```

#### **チャネルヘルスチェック**
```json
{
  "monitoring": {
    "healthChecks": [
      {
        "name": "telegram-connectivity",
        "type": "api_call",
        "target": "https://api.telegram.org/bot{token}/getMe",
        "interval": 300,  // 5分
        "timeout": 10
      },
      {
        "name": "discord-latency",
        "type": "ping",
        "threshold": 1000,  // 1秒
        "alertOnFailure": true
      }
    ]
  }
}
```

---

## 🔧 **トラブルシューティングガイド**

### **5.15 よくある問題の解決**

#### **問題1: Botが応答しない**
```bash
# Botステータスを確認
openclaw status --detailed

# Tokenの有効性を検証
curl "https://api.telegram.org/bot${BOT_TOKEN}/getMe"

# ネットワーク接続を確認
openclaw doctor --check connectivity
```

#### **問題2: メッセージルーティングエラー**
```bash
# ルーティングルールを確認
openclaw config get bindings

# ルーティングロジックをテスト
openclaw agent --simulate --from "user_id" --message "test"

# デバッグモードで実行
openclaw gateway --debug --verbose
```

#### **問題3: 権限が拒否される**
```bash
# ユーザー権限を確認
openclaw config get security.userRoles

# アカウント設定を検証
openclaw channels list --status

# 権限設定をリセット
openclaw config set security.userRoles.user.permissions '["read","basic_commands"]'
```

---

## 🎯 **実践ケース: マルチチャネルカスタマーサービスシステム**

### **5.16 エンタープライズカスタマーサービスBotアーキテクチャ**

**要件**: Telegram、Discord、Webをサポートするカスタマーサービスシステムの構築

#### **アーキテクチャ設計**
```json
{
  "agents": {
    "customer-service": {
      "name": "カスタマーサービス",
      "model": "claude-sonnet-4",
      "persona": "プロフェッショナルでフレンドリーなカスタマーサービス担当",
      "skills": ["customer-support", "order-tracking", "faq"]
    },
    "technical-support": {
      "name": "テクニカルサポート",
      "model": "claude-sonnet-4",
      "persona": "技術エキスパート、問題診断に長けている",
      "skills": ["troubleshooting", "system-diagnosis"]
    }
  },

  "telegram": {
    "accounts": [
      {
        "accountId": "customer-telegram",
        "botToken": "${CUSTOMER_BOT_TOKEN}",
        "name": "カスタマーサービスTelegram"
      }
    ]
  },

  "discord": {
    "accounts": [
      {
        "accountId": "support-discord",
        "botToken": "${SUPPORT_BOT_TOKEN}",
        "name": "テクニカルサポートDiscord"
      }
    ]
  },

  "bindings": [
    {
      "accountId": "customer-telegram",
      "agentId": "customer-service",
      "dmPolicy": "open",
      "keywords": ["order", "billing", "account"],
      "businessHours": {"start": "09:00", "end": "18:00"},
      "timezone": "Asia/Tokyo"
    },
    {
      "accountId": "support-discord",
      "agentId": "technical-support",
      "channels": ["tech-support", "bug-reports"],
      "keywords": ["error", "bug", "crash", "performance"]
    }
  ]
}
```

#### **ワークフロー自動化**
```bash
# 自動チケット作成
when user_message contains "bug report":
  create_ticket(title=extract_issue(), priority="high")
  message action="send" message="チケット #{ticket_id} が作成されました。迅速に対応いたします"

# クロスチャネルステータス同期
when ticket_status_changed:
  notify_all_channels(ticket_id, new_status)
  update_user_dashboard(ticket_id)

# インテリジェントルーティングのエスカレーション
when issue_complexity > threshold:
  transfer_to_human_agent()
  notify_manager(escalation_reason)
```

---

## 📋 **本章のまとめ**

### **コアポイント**
1. **マルチチャネルサポート**: Telegram、Discord、WhatsAppなど主要プラットフォーム
2. **インテリジェントルーティング**: ユーザー、コンテンツ、時間に基づく柔軟なルーティング戦略
3. **セキュリティメカニズム**: Token保護、段階的権限、レート制限
4. **監視と運用**: ヘルスチェック、ログ記録、障害診断

### **実装の提案**
1. **段階的デプロイ**: 単一チャネルから始め、徐々に拡張
2. **セキュリティ優先**: 権限とアクセス制御を厳密に設定
3. **監視と警告**: 完全な監視とアラートメカニズムを確立
4. **ユーザー体験**: 一貫したクロスチャネルのユーザー体験を維持

### **次の学習ステップ**
- 第6章: マルチAgent協調アーキテクチャの設計
- 第7章: メモリとデータ管理の最適化
- 第8章: 監視とデバッグ技術

---

**🔗 関連リソース:**
- [Telegram Bot API](https://core.telegram.org/bots/api)
- [Discord Developer Portal](https://discord.com/developers/docs)
- [WhatsApp Business API](https://developers.facebook.com/docs/whatsapp)
- [OpenClawチャネル設定ドキュメント](https://docs.openclaw.ai/channels)
