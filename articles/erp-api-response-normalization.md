---
title: "商品検索が白屏になった話 — APIレスポンス構造の不一致をどう直したか"
emoji: "🔧"
type: "tech"
topics: ["TypeScript", "デバッグ", "API設計", "フロントエンド"]
published: true
---

> 実体験: 2026-03-21、ERP管理画面の修正作業から

## 何が起きたか

ERP管理画面の「商品検索」ボタンを押すと、画面が真っ白になってクラッシュする。コンソールには `Cannot read properties of undefined (reading 'map')` のエラー。

状況整理：
- ビルド自体は通っている
- 他のページは動く
- 商品検索のダイアログを開いた瞬間に落ちる

## 原因を掘る

まず `SearchDialog` コンポーネントを確認。商品一覧を表示するため、APIから返ってきたデータに `.map()` をかけている。その前提は「`response.data` が配列である」こと。

次にAPIレスポンスを実際に叩いて確認：

```json
{
  "code": 1,
  "data": {
    "list": ["..."],
    "total": 42
  }
}
```

ここで気づいた。このERPのAPIクライアント（`request()`）は `code === 1` を「成功」として検知し、`data` の中身をアンラップして返す設計になっている。つまり呼び出し元が受け取るのは：

```json
{
  "list": ["..."],
  "total": 42
}
```

`SearchDialog` はこれを `response.data` として期待しているが、受け取ったオブジェクトに `data` プロパティは存在しない。だから `undefined.map()` でクラッシュ。

## どう直したか

APIクライアントのアンラップ挙動をコンポーネント側で変えるのはリスクが高い（他の箇所への影響が読めない）。代わりに `erpCompat.ts` という互換レイヤーに `getProducts()` 関数を追加し、レスポンスを正規化することにした：

```typescript
export async function getProducts(params: ProductSearchParams) {
  const result = await request('/erp/products', params);
  
  // APIがunwrap済みの { list, total } を返す場合に正規化
  if (result && Array.isArray(result.list)) {
    return {
      data: result.list,
      total: result.total ?? 0,
    };
  }
  
  // 既に { data: [...] } 形式の場合はそのまま
  return result;
}
```

`SearchDialog` はこの `getProducts()` を呼ぶだけ。APIの内部仕様が変わっても、正規化はここ一箇所で対応できる。

## 教訓

**APIレスポンスのアンラップは「透明じゃない」**。`request()` が勝手に `data` を剥いていることに気づかないと、呼び出し元での型が「ずれた」ままになる。TypeScriptを使っていても、型が `any` になっている箇所ではこの種のバグは静かに潜む。

対策としては：
1. **APIクライアントのアンラップ挙動をドキュメント化する**（または型で表現する）
2. **レスポンスの実値をコンソールで確認する習慣**（型推論を信じすぎない）
3. **互換レイヤーで正規化**（コンポーネントに生のAPIを直接触らせない）

今回はデプロイ後に発覚したバグだったが、互換レイヤーを挟む設計にしておいたおかげで修正範囲が最小で済んだ。「薄いラッパー関数を一枚挟む」の価値を改めて実感した。
