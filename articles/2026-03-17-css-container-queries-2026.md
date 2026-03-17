---
title: "CSS Container Queries実践ガイド：コンポーネントレベルのレスポンシブがついに本番投入可能に"
emoji: "📦"
type: "tech"
topics: ["CSS", "フロントエンド", "レスポンシブ", "Web"]
published: true
---

Media Queriesは10年以上レスポンシブレイアウトを支えてきたが、根本的な欠点がある：**ビューポート基準**であって、**コンポーネント基準ではない**。同じカードコンポーネントをメインエリア（広い）とサイドバー（狭い）に置いたとき、Media Queriesでは冗余なクラス名やJSパッチに頼るしかなかった。

Container Queriesがこの問題を解決する。2026年、全主要ブラウザで本番投入可能になった。

## 3種類のContainer Queries

### 1. Container Size Queries（最も一般的）

```css
.card-wrapper {
  container-type: inline-size;
  container-name: card;
}

@container card (width > 480px) {
  .card { flex-direction: row; }
}

.card { flex-direction: column; }
```

重要：親要素に `container-type` を明示的に宣言しないと効かない。

### 2. Container Style Queries

```css
.theme-wrapper { --theme: dark; }

@container style(--theme: dark) {
  .btn { background: #1a1a1a; color: #fff; }
}
```

### 3. Container Scroll-State Queries（最新）

純CSSでスクロール状態に反応。もうJSのscrollイベントは不要：

```css
@container scroll-state(scrolled-y) {
  .header { box-shadow: 0 2px 8px rgba(0,0,0,0.2); }
}
```

## コンテナ単位：cqw / cqh

```css
.card-title {
  font-size: clamp(1rem, 4cqw, 1.5rem);
}
```

## パフォーマンス

ResizeObserver JS方式と比較して約35%高速。追加のlayout shiftなし。純CSS、ランタイムコストゼロ。

## 注意点

1. **明示的opt-in必須** — `container-type` なしでは何も起きない
2. **自分自身は問い合わせ不可** — 親コンテナのみ
3. **`inline-size` を優先** — `size` はサイズ計算に制限をかける
4. **Safari 16+** で完全対応

## ブラウザ対応

Chrome 105+、Firefox 110+、Safari 16+、Edge 105+。React/Vueアプリ採用率72%。

## 実践アドバイス

- **マクロレイアウト** → Media Queries、**マイクロコンポーネント** → Container Queries
- 新規コンポーネントは最初からContainer Queriesで設計
- ResizeObserverベースのレスポンシブロジックは移行を検討
