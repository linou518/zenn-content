---
title: "Sequelize + TypeScript: `public` を使うと属性が消える罠"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "自動化"]
published: true
---

---
title: "Sequelize + TypeScript: `public` を使うと属性が消える罠"
emoji: "🐛"
type: "tech"
topics: ["TypeScript", "Sequelize", "ORM", "NodeJS"]
published: true
---

TypeScriptでSequelizeモデルを書くとき、何も考えずに `public` でフィールド宣言してないだろうか。23モデル全部壊れてた話をする。

## 何が起きたか

プロジェクトの結合テストで、顧客名・仕入先名が全部 `null` で返ってくる現象が発生した。DBにはちゃんとデータが入っている。`SELECT * FROM sales` すれば `customer_name = '株式会社テスト'` と表示される。なのにAPIレスポンスは `null`。

## 原因: class field が Sequelize の getter を潰す

問題のコードはこれ：

```typescript
// ❌ これがダメ
class Sale extends Model {
  public customer_name!: string;
  public total_amount!: number;
  // ...
}
```

TypeScriptの `public` キーワード付きクラスフィールドは、コンパイル後にコンストラクタ内で `this.customer_name = undefined` として初期化される。これがSequelizeがprototypeに設定したgetter/setterを**完全に上書き**する。

ES2022のクラスフィールド仕様では、`Object.defineProperty` ではなく `[[Define]]` セマンティクスでインスタンスプロパティを設定する。つまりprototypeチェーンのgetter/setterは無視され、インスタンスに直接 `undefined` が書き込まれる。

## 修正: `declare` を使う

```typescript
// ✅ これが正解
class Sale extends Model {
  declare customer_name: string;
  declare total_amount: number;
  // ...
}
```

`declare` はTypeScriptの型チェックのみに使われ、JavaScriptに一切出力されない。Sequelizeのgetter/setterが生きたまま残る。

注意点として、`declare` と `!`（non-null assertion）は併用できない（TS1255エラー）。`declare` 自体が「このプロパティは別の場所で初期化される」という宣言なので、`!` は不要。

## 影響範囲と修正作業

全23モデルで `public xxx!: type` → `declare xxx: type` に一括置換した。対象：

- Sale, SaleDetail, Arrival, ArrivalDetail, Inventory
- Customer, Supplier, Warehouse, Unit, Department
- Purchase, PurchaseDetail, Payment, PaymentDetail
- Shipment, ShipmentDetail, Quotation
- AccountsPayable, AccountsReceivable, CompanyAccount, Company
- DeliveryDestination, PaymentDestination, ProcurementAlert

`npx tsc` でビルド → エラー0件 → デプロイ → 全フィールド正常に返却されることを確認。

## なぜ今まで気づかなかったか

単体テストでは `Model.create()` の戻り値をそのまま使うことが多い。`create()` はインスタンスを新規生成してsetterで値を入れるので、getter shadowingの影響を受けにくい。

問題が顕在化するのは `Model.findAll()` や `Model.findOne()` でDBから読み出すとき。Sequelizeが内部でgetter経由で値を取得しようとして、shadowされた `undefined` を返す。

結合テストで初めて「DBにデータ入れる→別のAPIで読み出す」フローを通したから発覚した。

## 教訓

1. **Sequelize + TypeScriptでは `declare` 一択**。公式ドキュメントにも書いてあるが、古いチュートリアルや生成コードは `public` を使っていることが多い
2. **class field shadowing は静かに壊れる**。エラーもワーニングも出ない。値が `null`/`undefined` になるだけ
3. **結合テストは正義**。単体テストだけでは見つからないバグが実際にある

<!-- 本文已脱敏処理: プロジェクト名を一般化 -->

