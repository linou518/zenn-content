---
title: "Mini PCひとつで会社専用AIアシスタント環境を構築した話"
emoji: "🖥️"
type: "tech"
topics: ["OpenClaw", "Mattermost", "Docker", "SelfHosted", "MiniPC"]
published: true
---

Mini PCひとつで会社専用AIアシスタント環境を構築した話

新しい会社を立ち上げるとき、「社用PCにソフト入れたくない」「でもAIアシスタントは使いたい」という矛盾にぶつかった。解決策：Mini PCを1台用意して、そこに全部載せる。

## 背景

会社のセキュリティポリシーで、社用Macに余計なソフトをインストールできない。でもAIアシスタント（OpenClaw）を業務で使いたい。在宅勤務で家のネットワーク内に閉じているから、Mini PC（GMK）を1台追加して専用環境を作ることにした。

必要なもの：
- ファイル共有（社用Macからドキュメントをやりとりしたい）
- チャットUI（AIアシスタントと会話するインターフェース）
- AIエージェント本体

## 構成

```
社用Mac (Finder / ブラウザ)
    │
    ├── smb://192.168.x.x/share  → Samba（ファイル共有）
    └── http://192.168.x.x:8065  → Mattermost（チャットUI）
                                        ↕
                                   OpenClaw Agent ("Joy")
```

すべてMini PC（Ubuntu 24.04、メモリ16GB）の上で動く。LLMの推論自体はクラウド（Anthropic API）なので、ローカルのスペックは最低限で済む。

## Step 1: Samba — ファイル共有

MacのFinderから `Cmd+K` → `smb://192.168.x.x/share` でアクセスできるファイル共有。設定はシンプル：

```bash
sudo apt-get install samba
sudo mkdir -p /home/openclaw/mdk-share
```

`/etc/samba/smb.conf` に共有セクションを追加：

```ini
[mdk]
   path = /home/openclaw/mdk-share
   browseable = yes
   read only = no
   valid users = openclaw
```

```bash
sudo smbpasswd -a openclaw
sudo systemctl restart smbd
```

これだけ。Macからドキュメントをドラッグ&ドロップでやりとりできる。

## Step 2: Mattermost — チャットUI（Docker）

AIアシスタントと会話するUIとしてMattermostを使う。Slackライクなチャットツールで、セルフホストできるのがポイント。

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: mattermost
      POSTGRES_USER: mmuser
      POSTGRES_PASSWORD: <your-password>
    volumes:
      - pgdata:/var/lib/postgresql/data

  mattermost:
    image: mattermost/mattermost-team-edition:latest
    depends_on:
      - db
    environment:
      MM_SQLSETTINGS_DRIVERNAME: postgres
      MM_SQLSETTINGS_DATASOURCE: "postgres://mmuser:<password>@db:5432/mattermost?sslmode=disable"
      MM_SERVICESETTINGS_SITEURL: "http://192.168.x.x:8065"
    ports:
      - "8065:8065"
    volumes:
      - mmdata:/mattermost/data

volumes:
  pgdata:
  mmdata:
```

```bash
docker compose up -d
```

ブラウザで `http://192.168.x.x:8065` を開いて初期設定。管理者アカウントを作り、Bot Account（`@assistant`）を作成してトークンを控える。

## Step 3: OpenClaw — AIエージェント

OpenClawをインストールして、MattermostのBotと接続する。

```bash
npm install -g openclaw
openclaw init
openclaw plugins install @openclaw/mattermost
```

設定ファイル（`openclaw.yaml`）でMattermost接続を定義：

```yaml
channels:
  mattermost:
    baseUrl: http://localhost:8065    # ← "url"ではなく"baseUrl"！
    botToken: <bot-token>
    dmPolicy: open
```

**ハマりポイント：** 設定キーは `url` ではなく `baseUrl`。ドキュメントと実際の挙動が微妙にずれていて、ここで30分溶かした。

エージェントの身元定義（SOUL.md）を書いて、`openclaw gateway start` で起動。Mattermostに「connected as @assistant ✅」が出れば完了。

## Step 4: エージェントにアイデンティティを与える

OpenClawのエージェントは、SOUL.mdというファイルで「誰であるか」を定義する。今回は会社の技術アシスタント「Joy」として定義した：

- 専門：クラウドアーキテクチャ（AWS/GCP/Azure）、データエンジニアリング、IaC
- 性格：専業的、結果重視、簡潔
- 使用モデル：Claude Sonnet

Mattermostに `@joy こんにちは` と打てば、その人格で応答が返ってくる。

## 所要時間と費用

- **構築時間**: 約1.5時間（Samba 10分 + Mattermost 30分 + OpenClaw 30分 + デバッグ 20分）
- **ハードウェア**: GMK Mini PC（約3万円、既存在庫を転用）
- **ランニングコスト**: LLM API利用料のみ（電気代は月数百円）

## 学んだこと

1. **UIの選択が重要**。ターミナルでCLI叩くのは開発者しか無理。Mattermostを挟むことで、非エンジニアでもチャットでAIアシスタントを使える。
2. **セルフホストの安心感**。会社のデータがSaaSに流れない。LLMのAPI呼び出しは出るが、チャット履歴やファイルはすべてローカル。
3. **Mini PCで十分**。LLM推論はクラウドなので、ローカルはAPI中継とチャットUIを動かすだけ。16GBメモリで余裕。
4. **`baseUrl` は `url` じゃない**。ドキュメントを信じつつ、動かなかったらソースを読む。

## まとめ

「社用PCを汚さずにAIアシスタントを使いたい」というニーズに対して、Mini PC + Samba + Mattermost + OpenClawという構成で解決した。構築は1.5時間、追加費用はほぼゼロ（既存機材転用）。

在宅勤務でローカルネットワーク内に閉じているから成り立つ構成だが、将来的にはVPN + リバースプロキシでリモートアクセスも視野に入れている。
