---
title: "SPA × Flaskでミニマル管理ダッシュボードを作った話 — 7000行を500行に削いだリファクタリング"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "自動化"]
published: true
---

# SPA × Flaskでミニマル管理ダッシュボードを作った話 — 7000行を500行に削いだリファクタリング

先日、既存の社内ダッシュボードから「miniPC同梱版」を作る機会があった。要件はシンプル——プロジェクトタスク管理と、ローカルノードの状態確認だけでいい。タイムライン、家族カレンダー、エージェント管理……そういう機能は全部いらない。

結果として、Flask バックエンド7000行超・フロントエンド3800行超だったコードが、バックエンド約1000行・フロントエンド約2000行まで縮まった。このプロセスで学んだことを書き残しておく。

---

## なぜ「削る」のが難しいか

コードを増やすのは簡単だ。「この機能も便利そう」「あのケースも対応しよう」——積み重なっていく。削るのは難しい。どこまで削っていいかわからないし、壊れるのが怖い。

今回は制約が明確だった。**「タスク管理とノード管理だけ」**。この制約があったから迷わず削れた。

削除したもの：
- タイムライン（カレンダー連携、時間軸UI）
- 家族記念日・誕生日機能
- Claude API 使用量ダッシュボード
- 複数ノード管理（リモートノード一覧、各ノードのBot制御）
- エージェントタスク定期実行ビュー

残したもの：
- プロジェクトとタスクのCRUD
- **本機限定**のOpenClaw状態確認・Bot管理

「本機限定」がポイントだった。元のダッシュボードは複数のリモートノードを監視するマルチノード設計だったが、miniPC同梱版はそのminiPCしか管理しない。設計の前提が変わると、コードの複雑さが一気に落ちる。

---

## Flask × Single-File SPAの構成

技術スタックは変えなかった。Flask + バニラJS SPA（`index.html` 1ファイル）。

```
03_dashboard-lite/
├── server.py       # Flask バックエンド (~1000行)
├── index.html      # フロントエンド SPA (~2000行)
├── install.sh      # systemd 一括セットアップ
└── requirements.txt  # flask のみ
```

依存関係は Flask だけ。Node.js 不要、ビルドステップ不要。`pip3 install flask && python3 server.py` で動く。これは出荷する製品として重要な選択だった。

### server.py の設計思想

```python
# 環境変数で OC_PATH を自動検出
OC_PATH = os.environ.get('OC_PATH') or _detect_oc_path()

def _detect_oc_path():
    candidates = [
        Path.home() / '.openclaw',
        Path('/etc/openclaw'),
    ]
    for p in candidates:
        if (p / 'config.json').exists():
            return str(p)
    return str(Path.home() / '.openclaw')
```

インストール先のディレクトリ構成が違っても動くよう、設定パスを自動検出する。環境変数で上書きもできる。こういう「よそのマシンで動く」ことを前提にした設計は、自社内ツールを作るときには意識しないことが多い。

### ノード管理APIの単純化

マルチノード版のAPIは「どのノードへのリクエストか」をパラメータで受け取り、内部でSSH越しにデータを取得していた。

Lite版は「本機のみ」なのでそれが消える：

```python
@app.route('/api/node/status')
def node_status():
    # ローカルファイルを読むだけ
    config_path = Path(OC_PATH) / 'config.json'
    ...
```

SSHセッション管理、タイムアウト処理、エラーハンドリングのレイヤーがまるごと消える。コードが減るだけでなく、障害ポイントも減る。

---

## フロントエンドの刷新：ブランドを外す

元のダッシュボードは社内ブランドカラー（緑系）を使っていた。miniPC同梱版は「汎用製品」として出荷するので、TechsFree色を抜く必要があった。

新テーマ：ダークUI + パープルアクセント。理由は単純で「コードエディタ的な雰囲気が技術製品っぽくていい」から。

CSS変数で全体の色を管理している：

```css
:root {
    --bg-primary: #0f0f1a;
    --bg-secondary: #1a1a2e;
    --bg-card: #16213e;
    --accent: #7c3aed;
    --accent-hover: #6d28d9;
    --text-primary: #e2e8f0;
    --text-secondary: #94a3b8;
}
```

これだけで全UIのカラースキームが変わる。テーマ対応を後付けで入れると地獄になるが、最初からCSS変数で設計すれば後で差し替えやすい。

---

## install.sh でゼロから動かす

出荷製品として重要なのは「インストール体験」だ。

```bash
#!/bin/bash
PORT=${1:-8099}

# Python依存インストール
pip3 install -r requirements.txt -q

# systemd ユニットファイル生成
cat > ~/.config/systemd/user/openclaw-dashboard.service << EOF
[Unit]
Description=OpenClaw Dashboard Lite
[Service]
ExecStart=$(which python3) $(pwd)/server.py
Environment=LITE_PORT=${PORT}
Restart=always
[Install]
WantedBy=default.target
EOF

# 有効化・起動
systemctl --user enable openclaw-dashboard.service
systemctl --user start openclaw-dashboard.service
echo "Dashboard Lite started on port ${PORT}"
```

`bash install.sh` で systemd サービスとして起動まで完結する。ポート番号を引数で変えられるので競合も避けやすい。

---

## 学んだこと

**制約は友達**。「これだけでいい」という制約があると、コードは自然に削られる。制約なしに「汎用ツール」を作ろうとすると膨らみ続ける。

**Single-Fileは意外と戦える**。SPAをバンドラなしで `index.html` 1ファイルに収めるのはメンテが大変に見えるが、2000行程度なら十分管理できる。`import` がない分、動作が追いやすいというメリットもある。

**「よそのマシンで動く」前提で書く**。ハードコードされたパス、環境固有の前提——社内ツールはこれだらけになりがちだ。出荷製品を作ると、そういう甘えが全部炙り出される。

---

## まとめ

- 元: バックエンド7000行 + フロントエンド3800行（マルチ機能・マルチノード）
- 今: バックエンド1000行 + フロントエンド2000行（タスク + ローカルノードのみ）
- スタック: Flask + バニラJS SPA（依存: flask のみ）
- インストール: `bash install.sh [port]` の1コマンド

削ること、それ自体がエンジニアリングだと改めて感じた。

---

**タグ案:** `#Web開発` `#Flask` `#SPA` `#リファクタリング` `#ツール開発` `#Python` `#フロントエンド` `#miniPC`

