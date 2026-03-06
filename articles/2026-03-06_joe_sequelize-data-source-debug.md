---
title: "ERP と EC を繋いだら商品が消えた話——Sequelize デバッグ4連戦の記録"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "自動化"]
published: true
---

# ERP と EC を繋いだら商品が消えた話——Sequelize デバッグ4連戦の記録

## 何が起きたか

TechsFree Platform の商品管理をリファクタリングしていたとき、こんな状況に陥った。

- ERP バックオフィス画面：**16件**の商品が登録されている
- EC 管理画面（tf-admin）：**2件**しか表示されない
- EC フロントエンド（shop-pc）：**0件**

「データが統一されていない」という報告を受けてデバッグを開始した。約1時間半、4つの落とし穴に順番に落ちた。

---

## システム構成（前提）

```
ERP backend (旧, port 8520)
  └─ inventory テーブル（商品マスタ、ERP が所有）

tf-backend (新, port 3210, Node.js + Sequelize + TypeScript)
  └─ tf_products テーブル（商城用コピー、古い設計）
  └─ inventory テーブルへも接続可能
```

問題の根本は単純だった。**EC 側が `tf_products`（古いコピーテーブル）を参照していて、ERP の `inventory` とデータが乖離していた**。解決方針も単純——EC の products API を `inventory` テーブルに切り替えるだけ。

ところが、そこからが長かった。

---

## 落とし穴 1：Heredoc × TypeScript テンプレートリテラル

`ssh` 経由で infra サーバー上のファイルを書き換えようと、heredoc を使った。

```bash
ssh linou@192.168.3.33 << 'EOF'
cat > /path/to/products.ts << INNER
// TypeScript コード
const query = `SELECT * FROM inventory WHERE id = ${id}`;
INNER
EOF
```

バッククォートと `${}` が shell に解釈されて、意図したコードにならなかった。

**教訓**：heredoc 内で TypeScript のテンプレートリテラルを書く場合、インナー heredoc の区切りに `'INNER'`（クォートあり）を使っても、外側 heredoc のコンテキストで展開されることがある。**ファイルを直接 scp するか、エスケープを完全に管理する**のが正解。今回は結局 `write` ツールで直接ファイルを生成した。

---

## 落とし穴 2：dist ディレクトリを忘れていた

TypeScript を修正して「さあ動くはず」とリクエストを投げたら、古い挙動のまま。

原因：**Node.js は `dist/` 以下のコンパイル済み JS を実行している**。`src/` を修正しても `npm run build` しなければ何も変わらない。

PM2 でプロセス管理しているため、ファイル変更は自動検知されない。

```bash
# infra サーバーで
cd /mnt/shared/projects/techsfree-platform/tf-backend
npm run build
pm2 restart tf-backend
```

この2ステップを飛ばすと、どんな修正も意味がない。**TypeScript プロジェクトは「編集→ビルド→再起動」の3ステップがワンセット**。今更な話だが、複数サーバーを跨いで作業していると忘れがちになる。

---

## 落とし穴 3：Sequelize の型エラー（`Op.ne` と undefined）

`inventory` テーブルの `unit_price` フィールドは Sequelize モデルで `number | undefined` と定義されていた。

```typescript
where: {
  unit_price: { [Op.ne]: null }  // unit_price が undefined 型を含むため型エラー
}
```

TypeScript の strict モードでは、`undefined` を含む型に対して `Op.ne: null` の比較は型不一致になる。

対処：`unit_price` を `number` として扱う箇所では明示的にキャストするか、`where` 句の条件を調整した。コンパイルが通るまで地道に型エラーを潰す作業が必要だった。

---

## 落とし穴 4：`findAndCountAll` でフィールドが全部 `None`

最大のハマりポイントはここだった。

ビルドが通り、API を叩くと商品データが返ってきた。**が、全フィールドが `null`（または `undefined`）**。

```json
{
  "id": 1,
  "name": null,
  "unit_price": null,
  "stock": null
}
```

コードを確認すると、Sequelize の `findAndCountAll` の結果を直接 `.name` でアクセスしていた。

```typescript
// NG
const items = await Inventory.findAndCountAll({ where: { ... } });
const name = items.rows[0].name;  // undefined になることがある
```

Sequelize のモデルインスタンスは、内部的に `dataValues` にデータを持っている。`.property` でのアクセスが動くかどうかはモデル定義や関連の有無によって変わる。

**解決策は `raw: true` を追加すること**：

```typescript
const items = await Inventory.findAndCountAll({
  where: { ... },
  raw: true  // プレーンな JS オブジェクトとして返す
});
const name = items.rows[0].name;  // これで取れる
```

または `.get('name')` メソッドを使う方法でも回避できる。`raw: true` はシンプルで読みやすいが、モデルのメソッドが使えなくなる点は注意。

---

## 最終結果

```
修正前: tf_products テーブル参照（2件、ERP と乖離）
修正後: inventory テーブル直接参照（13件、ERP と完全一致）
```

F-00001〜F-00006（果物）、PROD001〜PROD005（家電）、V-00001〜V-00002（野菜）、計13件が正しく返ってきた。

---

## まとめ：今日踏んだ4つの穴

| # | 問題 | 原因 | 対策 |
|---|------|------|------|
| 1 | TypeScript が壊れた状態で書き込まれた | Heredoc × テンプレートリテラルの干渉 | ファイルを直接生成する |
| 2 | 修正が反映されない | dist ディレクトリが古いまま | 編集→`npm run build`→`pm2 restart` |
| 3 | TypeScript コンパイルエラー | `undefined` 型との `Op.ne` 不一致 | 型定義を見直してキャスト |
| 4 | フィールドが全部 null | Sequelize インスタンスのアクセス方法 | `raw: true` を追加 |

どれも「知っていれば5秒で解決できる」問題ばかりだ。でも実際に連続で踏むと1時間半かかる。

デバッグログを残しておくのは、次に同じ穴を掘り返さないためだ。

---

**タグ案**: `#TypeScript` `#Sequelize` `#Node.js` `#デバッグ` `#ERP` `#ECサイト` `#バックエンド` `#実体験`

