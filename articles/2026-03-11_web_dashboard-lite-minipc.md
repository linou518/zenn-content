---
title: "miniPC同梱サービスのために、DashboardをLite化した話"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "自動化"]
published: true
---

# miniPC同梱サービスのために、DashboardをLite化した話

既存の管理ダッシュボードから、miniPC販売用の「軽量同梱版」を作った。コードの削り方と、Flaskシングルファイル構成のノウハウを整理する。

---

## 背景

TechsFreeでは社内運用用のダッシュボード（以下、フルダッシュ）を運用している。タスク管理、ノード管理、家族行事カレンダー、スケジューラー……いろいろ詰め込んだ結果、Flaskバックエンドは7000行超、フロントは3800行を越えるSPAになっていた。

そこに一つのリクエストが来た。

> 「miniPCを販売するとき、付属サービスとして同梱できるダッシュボードが欲しい。でも社内専用の機能はいらない。シンプルで、汎用的で、かっこよく。」

つまり、フルダッシュから「TechsFree色」と「社内専用機能」を全部剥がして、どこのユーザーでも使える形にする、ということだ。

---

## 何を残して、何を切るか

フルダッシュの機能一覧を並べて、「誰でも使う？」で仕分けた。

| 機能 | 残す？ | 理由 |
|------|--------|------|
| プロジェクトタスク | ✅ | 汎用的 |
| ノード管理（本機のみ） | ✅ | 同梱miniPC向け |
| 家族カレンダー | ❌ | 個人/社内専用 |
| スケジューラー（今日の作業） | ❌ | 社内専用 |
| Claude使用量表示 | ❌ | サービス依存 |
| タイムライン | ❌ | 社内専用 |
| マルチノード管理 | ❌ | 同梱版は本機のみでOK |

結果として残ったのは2機能だけ。潔い。

---

## 「流用」より「書き直し」を選んだ

フルダッシュからコードをコピーして削る方法と、ゼロから書く方法、どちらが早いか。

今回はゼロから書くことにした。理由はシンプルで、フルダッシュのコードは「削らない前提」で書かれているため、条件分岐や参照が複雑に絡み合っている。削ろうとすると、切った後に何かが壊れているかどうかを確認するコストが大きい。

サーバー側：Flaskバックエンドはフルダッシュの7000行から500行未満に。それでも全機能を実装できた。肥大化の大半は「本当は分離すべきだったロジック」だったことがわかる。

フロント側：37KBのシングルHTMLファイル。依存はゼロ、CDNも使わない。同梱版のコンセプトとして、オフライン環境でも動くことを想定した。

---

## install.shで「一コマンドインストール」にした

```bash
bash install.sh 8099
```

これだけで次のことが走る：

1. Pythonと依存チェック
2. systemdサービスファイルの生成
3. サービス有効化・起動
4. 起動確認

miniPCを受け取ったユーザーがターミナルを開いてこれを叩けば、次回起動から自動で立ち上がる。サービス名は`dashboard-lite`、ログは`journalctl -u dashboard-lite`で見れる。

systemdのサービスファイルをスクリプト内で動的生成する部分は、パスをハードコードしないために`realpath`で解決している。

```bash
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
SERVICE_FILE="/etc/systemd/system/dashboard-lite.service"

cat > "$SERVICE_FILE" << EOF
[Unit]
Description=OpenClaw Dashboard Lite
After=network.target

[Service]
ExecStart=/usr/bin/python3 $SCRIPT_DIR/server.py
WorkingDirectory=$SCRIPT_DIR
Restart=always
Environment=LITE_PORT=$PORT

[Install]
WantedBy=multi-user.target
EOF
```

---

## OC_PATH自動検出

ノード管理機能はOpenClawのローカル設定（`openclaw.json`）を読む。ところが設定ファイルの場所はインストール環境によって異なる。

```python
def detect_oc_path():
    candidates = [
        os.environ.get("OC_PATH"),
        os.path.expanduser("~/.openclaw"),
        "/home/openclaw/.openclaw",
        "/opt/openclaw",
    ]
    for path in candidates:
        if path and os.path.exists(os.path.join(path, "openclaw.json")):
            return path
    return None
```

環境変数で明示指定できるようにしつつ、未指定なら候補を順番に試す。「動く確率を上げる」ことを優先した設計で、インストールスクリプトに事前設定ロジックを持たせなくて済む。

---

## デザイン：「社内ツール感」を消す

フルダッシュはTechsFreeのブランドカラーで作られている。Lite版は、どのブランドにも見えない「汎用プロダクト感」が必要だった。

- カラーパレット: 深めのダークグレー + 紫のアクセント
- フォント: システムフォントスタック（追加ロードなし）
- ロゴ: テキストのみ（SVGや画像ファイルの依存なし）

依存ゼロの方針が、見た目の設計にも効いている。

---

## 実際に動かして確認したこと

infraノード（192.168.3.33）でテスト起動して確認した項目：

- `/api/simple-tasks` → プロジェクト一覧が返る
- `/api/node/status` → `active`, OpenClaw v2026.2.26 を返す
- `/api/node/bots` → ローカルのBot一覧が返る

フロント側もブラウザで開いて、タスク追加→完了→削除の一連を確認。ノードページでGatewayのログ表示も動作済み。

---

## まとめ

「削る」作業は地味だが、やると構造的な負債が可視化される。今回の作業で、フルダッシュの肥大化が「機能追加の積み重ね」ではなく、「最初からLite相当で済む機能を過剰に実装した」せいだとわかった。

次にフルダッシュ側を触るときは、Lite版を参考にしてシンプルな実装パターンを逆輸入できるかもしれない。

---

**タグ案**: `flask`, `spa`, `minipc`, `dashboard`, `python`, `frontend`, `リファクタリング`, `openclaw`

