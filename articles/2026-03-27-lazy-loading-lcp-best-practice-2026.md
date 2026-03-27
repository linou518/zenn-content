---
title: "その画像最適化、実は逆効果かも：Lazy Loading と LCP の正しい使い方（2026年版）"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "自動化"]
published: true
---

---
title: "その画像最適化、実は逆効果かも：Lazy Loading と LCP の正しい使い方（2026年版）"
date: "2026-03-27"
category: "web-tech"
tags: ["パフォーマンス最適化", "LCP", "LazyLoading", "CoreWebVitals", "フロントエンド", "SEO", "fetchpriority"]
lang: "ja"
author: "TechsFree"
draft: false
summary: "16%のサイトが同じミスを犯しています——最重要な画像に loading='lazy' を設定してしまっているのです。Lazy Loading と LCP の境界線、そして 2026 年に必須の fetchpriority 属性を徹底解説します。"
readingTime: 8
---

最適化しているつもりが、実はサイトを遅くしている——そんなケースが後を絶ちません。

2025年 Web Almanac のデータによると、**モバイルページの約 16%** が LCP 画像（ファーストビューの最大コンテンツ）に誤って遅延読み込みを適用しています。これは些細なミスではありません。Core Web Vitals が赤ゾーンに落ち、Google の検索順位に直接影響し、コンテンツが表示される前にユーザーが離脱する原因となります。

問題の根本は、本来は優れた技術である「遅延読み込み」を間違った場所で使ってしまっていることです。

---

## Lazy Loading 自体は悪くない——使い方が問題

Lazy Loading の考え方は正しいです。ユーザーがすぐに見ない領域のコンテンツの読み込みを遅らせ、初期読み込みの負荷を軽減する。正しく使えば、ファーストビューの読み込み時間を 30〜50% 削減できます。

しかし、多くの開発者が見落としている重要なメカニズムがあります：ブラウザの**プリロードスキャナー（Preload Scanner）**です。

ブラウザは HTML を解析する際、プリロードスキャナーがドキュメントを並行してスキャンし、リソースを早期に検出してダウンロードキューに追加します。これが現代ブラウザのパフォーマンスの重要な源泉です。ところが、画像に `loading="lazy"` を設定すると、プリロードスキャナーはその画像を**完全にスキップ**し、最低優先度でキュー末尾に回します。

そのスキップされた画像がページ最大の要素（バナー、ヒーロー画像、メイン商品画像）だったとしたら？ LCP の時間が一気に跳ね上がります。

Chrome DevTools で見るとはっきりわかります。7枚の画像があるページで、遅延読み込みが設定されたヒーロー画像だけが**最後に**ダウンロードを開始する——見た目上は最重要な画像なのに。

---

## 2種類の遅延読み込み、どちらも落とし穴

lazysizes.js などの JS 遅延読み込みライブラリを使っているプロジェクトも多いですが、実はネイティブの `loading="lazy"` より悪い場合があります。

| 方式 | 遅延の原因 |
|------|-----------|
| `loading="lazy"` | プリロードスキャナーにスキップされ、キュー末尾へ |
| JS 遅延読み込み | JS のダウンロード＋実行を待ってから画像読み込みが始まる |

JS 遅延読み込みは二重の遅延——スクリプト待ち、そして画像待ち。

---

## 2026年の正しいアプローチ

核心原則は一つ：**折りたたみ線より下（below the fold）のコンテンツにのみ遅延読み込みを使う**。

```
ビューポート内（Above the fold）
├── LCP 要素（ヒーロー画像、メインバナー）
│   ├── loading="lazy" を付けない
│   ├── fetchpriority="high" を付ける
│   └── <link rel="preload"> で早期検出
│
└── その他のビューポート内画像
    └── デフォルトのまま（ブラウザが自動で eager 読み込み）

ビューポート外（Below the fold）
└── すべての画像 → loading="lazy" ✅ これが正しい使い方
```

### fetchpriority：2026年で最も過小評価されている属性

`fetchpriority="high"` は `loading="lazy"` の逆です。ブラウザに「この画像が最重要なので優先してダウンロードしろ」と明示的に伝えます。Chrome/Edge のサポート率は 90% 超ですが、使用率は本来あるべき水準をはるかに下回っています。

```html
<!-- LCP ヒーロー画像：最高優先度 -->
<img src="hero.webp"
     fetchpriority="high"
     width="1200" height="600"
     alt="hero">

<!-- 折りたたみ線より下の画像：遅延読み込み -->
<img src="product.webp"
     loading="lazy"
     width="400" height="300"
     alt="product">

<!-- 最適な書き方（プリロード＋高優先度） -->
<link rel="preload" as="image" href="hero.webp" fetchpriority="high">
<img src="hero.webp" fetchpriority="high" width="1200" height="600" alt="hero">
```

実測データ：`fetchpriority="high"` を正しく使うと、LCP が **100〜200ms** 改善されます。

---

## フレームワーク別の注意点

### WordPress
WordPress 6.3+ は最初の画像に自動で `fetchpriority="high"` を設定しますが、サードパーティの最適化プラグイン（WP Rocket、Autoptimize など）がこの動作を上書きする場合があります。

**確認方法**：Lighthouse を実行 → 「LCP image was lazily loaded」警告を確認。

### Next.js
Next.js の `<Image>` コンポーネントはデフォルトですべての画像を遅延読み込みします。ヒーロー画像には必ず `priority` を付けてください：

```jsx
// ❌ 誤り：ヒーロー画像が遅延読み込みされる
<Image src="/hero.jpg" alt="hero" />

// ✅ 正解：priority = fetchpriority="high" + 遅延読み込みなし
<Image src="/hero.jpg" priority alt="hero" />
```

---

## 目標値

- LCP **< 2.5 秒** = Good
- LCP **2.5〜4 秒** = 要改善
- LCP **> 4 秒** = Poor（検索順位に直接影響）

LCP 画像への遅延読み込み設定が引き起こす遅延：通常 **50〜300ms 以上**、深刻な場合は 1 秒超。

---

## まとめ：画像パフォーマンスの階層的な考え方

遅延読み込みは良いツールです。ただし、場所を間違えれば悪いツールになります。

重要なのは階層的な思考を持つこと：**ビューポート内は速く、ビューポート外だけ遅延させる**。`fetchpriority="high"` で何が最重要かをブラウザに伝え、`loading="lazy"` で何が待てるかを伝える。

プラグインもライブラリも不要です。数行の HTML だけで、LCP スコアを赤ゾーンから緑ゾーンへ変えることができます。

---

*データソース：2025 Web Almanac, Chrome DevTools 実測値, corewebvitals.io*

