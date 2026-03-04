---
title: "LINE Bot と自社 ERP を「電話番号マッチング」でつなぐ権限バインドシステムを作った"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "自動化"]
published: true
---

---
title: "LINE Bot と自社 ERP を「電話番号マッチング」でつなぐ権限バインドシステムを作った"
emoji: "📱"
type: "tech"
topics: ["LINE", "ERP", "TypeScript", "Node.js", "Sequelize"]
published: true
category: "infra"
tags: ["LINE Bot", "ERP", "TypeScript", "Node.js", "Sequelize", "業務システム", "権限管理"]
---

# LINE Bot と自社 ERP を「電話番号マッチング」でつなぐ権限バインドシステムを作った

LINEで在庫を確認したい。でも誰でも見られるのは困る。

そんなシンプルな要件から始まったのが、今回実装した「LINE × ERP 権限バインドシステム」だ。

## 背景

TechsFree では食品・果物の卸売業向けに自社 ERP を開発・運用している。担当スタッフが LINEで在庫確認・売上確認できたら現場が楽になる──そう考えて、LINE Bot に ERP のバックエンドを連携させる構成にした。

ただし問題がある。LINE の userId（`Uxxxxxxxx...`）と ERP の内部ユーザー（メール＋パスワード）は別物だ。「この LINE ユーザーが誰か」を ERP 側で特定しないと、権限チェックもできない。

## 解決策：電話番号マッチング

ERP のユーザーテーブルにはすでに `phone` カラムがある。LINEでボットに話しかけてきたとき、「あなたの電話番号を教えて」と聞いて、その番号で ERP ユーザーを検索・紐付ける。

```sql
-- users テーブルに LINE 関連カラムを追加
ALTER TABLE users ADD COLUMN line_user_id VARCHAR(64) UNIQUE NULL;
ALTER TABLE users ADD COLUMN line_display_name VARCHAR(255) NULL;
```

Sequelize を使っているので、`User.ts` にフィールドを追加すれば起動時に自動 sync される。

## API 設計

3 本のエンドポイントを追加した。

```
GET  /api/line/user/:line_user_id   # LINE IDで身元照会
POST /api/line/bind                  # 電話番号でERPアカウントにバインド
POST /api/line/unbind                # バインド解除
```

`/api/line/user/:id` のレスポンスはこんな感じ：

```json
// 未バインド
{ "bound": false }

// バインド済み
{
  "bound": true,
  "user": {
    "id": 1,
    "name": "田中 太郎",
    "role": "admin",
    "permissions": [
      "sales", "purchases", "inventory",
      "products", "customers", "suppliers",
      "payments", "reports", "staff"
    ]
  }
}
```

## 権限グループ

ロールは 2 種類に絞った。

| role | 使える機能 |
|------|-----------|
| admin | 売上/仕入/在庫/商品/顧客/仕入先/入金/レポート/スタッフ管理 |
| user（staff） | 在庫/商品/仕入/売上/出荷 |

「customer ロールも作る？」という話もあったが、LINE Bot 経由で一般顧客が ERP の在庫を直接見る必要はない（Shop フロントエンドが別にある）ので今回は除外した。シンプルに保つのが大事。

## Agent との連携フロー

LINE Bot 本体は別サーバー上で動く Agent が担当している。Agent は LINEメッセージを受け取ったとき、まず `/api/line/user/:line_user_id` を叩いて `bound` を確認する。

- `bound: false` → 「電話番号を教えてください」と案内してバインドフローへ
- `bound: true` → `permissions` に応じて使える機能を提示

ERP API への実際の呼び出しは Agent が直接行う（内部ネットワーク内なので認証トークン不要）。

## 実装で詰まったところ

**TypeScript の型定義**が地味に面倒だった。`User` モデルに新しいカラムを追加するとき、`sequelize-typescript` のデコレーターと Sequelize の `sync` の両方を正しく書かないとランタイムエラーになる。コンパイルは通っても実行時に「カラムが存在しない」と言われるやつ。

```typescript
@Column({ type: DataType.STRING(64), unique: true, allowNull: true })
line_user_id!: string | null;
```

`allowNull: true` を忘れると既存レコードの sync で死ぬ。

**もう一つ**は、LINE の channelAccessToken の大文字小文字問題。デバッグに時間がかかったが、token 自体が無効化されていたのが原因だった。LINE Developers Console で再発行すれば即解決。**ログの文字を信じすぎると詰まる**という教訓だ。

## まとめ

「外部 Bot に自社 ERP の権限チェックを統合する」という要件は、思ったよりシンプルな構造で実現できた。

ポイントは：

1. **既存の属性（電話番号）でマッチング**する。OAuthを入れずに済む
2. **バインド情報は ERP 側で管理**する。Bot 側に状態を持たせない
3. **権限をフラットなリストで返す**。Bot 側のロジックがシンプルになる

「LINE と ERP を連携させたい」という話は業務システムではよくある要件だと思う。認証基盤を作り直さずに、既存ユーザーテーブルの電話番号カラムを使うというアプローチは参考になるかもしれない。

