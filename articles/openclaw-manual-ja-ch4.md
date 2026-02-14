---
title: "OpenClaw完全ガイド第4章：ツールとスキルの活用 - Agent能力の拡張"
emoji: "⚙️"
type: "tech"
topics: ["openclaw", "ai-agent", "automation", "llm"]
published: true
---

# 第4章：ツールとスキルの活用

> 🎯 学習目標：OpenClaw組み込みツールの使用方法を習得し、Skillsスキルパックのインストールと使用を学び、カスタムツールを作成できるようになる

## 📚 **ツールシステム概要**

OpenClawの強みは、その豊富なツールエコシステムにあります。Agentはツールを通じて外部世界とインタラクションし、さまざまなタスクを実行できます。

### **ツールの分類**
- 🔧 **組み込みツール**: OpenClawコアが提供する基本ツール
- 🎨 **Skillsスキル**: インストール可能な拡張機能パック
- 🛠️ **カスタムツール**: ユーザーが開発した専用ツール

---

## 🔧 **組み込みツール詳解**

### **4.1 ファイル操作ツール**

#### **read - ファイル内容の読み取り**
```bash
# 基本的な使い方
read file_path="example.txt"

# 大きなファイルのページ読み取り
read file_path="large_log.txt" offset=100 limit=50

# 画像ファイルの読み取り
read file_path="screenshot.png"  # フォーマットを自動認識
```

**実際の例:**
```bash
# Agentが設定ファイルを読み取り
read file_path="~/.openclaw/openclaw.json"

# ログファイルの確認
read file_path="/var/log/openclaw.log" limit=20
```

#### **write - ファイルの作成または上書き**
```bash
# 新規ファイルの作成
write file_path="output.txt" content="Hello OpenClaw"

# 既存ファイルの上書き
write path="config.json" content='{"debug": true}'
```

**セキュリティに関する注意事項:**
- writeは既存ファイルを完全に上書きします
- 事前にreadでファイルの存在を確認することを推奨
- 重要なファイル操作の前にバックアップを取得

#### **edit - ファイルの精密編集**
```bash
# 特定の内容を置換
edit file_path="config.txt"
     old_string="debug=false"
     new_string="debug=true"
```

**ベストプラクティス:**
```bash
# 1. まず内容を読み取って確認
read file_path="config.conf"

# 2. 精密に置換（スペースを含む完全な文字列に一致）
edit file_path="config.conf"
     old_string="# production mode
server.debug = false"
     new_string="# production mode
server.debug = true"

# 3. 修正結果を確認
read file_path="config.conf"
```

### **4.2 コマンド実行ツール**

#### **exec - Shellコマンドの実行**
```bash
# 基本的なコマンド実行
exec command="ls -la"

# ワークディレクトリを指定
exec command="git status" workdir="/project/path"

# バックグラウンド実行
exec command="long_running_task" background=true

# 環境変数を設定
exec command="npm start" env={"NODE_ENV": "production"}
```

**セキュリティ設定:**
```json
// openclaw.jsonのセキュリティ設定
{
  "tools": {
    "exec": {
      "security": "allowlist",  // deny | allowlist | full
      "allowCommands": ["git", "npm", "docker", "ls", "cat"],
      "denyPaths": ["/etc", "/root", "/usr/bin/rm"]
    }
  }
}
```

#### **process - バックグラウンドプロセスの管理**
```bash
# アクティブなプロセスを一覧表示
process action="list"

# プロセスの出力を確認
process action="log" sessionId="abc123"

# プロセスに入力を送信
process action="write" sessionId="abc123" data="exit\n"

# プロセスを終了
process action="kill" sessionId="abc123"
```

**実際の活用シナリオ:**
```bash
# 長時間実行されるサービスの起動
exec command="python3 web_scraper.py" background=true

# プロセスステータスの監視
process action="poll" sessionId="scraper-session"

# リアルタイムログの確認
process action="log" sessionId="scraper-session" limit=50
```

### **4.3 ネットワークツール**

#### **web_search - ウェブ検索**
```bash
# 基本検索
web_search query="OpenClaw documentation"

# 結果数を制限
web_search query="AI assistant deployment" count=5

# 地域別検索
web_search query="人工知能" country="JP" search_lang="ja"

# 時間範囲フィルター
web_search query="OpenClaw news" freshness="pw"  # 過去1週間
```

#### **web_fetch - ウェブページコンテンツの取得**
```bash
# ウェブページの内容を取得
web_fetch url="https://docs.openclaw.ai/quickstart"

# テキストのみ抽出
web_fetch url="https://example.com" extractMode="text"

# コンテンツ長を制限
web_fetch url="https://long-article.com" maxChars=5000
```

#### **browser - ブラウザ自動化**
```bash
# ウェブページを開く
browser action="open" targetUrl="https://github.com"

# スクリーンショット
browser action="screenshot"

# 要素をクリック
browser action="act" request={"kind": "click", "ref": "login-button"}

# フォームに入力
browser action="act" request={
  "kind": "fill",
  "ref": "username-field",
  "text": "myusername"
}
```

### **4.4 メッセージツール**

#### **message - メッセージ送信**
```bash
# シンプルなメッセージを送信
message action="send" target="user123" message="Hello!"

# チャネルを指定
message action="send" channel="telegram" target="@mychat" message="Status update"

# ファイルを送信
message action="send" target="user123" media="/path/to/file.pdf"
```

#### **tts - テキスト音声変換**
```bash
# 音声ファイルを生成
tts text="OpenClawアシスタントへようこそ" channel="telegram"
```

---

## 🎨 **Skillsスキルシステム**

### **4.5 Skillsとは**

SkillsはOpenClawの拡張機能パックであり、Agentに専門領域の能力を提供します。

**コア概念:**
- **Skillパック**: SKILL.md設定と関連スクリプト/リソースを含む
- **スキル説明**: AIモデルが理解する機能説明
- **ツール統合**: Skillsはツールを呼び出して複雑なタスクを完了できる

### **4.6 Skillsのインストールと使用**

#### **ClawHubからのインストール**
```bash
# 利用可能なスキルを一覧表示
openclaw skills list

# 天気スキルをインストール
openclaw skills install weather

# インストール済みスキルを更新
openclaw skills update weather

# スキルをアンインストール
openclaw skills uninstall weather
```

#### **ローカルスキルの開発**
```bash
# スキルディレクトリを作成
mkdir -p ~/.openclaw/workspace-main/skills/my-custom-skill

# スキル設定を作成
cat > ~/.openclaw/workspace-main/skills/my-custom-skill/SKILL.md << 'EOF'
---
name: my-custom-skill
description: Custom automation for specific tasks
version: 1.0.0
---

# My Custom Skill

This skill provides automation for...

## Usage
When user asks for X, do Y using the following tools:
1. tool1 with parameters...
2. tool2 with results from step 1...
EOF
```

### **4.7 おすすめSkills**

| スキル名 | 機能 | 適用シナリオ |
|--------|------|----------|
| weather | 天気情報の取得 | 日常アシスタント、外出計画 |
| healthcheck | システム監視 | サーバー運用、セキュリティチェック |
| slack | Slack統合 | 企業コラボレーション、通知管理 |
| continuous-learning-v2 | 学習記録 | ナレッジ管理、経験の蓄積 |

#### **実際の使用例**

**天気スキルの使用:**
```bash
# Agentが自動的に天気スキルを呼び出す
User: "今日の東京の天気はどうですか？"
Agent: [天気スキルを呼び出し] -> "東京は今日曇り、15-22°C、降水確率20%"
```

**ヘルスチェックスキル:**
```bash
# システム自動ヘルスチェック
User: "サーバーの状態を確認してください"
Agent: [ヘルスチェックスキルを呼び出し] -> セキュリティレポートと最適化提案を生成
```

---

## 🛠️ **ツールのセキュリティと設定**

### **4.8 セキュリティ設定のベストプラクティス**

#### **ツール権限制御**
```json
{
  "tools": {
    "exec": {
      "security": "allowlist",
      "allowCommands": [
        "git", "npm", "yarn", "docker", "kubectl",
        "ls", "cat", "grep", "find", "ps"
      ],
      "denyCommands": ["rm", "sudo", "chmod", "chown"],
      "denyPaths": ["/etc", "/root", "/bin", "/sbin"]
    },
    "read": {
      "denyPaths": ["/etc/passwd", "/etc/shadow", "~/.ssh"]
    },
    "write": {
      "denyPaths": ["/etc", "/root", "/sys", "/proc"]
    }
  }
}
```

#### **ネットワークアクセス制御**
```json
{
  "tools": {
    "web_fetch": {
      "allowDomains": ["docs.openclaw.ai", "api.github.com"],
      "denyDomains": ["internal.company.com"],
      "maxContentSize": "10MB"
    },
    "browser": {
      "headless": true,
      "allowedDomains": ["trusted-site.com"],
      "timeout": 30000
    }
  }
}
```

### **4.9 ツール使用のベストプラクティス**

#### **エラーハンドリング**
```bash
# 良い方法: ファイルの存在を確認
read file_path="config.json"
if [[ $? -eq 0 ]]; then
  edit file_path="config.json" old_string="..." new_string="..."
else
  write file_path="config.json" content='{"default": true}'
fi
```

#### **ログ記録**
```bash
# 重要な操作をログに記録
echo "$(date): Starting backup process" >> backup.log
exec command="rsync -av /data/ /backup/" workdir="/scripts"
echo "$(date): Backup completed" >> backup.log
```

#### **リソースのクリーンアップ**
```bash
# 一時ファイルのクリーンアップ
exec command="find /tmp -name 'openclaw-*' -mtime +1 -delete"

# バックグラウンドプロセスのクリーンアップ
process action="list" | grep "completed" | process action="kill"
```

---

## 🎯 **実践演習**

### **4.10 総合ケース: 自動化レポート生成**

**要件**: 毎日のシステムステータスレポートを自動生成

```bash
#!/bin/bash
# 自動化レポート生成フロー

# 1. システム情報の収集
exec command="uptime" > system_uptime
exec command="df -h" > disk_usage
exec command="free -h" > memory_usage

# 2. サービスステータスの確認
exec command="systemctl status openclaw-gateway" > gateway_status
exec command="docker ps" > container_status

# 3. 最新ログの取得
read file_path="/var/log/openclaw.log" offset=1000 limit=100 > recent_logs

# 4. レポートの作成
write file_path="daily_report_$(date +%Y%m%d).md" content="
# システムステータスレポート - $(date)

## システム概要
$(cat system_uptime)

## ディスク使用状況
\`\`\`
$(cat disk_usage)
\`\`\`

## メモリ使用状況
\`\`\`
$(cat memory_usage)
\`\`\`

## サービスステータス
$(cat gateway_status)

## 最近のログ
\`\`\`
$(cat recent_logs)
\`\`\`
"

# 5. 通知の送信
message action="send" target="admin@company.com"
        message="日次レポートが生成されました"
        media="daily_report_$(date +%Y%m%d).md"

# 6. 一時ファイルのクリーンアップ
exec command="rm -f system_* disk_* memory_* gateway_* container_* recent_logs"
```

**定期実行の設定:**
```bash
# OpenClawのcron機能を使用
openclaw cron add --name "daily-report"
                  --schedule "0 8 * * *"
                  --command "bash report_generator.sh"
```

### **4.11 トラブルシューティングガイド**

#### **よくあるエラーと解決策**

**エラー1: execツールが拒否される**
```
Error: Command 'sudo apt update' denied by security policy
```
**解決策:**
```json
// openclaw.jsonを修正
{
  "tools": {
    "exec": {
      "security": "allowlist",
      "allowCommands": ["sudo"]  // 許可リストにsudoを追加
    }
  }
}
```

**エラー2: ファイル権限が不足**
```
Error: EACCES: permission denied, open '/etc/hosts'
```
**解決策:**
```bash
# 方法1: ファイル権限を変更
sudo chmod 644 /etc/hosts

# 方法2: sudoで実行
exec command="sudo cat /etc/hosts"
```

**エラー3: ネットワークリクエストがタイムアウト**
```
Error: Request timeout after 30000ms
```
**解決策:**
```json
{
  "tools": {
    "web_fetch": {
      "timeout": 60000,  // タイムアウト時間を延長
      "retry": 3         // リトライ回数
    }
  }
}
```

---

## 📋 **本章のまとめ**

### **重要ポイント**
1. **ツールの分類**: 組み込みツール + Skillsスキル + カスタム開発
2. **セキュリティ最優先**: ツール権限を厳密に設定し、allowlistモードを使用
3. **エラーハンドリング**: 戻り値の確認、ログ記録、グレースフルデグラデーション
4. **ベストプラクティス**: テスト検証、リソースクリーンアップ、セキュリティ設定

### **次の学習ステップ**
- 第5章: 異なるメッセージチャネルの接続方法を学ぶ
- 第6章: マルチAgent協調アーキテクチャを習得する
- 第7章: メモリ管理メカニズムを深く理解する

### **演習の提案**
1. 各組み込みツールの基本的な使い方を試す
2. 2-3個のよく使うSkillsをインストールして使用する
3. カスタム自動化スクリプトを作成する
4. 自分の環境に適したセキュリティポリシーを設定する

---

**📚 参考資料:**
- [OpenClaw公式ツールドキュメント](https://docs.openclaw.ai/tools)
- [ClawHubスキルマーケット](https://clawhub.com)
- [セキュリティ設定ベストプラクティス](https://docs.openclaw.ai/security)
