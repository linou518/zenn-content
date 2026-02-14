---
title: "OpenClaw完全ガイド第3章：最初のAgentを作成する - 実践チュートリアル"
emoji: "🤖"
type: "tech"
topics: ["openclaw", "ai-agent", "automation", "llm"]
published: true
---

# 第3章：最初のAgentを作成する

> 🎯 学習目標：最初のAI Agentの作成と設定に成功し、基本的な会話インタラクションを行う

## 🤖 **Agentとは？**

OpenClawにおけるAgentは、固有のアイデンティティ、メモリ、能力を持つAIインスタンスです。各Agentは専門のAIアシスタントのようなもので、以下が可能です：

- 🧠 独立したメモリと学習能力を持つ
- 🛠️ 特定のツールセットを使用する
- 📁 自分のワークスペースを管理する
- 💬 異なるチャネルを通じてユーザーと対話する
- 🎯 特定のタスク領域に集中する

---

## 🏗️ **Agent設定の構造**

### **基本設定テンプレート**
```json
{
  "agents": [
    {
      "id": "my-first-agent",              // 一意の識別子
      "name": "私の最初のアシスタント",        // 表示名
      "model": "anthropic/claude-sonnet-4", // AIモデル
      "systemPrompt": "あなたは...",         // システムプロンプト
      "workspace": {
        "root": "./agents/my-first-agent"   // ワークディレクトリ
      },
      "memory": {
        "enabled": true,                    // メモリを有効化
        "maxTokens": 50000                  // メモリ容量
      },
      "tools": {
        "allowlist": ["read", "write", "exec"] // 許可するツール
      }
    }
  ]
}
```

### **設定フィールドの詳細**

| フィールド | 必須 | 説明 | 例 |
|------|------|------|------|
| `id` | ✅ | Agentの一意識別子 | "main", "coding-assistant" |
| `name` | ❌ | 表示名 | "プログラミングアシスタント", "生活管家" |
| `model` | ✅ | AIモデル | "anthropic/claude-sonnet-4" |
| `systemPrompt` | ❌ | システムプロンプト | Agentの役割と振る舞いを定義 |
| `workspace.root` | ❌ | ワークディレクトリ | "./agents/coding" |
| `tools.allowlist` | ❌ | ツールホワイトリスト | ["read", "write", "exec"] |

---

## 🎯 **実践：最初のAgentを作成**

### **ステップ1: Agentの役割を計画**

「パーソナルアシスタント」Agentを作成しましょう。機能には以下を含みます：
- 📅 スケジュール管理とリマインダー
- 📝 ノートの記録と整理
- 🌐 情報検索とサマリー
- 💻 簡単なファイル操作

### **ステップ2: ワークディレクトリの作成**
```bash
# OpenClawワークスペースに移動
cd ~/.openclaw/workspace-main

# Agent専用ディレクトリを作成
mkdir -p agents/personal-assistant
cd agents/personal-assistant

# 基本ファイル構造を作成
mkdir -p {memory,logs,projects}
touch MEMORY.md SOUL.md USER.md
```

### **ステップ3: Agent設定の作成**

`~/.openclaw/workspace-main/openclaw.json` を編集：

```json
{
  "gateway": {
    "port": 18789,
    "bind": "loopback"
  },
  "agents": [
    {
      "id": "personal-assistant",
      "name": "パーソナルアシスタント",
      "model": "anthropic/claude-sonnet-4-20250514",
      "systemPrompt": "あなたはプロフェッショナルなパーソナルアシスタントAIです。ユーザーの日常業務を管理する役割を担っています。具体的には、スケジュール調整、情報検索、ファイル整理などです。\n\n1. 主体的、効率的、親切であること\n2. ユーザーの好みや習慣を記憶すること\n3. 実用的な提案やソリューションを提供すること\n4. ユーザーのプライバシーとデータセキュリティを保護すること\n\n簡潔明瞭な言葉で回答し、必要に応じて構造化フォーマット（リスト、テーブル）を使用して可読性を向上させてください。",
      "workspace": {
        "root": "./agents/personal-assistant"
      },
      "memory": {
        "enabled": true,
        "maxTokens": 100000
      }
    }
  ],
  "auth": {
    "profiles": [
      {
        "id": "anthropic",
        "provider": "anthropic"
      }
    ]
  },
  "tools": {
    "allowlists": {
      "personal-assistant": [
        "read",
        "write",
        "exec",
        "web_search",
        "memory_search",
        "memory_get",
        "sessions_list",
        "cron"
      ]
    }
  }
}
```

### **ステップ4: Agentのアイデンティティファイルを作成**

#### **SOUL.md - Agentのパーソナリティ**
```markdown
# SOUL.md - パーソナルアシスタントのパーソナリティ特性

## コア特性
- 🎯 **主体的積極性**: ユーザーのニーズを能動的に発見し、支援を提案
- 🧠 **知的効率性**: 指示を迅速に理解し、的確なソリューションを提供
- 😊 **親切さ**: 温かいコミュニケーションスタイル、ユーザーの気持ちに配慮
- 🔒 **信頼性**: プライバシーを保護し、タスクを確実に実行

## コミュニケーションスタイル
- 簡潔明瞭な言葉を使用
- 重要な情報は構造化フォーマットで表示
- 適度に絵文字を使用して親しみやすさを向上
- 常に礼儀正しくプロフェッショナルに

## 行動原則
1. **ユーザーファースト**: ユーザーのニーズと好みが最優先
2. **主体的な思考**: 質問に答えるだけでなく、関連する提案も提供
3. **継続的な学習**: ユーザーの習慣を記憶し、サービスを継続的に改善
4. **セキュリティ最優先**: ユーザーデータとプライバシーの保護
```

#### **USER.md - ユーザー情報**
```markdown
# USER.md - ユーザー情報

## 基本情報
- **名前**: [ユーザー名]
- **タイムゾーン**: Asia/Tokyo
- **言語設定**: 日本語
- **職業**: [職業情報]

## 利用設定
- **コミュニケーションスタイル**: 簡潔で直接的
- **情報フォーマット**: リストやテーブルを好む
- **リマインダー方式**: 重要な予定の1時間前にリマインド
- **プライバシーレベル**: プライバシー保護を非常に重視

## よく使うタスク
- [ ] スケジュール管理とリマインダー
- [ ] 情報検索と整理
- [ ] ドキュメントの作成と編集
- [ ] 学習資料の収集
- [ ] 生活関連のリマインダー

## 重要事項
- 勤務時間: 9:00-18:00
- 休息時間: 22:00-8:00 (緊急時以外は通知しない)
- 重要な連絡先: [連絡先情報]
```

#### **MEMORY.md - 初期メモリ**
```markdown
# MEMORY.md - パーソナルアシスタントメモリ

## 作成記録
- **作成日時**: 2026-02-15
- **作成者**: OpenClaw新規ユーザー
- **用途**: 個人の日常業務管理

## ユーザー設定
- コミュニケーションスタイル: 簡潔で効率的
- 情報フォーマット: 構造化表示
- 作業習慣: 把握中

## 重要タスク記録
- [ ] ユーザーのOpenClaw利用を支援
- [ ] 日常ワークフローの確立
- [ ] 個人情報プロファイルの充実

## 学習メモ
- OpenClawの基礎概念を習得済み
- Agent設定とデプロイが完了
- ツールの使用方法は実践が必要
```

---

## 🚀 **Agentの起動とテスト**

### **ステップ1: OpenClaw Gatewayの起動**
```bash
# 正しいワークディレクトリにいることを確認
cd ~/.openclaw/workspace-main

# Gatewayを起動
openclaw gateway start

# 起動ログを確認
tail -f logs/gateway.log
```

### **期待されるログ**:
```
[INFO] Gateway starting on port 18789
[INFO] Loading agent: personal-assistant
[INFO] Agent personal-assistant workspace: ./agents/personal-assistant
[INFO] Tools loaded for personal-assistant: read,write,exec,web_search...
[INFO] Gateway ready for connections
```

### **ステップ2: APIテスト**
```bash
# Agentのレスポンスをテスト
curl -X POST http://localhost:18789/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "personal-assistant",
    "messages": [
      {
        "role": "user",
        "content": "こんにちは、自己紹介してください"
      }
    ]
  }'
```

### **期待されるレスポンス**:
```json
{
  "id": "chatcmpl-xxx",
  "object": "chat.completion",
  "model": "personal-assistant",
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "こんにちは！😊 あなたのパーソナルアシスタントAIです。以下のことをお手伝いできます：\n\n📅 **スケジュール管理**: 会議の調整、リマインダーの設定\n📝 **情報整理**: 資料の検索、ドキュメントの整理\n💻 **ファイル操作**: ファイルの作成、編集、管理\n🔍 **情報検索**: ウェブ情報の検索とサマリー\n\n会話の内容やお好みを記憶し、よりきめ細かいサービスを提供していきます。何かお手伝いできることはありますか？"
      }
    }
  ]
}
```

---

## 💬 **基本的な会話テスト**

### **テスト1: 基本的なインタラクション**
```bash
# リクエスト
{
  "model": "personal-assistant",
  "messages": [
    {"role": "user", "content": "今日の作業計画を作成してください"}
  ]
}

# 期待されるレスポンス: Agentが具体的なニーズを尋ね、計画作成を支援
```

### **テスト2: ツールの使用**
```bash
# リクエスト
{
  "model": "personal-assistant",
  "messages": [
    {"role": "user", "content": "ワークディレクトリにREADME.mdファイルを作成してください"}
  ]
}

# 期待: Agentがwriteツールを使用してファイルを作成
```

### **テスト3: 情報検索**
```bash
# リクエスト
{
  "model": "personal-assistant",
  "messages": [
    {"role": "user", "content": "今日の天気を調べてください"}
  ]
}

# 期待: Agentがweb_searchツールを使用して天気を検索
```

---

## 📱 **Telegram Botの統合（オプション）**

### **Telegram Botの作成**
1. **@BotFatherを見つける**: Telegramで @BotFather を検索
2. **新しいBotを作成**: `/newbot` を送信
3. **名前を設定**: 例：「マイパーソナルアシスタント」
4. **ユーザー名を設定**: 例：「my_personal_assistant_bot」
5. **Tokenを取得**: Bot Tokenを保存

### **Telegram統合の設定**
`openclaw.json` を編集し、Telegram設定を追加：

```json
{
  "gateway": {
    "port": 18789,
    "bind": "loopback"
  },
  "agents": [
    {
      "id": "personal-assistant",
      // ... その他の設定は変更なし
    }
  ],
  "telegram": {
    "accounts": [
      {
        "name": "personal-assistant-bot",
        "botToken": "1234567890:ABCDEF...",  // あなたのBot Token
        "binding": "personal-assistant"      // personal-assistantにバインド
      }
    ]
  },
  "auth": {
    "profiles": [
      {
        "id": "anthropic",
        "provider": "anthropic"
      }
    ]
  },
  "tools": {
    "allowlists": {
      "personal-assistant": [
        "read", "write", "exec", "web_search",
        "memory_search", "memory_get", "message",
        "sessions_list", "cron"
      ]
    }
  }
}
```

### **再起動とテスト**
```bash
# Telegram設定を読み込むためにGatewayを再起動
openclaw gateway restart

# TelegramであなたのBotを見つけてメッセージを送信
# Agentからの返信が届くはずです
```

---

## 🛠️ **ツール使用例**

### **ファイル操作の例**
```bash
# ユーザー: 「買い物リストを作成してください」
# Agentが実行:
write("shopping-list.md", "# 買い物リスト\n\n- [ ] パン\n- [ ] 牛乳\n- [ ] 卵")
```

### **情報検索の例**
```bash
# ユーザー: 「今日の東京の天気はどうですか？」
# Agentが実行:
web_search("東京 天気 今日")
# 検索結果を整理してユーザーに返す
```

### **メモリ操作の例**
```bash
# ユーザー: 「コーヒーが好きだということを覚えておいてください」
# Agentが実行:
memory_update("ユーザー設定", "飲み物: コーヒーが好き")
```

---

## ❗ **よくある問題の解決**

### **問題1: Agent起動に失敗**
```
[ERROR] Failed to load agent personal-assistant: Invalid model configuration
```

**解決策**:
1. `openclaw.json` の構文が正しいか確認
2. APIキー設定を確認: `openclaw auth list`
3. モデル接続をテスト: `openclaw test anthropic`

### **問題2: ツール権限エラー**
```
[ERROR] Tool 'web_search' not allowed for agent personal-assistant
```

**解決策**:
`tools.allowlists` に必要なツールを追加：
```json
"tools": {
  "allowlists": {
    "personal-assistant": ["read", "write", "exec", "web_search"]
  }
}
```

### **問題3: メモリファイルにアクセスできない**
```
[ERROR] Cannot read MEMORY.md: ENOENT
```

**解決策**:
```bash
# Agentのワークディレクトリが存在することを確認
mkdir -p agents/personal-assistant
cd agents/personal-assistant

# 必要なメモリファイルを作成
touch MEMORY.md SOUL.md USER.md
```

### **問題4: Telegram Botが応答しない**
```
[ERROR] Telegram polling failed: 401 Unauthorized
```

**解決策**:
1. Bot Tokenが正しいか確認
2. Botがアクティブか確認
3. Tokenをテスト: `curl https://api.telegram.org/bot<TOKEN>/getMe`

---

## 🎯 **Agent最適化の提案**

### **パフォーマンス最適化**
```json
{
  "id": "personal-assistant",
  "model": "anthropic/claude-sonnet-4-20250514",
  "maxConcurrency": 2,           // 同時実行数を制限
  "timeout": 30000,              // 30秒タイムアウト
  "contextPruning": {            // コンテキストプルーニング
    "enabled": true,
    "maxTokens": 50000,
    "strategy": "sliding-window"
  },
  "caching": {                   // キャッシュを有効化
    "enabled": true,
    "maxEntries": 1000
  }
}
```

### **セキュリティ設定**
```json
{
  "security": {
    "sandbox": true,             // サンドボックスを有効化
    "allowedPaths": [            // ファイルアクセスパスを制限
      "./agents/personal-assistant/*",
      "./shared/*"
    ],
    "deniedCommands": [          // 禁止コマンド
      "rm -rf",
      "sudo",
      "passwd"
    ]
  }
}
```

---

## 📊 **監視とログ**

### **Agentステータスの確認**
```bash
# すべてのAgentのステータスを確認
openclaw status

# 特定のAgentのステータスを確認
openclaw status personal-assistant

# リアルタイムログを確認
tail -f logs/agents/personal-assistant.log
```

### **パフォーマンス監視**
```bash
# システムリソース使用状況を確認
openclaw metrics

# Agent統計情報を確認
openclaw stats personal-assistant
```

---

## ✅ **本章のまとめ**

本章の学習を通じて、以下を完了しているはずです：

- [x] Agentの概念と設定構造の理解
- [x] 最初のパーソナルアシスタントAgentの作成に成功
- [x] Agentのアイデンティティファイル（SOUL.md、USER.md、MEMORY.md）の設定
- [x] 基本的な会話テストの完了
- [x] ツールの使用と権限設定の習得
- [x] Telegram Bot統合方法の理解
- [x] よくある問題の解決策の把握

---

## 🚀 **次のステップ**

おめでとうございます！動作するAI Agentを手に入れました。次に進みましょう：

**[次の章：ツールとスキルの活用 →](../chapter4/README.md)**

---

## 📝 **実践演習**

1. **パーソナライズ**: `SOUL.md` を修正して異なるパーソナリティのAgentを作成
2. **スキル拡張**: Agentに新しいツール権限を追加
3. **マルチチャネルテスト**: Telegram Botを設定してインタラクションをテスト
4. **メモリ機能**: Agentにあなたの好みと習慣を記憶させる
5. **エラーハンドリング**: 意図的に誤った設定を入力し、トラブルシューティングを練習

**さらに高度な機能を探索する準備はできましたか？** 🎯
