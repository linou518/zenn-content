---
title: "OpenClaw完全ガイド第2章：環境準備とインストール - ゼロからの構築手順"
emoji: "🛠️"
type: "tech"
topics: ["openclaw", "ai-agent", "automation", "llm"]
published: true
---

# 第2章：環境準備とインストール

> 🎯 学習目標：OpenClawのインストールと設定を完了し、最初のAgentの起動に成功する

## 🔍 **環境準備チェックリスト**

始める前に、環境が以下の要件を満たしていることを確認してください：

### **📋 ハードウェア要件**

| デプロイモード | CPU | メモリ | ディスク | ネットワーク |
|----------|-----|------|------|------|
| 開発・学習 | 2コア | 4GB | 20GB | 安定したインターネット |
| 小規模本番 | 4コア | 8GB | 50GB | 10Mbps以上 |
| エンタープライズ | 8コア以上 | 16GB以上 | 100GB以上 | 100Mbps以上 |

### **💻 対応OS**

✅ **Ubuntu 22.04 LTS** (推奨)
✅ **Debian 11以上**
✅ **CentOS 8以上**
✅ **macOS 12以上**
✅ **Windows 11** (WSL2)

### **🛠️ 必須ソフトウェア**

- **Node.js 18以上** (必須)
- **npm 8以上** (必須)
- **Git 2.0以上** (必須)
- **Docker 20以上** (オプション、コンテナ化用)
- **SSL証明書** (オプション、HTTPS用)

---

## 🚀 **インストール方法の比較**

| 方法 | 適用シナリオ | 利点 | 欠点 | 所要時間 |
|------|----------|------|------|------|
| **ワンクリックスクリプト** | 初心者の迅速な体験 | 全自動化 | カスタマイズ性が低い | 5-10分 |
| **手動インストール** | 本番環境 | 完全な制御 | 手順が多い | 20-30分 |
| **Dockerデプロイ** | コンテナ環境 | 環境分離 | Docker知識が必要 | 10-15分 |
| **クラスターデプロイ** | エンタープライズ | 高可用性 | 複雑さが高い | 1-2時間 |

---

## 🎯 **方法1: ワンクリックインストールスクリプト** ⭐初心者推奨

### **スクリプトのダウンロードと実行**
```bash
# インストールスクリプトをダウンロード
curl -sSL https://raw.githubusercontent.com/openclaw/openclaw/main/scripts/install.sh -o install-openclaw.sh

# スクリプトの内容を確認（推奨）
cat install-openclaw.sh

# インストールスクリプトを実行
chmod +x install-openclaw.sh
./install-openclaw.sh
```

### **スクリプトの機能**
✅ OSの自動検出
✅ Node.jsと依存関係のインストール
✅ OpenClaw CLIのダウンロード
✅ ワークディレクトリ構造の作成
✅ 初期設定ファイルの生成
✅ 環境変数の設定

### **インストール過程のプレビュー**
```
🔍 OSを検出中: ubuntu
✅ Node.jsバージョン要件を満たしています: v20.10.0
📦 OpenClaw CLIをインストール中...
📁 ワークディレクトリを作成: ~/.openclaw/workspace-main
⚙️  設定ファイルを生成: openclaw.json
🎉 インストール完了！
```

---

## 🛠️ **方法2: 手動ステップバイステップインストール**

### **ステップ1: Node.jsのインストール**

#### **Ubuntu/Debian**
```bash
# NodeSourceリポジトリを追加
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -

# Node.jsをインストール
sudo apt-get install -y nodejs

# インストールを確認
node --version  # v20.x.x と表示されるはず
npm --version   # 9.x.x と表示されるはず
```

#### **CentOS/RHEL**
```bash
# NodeSourceリポジトリを追加
curl -fsSL https://rpm.nodesource.com/setup_20.x | sudo bash -

# Node.jsをインストール
sudo yum install -y nodejs

# インストールを確認
node --version
npm --version
```

#### **macOS**
```bash
# Homebrewを使用
brew install node

# または公式インストーラーをダウンロード
# https://nodejs.org/en/download/
```

### **ステップ2: OpenClaw CLIのインストール**
```bash
# OpenClawをグローバルにインストール
sudo npm install -g openclaw

# インストールを確認
openclaw --version
openclaw help
```

### **ステップ3: ワークディレクトリの作成**
```bash
# OpenClawメインディレクトリを作成
mkdir -p ~/.openclaw/workspace-main
cd ~/.openclaw/workspace-main

# Gitリポジトリを初期化
git init
git config --local user.name "Your Name"
git config --local user.email "your.email@example.com"

# ディレクトリ構造を作成
mkdir -p {memory,skills,projects,logs}
```

### **ステップ4: 設定の初期化**
```bash
# インタラクティブ設定ウィザード
openclaw onboard

# または手動で設定ファイルをセットアップ
openclaw setup

# または手動でopenclaw.jsonを作成
cat > openclaw.json << 'EOF'
{
  "gateway": {
    "port": 18789,
    "bind": "loopback"
  },
  "agents": [
    {
      "id": "main",
      "name": "メインアシスタント",
      "model": "anthropic/claude-sonnet-4-20250514",
      "workspace": {
        "root": "."
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
  }
}
EOF
```

---

## 🐳 **方法3: Dockerコンテナ化デプロイ**

### **Dockerfileの作成**
```dockerfile
# Node.js公式イメージをベースに
FROM node:20-slim

# ワークディレクトリを設定
WORKDIR /app

# システム依存関係をインストール
RUN apt-get update && apt-get install -y \
    git \
    curl \
    && rm -rf /var/lib/apt/lists/*

# OpenClawをインストール
RUN npm install -g openclaw

# ワークディレクトリを作成
RUN mkdir -p /app/workspace
WORKDIR /app/workspace

# 設定ファイルをコピー
COPY openclaw.json .
COPY auth-profiles.json .

# ポートを公開
EXPOSE 18789

# 起動コマンド
CMD ["openclaw", "gateway", "start", "--port", "18789"]
```

### **Docker Composeデプロイ**
```yaml
version: '3.8'

services:
  openclaw-gateway:
    build: .
    ports:
      - "18789:18789"
    volumes:
      - ./workspace:/app/workspace
      - ./logs:/app/logs
    environment:
      - NODE_ENV=production
    restart: unless-stopped

  openclaw-agent-1:
    build: .
    ports:
      - "18790:18789"
    volumes:
      - ./agents/agent-1:/app/workspace
    environment:
      - AGENT_ID=agent-1
      - NODE_ENV=production
    restart: unless-stopped
```

---

## 🔧 **APIキーの設定**

インストール完了後、AIモデルのAPIキーを設定する必要があります：

### **Anthropic Claudeの設定**
```bash
# インタラクティブ設定
openclaw auth add anthropic

# または環境変数を直接設定
export ANTHROPIC_API_KEY="sk-ant-api03-..."

# 設定を確認
openclaw auth list
```

### **OpenAI GPTの設定**
```bash
# OpenAI認証を追加
openclaw auth add openai

# APIキーを設定
export OPENAI_API_KEY="sk-..."

# 接続をテスト
openclaw test openai
```

### **その他のモデル設定**
```bash
# Google Gemini
export GOOGLE_API_KEY="..."
openclaw auth add google

# Azure OpenAI
export AZURE_OPENAI_API_KEY="..."
export AZURE_OPENAI_ENDPOINT="https://..."
openclaw auth add azure-openai
```

---

## 🎮 **初回起動とテスト**

### **Gatewayの起動**
```bash
# ワークディレクトリに移動
cd ~/.openclaw/workspace-main

# Gatewayを起動（フォアグラウンド実行）
openclaw gateway start

# またはバックグラウンド実行
openclaw gateway start --daemon

# ステータスを確認
openclaw status
```

### **基本機能のテスト**
```bash
# 別のターミナルでテスト
curl -X POST http://localhost:18789/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "main",
    "messages": [
      {"role": "user", "content": "Hello, OpenClaw!"}
    ]
  }'
```

### **期待されるレスポンス**
```json
{
  "id": "chatcmpl-...",
  "object": "chat.completion",
  "model": "main",
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "こんにちは！OpenClaw AIアシスタントです。お手伝いできることがあれば、お気軽にどうぞ！"
      }
    }
  ]
}
```

---

## ❗ **よくある問題と解決策**

### **問題1: Node.jsバージョンが古い**
```bash
# エラーメッセージ
Error: OpenClaw requires Node.js 18 or higher

# 解決策
# 旧バージョンをアンインストール
sudo apt remove nodejs npm

# 最新バージョンを再インストール
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs
```

### **問題2: 権限エラー**
```bash
# エラーメッセージ
EACCES: permission denied, mkdir '/usr/local/lib/node_modules/@openclaw'

# 解決策：npmグローバルディレクトリを設定
mkdir ~/.npm-global
npm config set prefix '~/.npm-global'
echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
npm install -g @openclaw/openclaw
```

### **問題3: ポートが使用中**
```bash
# エラーメッセージ
Error: listen EADDRINUSE :::18789

# 解決策：ポートを使用しているプロセスを特定
sudo netstat -tlnp | grep 18789
sudo kill -9 <PID>

# または別のポートを使用
openclaw gateway start --port 18790
```

### **問題4: APIキーが無効**
```bash
# エラーメッセージ
AuthenticationError: Invalid API key

# 解決策：キーを再設定
openclaw auth remove anthropic
openclaw auth add anthropic
# 正しいAPIキーを入力

# 設定を確認
openclaw auth test anthropic
```

### **問題5: ネットワーク接続の問題**
```bash
# エラーメッセージ
fetch failed

# 解決策：ネットワークとファイアウォールを確認
# ネットワーク接続をテスト
curl -I https://api.anthropic.com
curl -I https://api.openai.com

# プロキシを設定（必要な場合）
export HTTPS_PROXY=http://proxy.company.com:8080
export HTTP_PROXY=http://proxy.company.com:8080
```

---

## ✅ **インストール確認チェックリスト**

インストール完了後、以下を一つずつ確認してください：

- [ ] **Node.jsバージョン**: `node --version` >= v18.0.0
- [ ] **OpenClaw CLI**: `openclaw --version` でバージョン情報が表示される
- [ ] **ワークディレクトリ**: `~/.openclaw/workspace-main` が存在する
- [ ] **設定ファイル**: `openclaw.json` のフォーマットが正しい
- [ ] **APIキー**: `openclaw auth list` で設定済みのプロバイダーが表示される
- [ ] **Gateway起動**: `openclaw gateway start` でエラーなし
- [ ] **APIテスト**: HTTPリクエストが正常なレスポンスを返す
- [ ] **ログ記録**: `logs/gateway.log` に正常なログがある

---

## 📊 **パフォーマンス最適化の提案**

### **システムレベルの最適化**
```bash
# ファイルディスクリプタ制限を増やす
echo "* soft nofile 65536" | sudo tee -a /etc/security/limits.conf
echo "* hard nofile 65536" | sudo tee -a /etc/security/limits.conf

# Node.jsメモリを最適化
export NODE_OPTIONS="--max-old-space-size=4096"

# ハイパフォーマンスモードを有効化
echo performance | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

### **OpenClaw設定の最適化**
```json
{
  "gateway": {
    "maxConcurrency": 10,
    "timeout": 30000,
    "keepAlive": true
  },
  "agents": [
    {
      "id": "main",
      "maxConcurrency": 3,
      "contextPruning": {
        "enabled": true,
        "maxTokens": 100000
      }
    }
  ]
}
```

---

## 🚀 **次のステップ**

おめでとうございます！OpenClawのインストールが完了しました。次に進みましょう：

**[次の章：最初のAgentを作成 →](../chapter3/README.md)**

---

## 📝 **本章のまとめ**

本章の学習を通じて、以下を完了しているはずです：

- [x] OpenClawのインストールと基本設定の完了
- [x] ワンクリックスクリプトと手動インストールの2つの方法の習得
- [x] AIモデルのAPIキー設定
- [x] OpenClaw Gatewayの起動成功
- [x] よくある問題の解決策の理解
- [x] パフォーマンス最適化の方法の把握

**最初のAgentを作成する準備はできましたか？** 🎯
