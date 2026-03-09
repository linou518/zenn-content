---
title: "7000行のSPAから500行へ——「売れる軽量ダッシュボード」をゼロから作り直した話"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "自動化"]
published: true
---

# 7000行のSPAから500行へ——「売れる軽量ダッシュボード」をゼロから作り直した話

## はじめに

社内で運用しているOpenClaw Dashboard（タスク管理・ノード監視・家族カレンダー・AI使用量…）は気がつけば7000行を超えるSPAになっていた。機能は豊富だが、「miniPCに同梱して顧客に渡す製品」としては完全にオーバースペックだった。

今回はそこから必要な機能だけを抽出し、**バックエンド500行・フロントエンド37KB**の独立した軽量版「Dashboard Lite」を1時間で作り直した記録。

---

## 問題：肥大化したSPAを「製品として売る」難しさ

元のDashboardには以下が詰め込まれていた：

- 今日のスケジュール・タイムライン
- 家族の記念日カレンダー
- 複数ノードの一括監視（8台分のOpenClaw状態）
- AI（Claude）の使用量グラフ
- 外部APIとの連携（OCM Server, ホーム automation等）

これを顧客のminiPCに入れようとすると問題が噴出する：

1. **依存が多い** — 社内の複数サービス（192.168.3.33:8001, :8091 etc.）を前提に書かれている
2. **ブランドが露出** — 「TechsFree」「家族」などプライベートな要素が混在
3. **設定が複雑** — どのAPIを無効化すべきか、どの定数を書き換えるべきか不明瞭

「既存コードを削ぎ落として製品版にする」より「要件を定義して書き直す」方が速いと判断した。

---

## アプローチ：ゼロから書く、依存を持ち込まない

### 要件の絞り込み

顧客に渡すminiPCで必要なのは2機能だけ：

1. **プロジェクトタスク管理** — シンプルなTodo（作成・完了・削除）
2. **ノード管理（本機のみ）** — 本機のOpenClaw状態・Botリスト・ログ確認

タイムラインも家族カレンダーもAI使用量も要らない。**「要らないもの」を定義することが設計の第一歩**だった。

### バックエンド：Flask、依存1個

```python
# requirements.txt
flask
```

それだけ。元のserver.pyはflask以外にrequests、websocketなど複数の依存があった。Liteは`flask`一本で完結させた。

エンドポイントも最小限：

```
GET  /api/simple-tasks          # タスク一覧
POST /api/simple-tasks/toggle   # 完了トグル
POST /api/simple-tasks/add      # タスク追加
POST /api/node/status           # 本機ノード状態
GET  /api/node/bots             # Bot一覧
POST /api/node/restart-gateway  # Gateway再起動
```

ノード情報は外部APIを叩かず、**ローカルの`openclaw.json`を直読み**する方式にした。ネットワーク依存ゼロ。

```python
OC_PATH = os.environ.get('OC_PATH') or detect_oc_path()

def detect_oc_path():
    candidates = [
        Path.home() / '.openclaw',
        Path('/etc/openclaw'),
    ]
    for p in candidates:
        if (p / 'openclaw.json').exists():
            return str(p)
    return str(Path.home() / '.openclaw')
```

環境変数で上書き可能にしておくことで、非標準インストール先にも対応できる。

### フロントエンド：ブランドを剥がして「汎用感」を出す

元のDashboardはTechsFree専用カラー（オレンジ系）とロゴを使っていた。製品版では：

- アクセントカラーを**パープル系**に変更（ニュートラルで押し付けがましくない）
- ロゴを「OpenClaw Dashboard」のテキストロゴに
- フォントをシステムフォント（`-apple-system, BlinkMacSystemFont`）に統一

結果、「誰の会社のツール感」がなく、「よくできた汎用ツール」に見える外観になった。

---

## デプロイ：install.shで1コマンド完結

```bash
#!/bin/bash
PORT=${1:-8099}
INSTALL_DIR="$HOME/.dashboard-lite"

pip3 install flask --quiet
mkdir -p "$INSTALL_DIR"
cp server.py index.html project_tasks.json "$INSTALL_DIR/"

# systemdサービス登録
cat > ~/.config/systemd/user/dashboard-lite.service << EOF
[Unit]
Description=Dashboard Lite
After=network.target

[Service]
WorkingDirectory=$INSTALL_DIR
ExecStart=python3 $INSTALL_DIR/server.py
Environment=LITE_PORT=$PORT
Restart=always

[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user enable --now dashboard-lite.service
echo "Dashboard Lite started at http://localhost:$PORT"
```

`bash install.sh 8099` を実行するだけ。systemdのuserサービスなのでroot不要、sudo不要。

---

## 結果と学び

| 指標 | 元のDashboard | Dashboard Lite |
|------|-------------|----------------|
| バックエンド行数 | ~7000行 | ~500行 |
| フロントエンドサイズ | 100KB超 | 37KB |
| 外部依存（Python） | flask, requests, websockets... | flask のみ |
| 外部APIへの依存 | 5+ エンドポイント | 0（ローカルのみ） |
| セットアップコマンド | 複数手順 | 1コマンド |

**「製品として出荷できるか」という視点は、コードの設計を根本から変える。**

内部ツールは「動けばいい」で積み重なる。外部に出すものは「誰でも動かせる」が必須条件になる。その差を埋めるのは、依存の断ち切りと、ゼロから書く勇気だった。

---

## タグ案

`#Web開発` `#Flask` `#SPA` `#リファクタリング` `#プロダクト設計` `#Python` `#TechsFree` `#OpenClaw`

