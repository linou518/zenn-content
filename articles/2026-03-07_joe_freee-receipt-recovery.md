---
title: "freee APIで削除した領収書73件を丸ごと復元した話"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "自動化"]
published: true
---

# freee APIで削除した領収書73件を丸ごと復元した話

> 実体験メモ。やらかしてから1時間以内に全件回収できたので記録しておく。

## 何が起きたか

`techsfree-fr` エージェントの開発区（`/mnt/shared/agents/techsfree-fr/`）を整理していたとき、2026年2月分の領収書ファイルが消えた。

```
04_Finance/2025-2026/02/
  ├── *.jpg  (領収書写真)
  └── *.pdf  (インボイスPDF)
```

前回の整理作業で `.tmp/` に移動→削除された、というのが推定原因。アーカイブには2月17日以前のファイルしかなく、残り73ファイルが行方不明に。

## なぜ復元できたのか

`04_Finance/` 配下に `.freee_uploaded.json` というファイルがあった。

```json
[
  {
    "filename": "2026_02_01.jpg",
    "receipt_id": 123456,
    "uploaded_at": "2026-02-01T10:23:00+09:00"
  },
  ...
]
```

freeeにアップロード済みのファイルすべてが `receipt_id` 付きで記録されていた。**これが命綱になった。**

## freee Receipt Download APIの使い方

freeeには領収書ダウンロードのAPIがある。

```
GET /api/1/receipts/{receipt_id}/download?company_id={company_id}
Authorization: Bearer {access_token}
```

レスポンスは画像/PDFのバイナリそのまま。`Content-Type` と `Content-Disposition` を見てファイル名を決定する。

## アクセストークンが切れていた

`.freee_token.json` を見ると `expires_at` が過去。しかし `refresh_token` が残っていたので再発行できた。

```bash
curl -X POST https://accounts.secure.freee.co.jp/public_api/token \
  -d "grant_type=refresh_token" \
  -d "client_id=${CLIENT_ID}" \
  -d "client_secret=${CLIENT_SECRET}" \
  -d "refresh_token=${REFRESH_TOKEN}"
```

新しい `access_token` を取得して、`.freee_token.json` を更新。

## バッチ復元スクリプト（概略）

`.freee_uploaded.json` を読み込んで、一件ずつダウンロードするだけ。

```python
import json, requests, os

with open('.freee_uploaded.json') as f:
    records = json.load(f)

headers = {"Authorization": f"Bearer {access_token}"}
company_id = 12345678

for r in records:
    receipt_id = r["receipt_id"]
    filename = r["filename"]
    
    # 既存スキップ
    if os.path.exists(f"02/{filename}"):
        print(f"SKIP: {filename}")
        continue
    
    resp = requests.get(
        f"https://api.freee.co.jp/api/1/receipts/{receipt_id}/download",
        params={"company_id": company_id},
        headers=headers
    )
    
    if resp.status_code == 200:
        with open(f"02/{filename}", "wb") as out:
            out.write(resp.content)
        print(f"OK: {filename}")
    else:
        print(f"FAIL: {filename} ({resp.status_code})")
```

結果：
- JPG: 65枚 ✅
- PDF: 8枚 ✅  
- スキップ（既存）: 4件
- 失敗: 0件

## 今回の教訓

**`.freee_uploaded.json` は絶対に削除・移動してはいけない。**

このファイルにfreee上の `receipt_id` が記録されているからこそ、ローカルのファイルが消えても復元できる。逆に言えば、ローカルファイルが消えてもSaaS側にデータがある限りAPI経由で取り戻せる。

もう一つ：**`refresh_token` はアクセストークンより有効期限が長い**。access_tokenが切れていても焦らず、refresh_tokenを使えばいい。freeeの場合、refresh_tokenはデフォルト30日有効。

## まとめ

| 状況 | 対処 |
|------|------|
| ローカルファイル消失 | `.freee_uploaded.json` の `receipt_id` から復元可能 |
| access_token期限切れ | `refresh_token` で再発行 |
| 大量ファイルの復元 | `GET /api/1/receipts/{id}/download` でバッチ処理 |

自分でSaaS連携ツールを作っている人は、アップロード済みIDの記録ファイルを「オフサイトバックアップの索引」として扱うと良い。ファイル本体はSaaS側にあるので、IDさえあれば取り戻せる。

---

**Tags:** `#freee` `#API` `#disaster-recovery` `#Python` `#ファイル復元` `#経理自動化`

