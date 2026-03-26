---
title: "Core Web Vitals 2025完全ガイド：INP新指標と厳しくなった閾値に今すぐ対応する"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "自動化"]
published: true
---

---
title: "Core Web Vitals 2025完全ガイド：INP新指標と厳しくなった閾値に今すぐ対応する"
date: "2026-03-26"
category: "web-tech"
tags: ["CoreWebVitals", "INP", "LCP", "CLS", "Webパフォーマンス", "SEO", "フロントエンド", "パフォーマンス最適化"]
lang: "ja"
author: "TechsFree"
draft: false
summary: "GoogleのCore Web VitalsがINP導入で大幅改定。LCPが2.5s→2.0s、CLSが0.1→0.08と閾値も厳格化。今すぐチェックすべきポイントと具体的な最適化コードを解説。"
readingTime: 10
---

昨日まで「合格」だったサイトが、今日からGoogleの「Good」ゾーンを外れているかもしれません。

2025年、GoogleはCore Web Vitalsを史上最大規模で改定しました。新指標 **INP（Interaction to Next Paint）** の導入に加え、LCP・CLS・FCPの閾値を全面的に厳格化。これは単なる技術仕様の変更ではなく、Googleの「ユーザー体験」に対する根本的な考え方の転換を意味します。**「ページ読み込みが速い」だけではもはや不十分、「クリック後も速い」ことが求められる時代へ。**

---

## 2025年の変更がこれまでより重要な理由

Core Web VitalsはGoogleのランキング要因として2021年から採用されていますが、2025年の変更には三つの重要なポイントがあります。

1. **FIDが廃止、INPが後継指標に**：FIDは「最初のインタラクション」の遅延のみを計測していました。INPは**セッション全体を通じて最も遅かったインタラクション**を記録します。メニューのクリック、フォーム送信、ボタンのタップ——すべてが評価対象です。
2. **FCPが正式指標に追加**：First Contentful Paint（< 1.5s）はこれまで参考値でしたが、2025年から正式な評価指標になりました。
3. **全閾値の厳格化**：猶予期間は終わりました。2025年は結果が問われる年です。

| 指標 | 旧「Good」閾値 | 2025年新「Good」閾値 | 変化 |
|------|--------------|-------------------|------|
| LCP（最大コンテンツ描画） | < 2.5s | **< 2.0s** | 20%厳格化 |
| FID → **INP**（インタラクション応答） | FID < 100ms | **INP < 200ms** | 新指標 |
| CLS（累積レイアウトシフト） | < 0.1 | **< 0.08** | 25%厳格化 |
| FCP（最初のコンテンツ描画） | 非公式 | **< 1.5s** | 新規追加 |

---

## INPとは何か、なぜFIDより最適化が難しいか

FIDは「最初のクリックを速くすれば高スコアが取れる」という抜け道がありました。INPにはその逃げ道がありません。

INPは**セッション内の全インタラクション遅延の98パーセンタイル値を最終スコアとします**。つまり、最も遅い瞬間がスコアを決めます。

| 評価 | INP値 | ユーザー体験 |
|------|-------|------------|
| 🟢 Good | < 200ms | スムーズ、ユーザーは遅延を感じない |
| 🟡 要改善 | 200–500ms | 遅延を感じる、体験が損なわれる |
| 🔴 Poor | > 500ms | 明確な遅延、ユーザー離脱 |

**INPが悪い根本原因のほぼすべては、JavaScriptがメインスレッドをブロックしていること。** ユーザーがクリックした瞬間、ブラウザは次のフレームを描画しようとしますが、メインスレッドがJSの実行で占有されているため、インタラクションが止まってしまいます。

---

## 4つの指標の実践的最適化

### 1. INP最適化：メインスレッドに「息継ぎ」を

**戦術1：タスク分割（Task Chunking）**

長いタスク（50ms以上のJS実行）はINPの最大の敵。`setTimeout(0)` で細かく分割：

```javascript
// ✅ 推奨：大量データ処理を分割してメインスレッドをブロックしない
function processDataInChunks(data, chunkSize = 100) {
  let index = 0;
  function processChunk() {
    const chunk = data.slice(index, index + chunkSize);
    chunk.forEach(item => processItem(item));
    index += chunkSize;
    if (index < data.length) {
      setTimeout(processChunk, 0); // ブラウザに制御を返す
    }
  }
  processChunk();
}
```

Chrome 122+では `scheduler.yield()` というより精密な方法も使えます：

```javascript
async function processWithYield(data) {
  for (const item of data) {
    processItem(item);
    await scheduler.yield(); // 明示的に制御を譲る
  }
}
```

**戦術2：Web Workerで計算集約タスクをオフロード**

```javascript
// main.js
const worker = new Worker('dataWorker.js');
worker.postMessage({ data: largeDataset });
worker.onmessage = (e) => updateUI(e.data.result);

// dataWorker.js
self.onmessage = (e) => {
  const result = heavyComputation(e.data.data);
  self.postMessage({ result });
};
```

**戦術3：イベントハンドラのデバウンス**

```javascript
// ❌ キーストロークごとに重い検索処理
input.addEventListener('input', () => fetchSearchResults(input.value));

// ✅ 300ms デバウンス、実行回数を90%削減
const debouncedSearch = debounce((val) => fetchSearchResults(val), 300);
input.addEventListener('input', () => debouncedSearch(input.value));
```

---

### 2. LCP最適化：2.0sのラインを守る

LCPはファーストビューで最も大きなコンテンツ（通常はヒーロー画像や大きな見出し）が表示されるまでの時間。

**アクションチェックリスト：**
- ✅ 画像をWebP / AVIF形式に変換（JPEGより30–50%軽量）
- ✅ ヒーロー画像に `fetchpriority="high"` を追加
- ✅ `<link rel="preload" as="image" href="/hero.webp">` でプリロード
- ✅ CDNで静的リソースを配信、ラウンドトリップを最小化
- ✅ TTFB < 600ms（サーバーレスポンス前にLCPは始まらない）

```html
<!-- ✅ LCP画像のベストプラクティス -->
<link rel="preload" as="image" href="/hero.webp" fetchpriority="high">
<img 
  src="/hero.webp" 
  alt="Hero"
  width="1200" 
  height="600"
  fetchpriority="high"
  loading="eager"
>
```

---

### 3. CLS最適化：0.08はレイアウトシフト「ゼロ許容」の意味

CLSはページロード中に要素が予期せず移動する量を測定します——広告の突然表示、フォントのスワップ、サイズ未指定の画像が主な原因。

| よくある原因 | 対策 |
|------------|------|
| サイズ未指定の画像 | `width` と `height` 属性を必ず指定 |
| 広告・埋め込みコンテンツ | `min-height: 250px` でコンテナを予約 |
| フォント読み込みのフラッシュ | `font-display: swap` + フォントファイルのプリロード |

```css
/* ✅ 広告コンテナのスペース予約でCLSを防止 */
.ad-container {
  min-height: 250px;
  background: #f0f0f0;
  display: flex;
  align-items: center;
  justify-content: center;
}
```

---

### 4. FCP最適化：1.5秒以内に何かを描画する

FCPはユーザーが最初にコンテンツを見る瞬間。**クリティカルCSSのインライン化** が最も直接的な方法：

```html
<head>
  <!-- ✅ クリティカルCSSはインライン化——外部ファイルの待機不要 -->
  <style>
    body { font-family: sans-serif; margin: 0; }
    .hero { background: #1a1a2e; color: white; padding: 60px; }
  </style>
  
  <!-- 非クリティカルCSSは非同期読み込み -->
  <link rel="preload" href="/styles/main.css" as="style" 
        onload="this.rel='stylesheet'">
</head>
```

---

## 10分でサイトを診断する方法

1. **[PageSpeed Insights](https://pagespeed.web.dev)** — 最速、無料、実ユーザーデータ
2. **Chrome DevTools → Performanceパネル** — インタラクションを録画してロングタスクを特定
3. **Search Console → Core Web Vitalsレポート** — Googleがランキングに実際に使用するフィールドデータ

> 💡 ラボデータ（Lighthouse）とフィールドデータ（CrUX）は大きく異なる場合があります。Googleのランキングに影響するのはフィールドデータです。

---

## ビジネスへの影響：これは技術問題だけではない

計測された実際のデータです：

- INP > 500ms（Poor）→ ユーザーエンゲージメント **23%低下**
- LCP > 2.5s → コンバージョン率 **7%低下**
- CLS > 0.25 → 直帰率 **15%上昇**

月間10万UVのECサイトなら、LCPをPoorからGoodに改善するだけで、コンバージョン漏斗に入るユーザーが月7,000人増える計算です。

---

## 2025年アクションチェックリスト

**今週やること：**
- [ ] PageSpeed InsightsでメインページのCore Web Vitalsを計測してベースラインを確立
- [ ] `width`/`height` 属性なしの画像を確認（CLS最多発生源）
- [ ] ヒーロー画像に `fetchpriority="high"` を追加

**今月やること：**
- [ ] Chrome DevToolsでINPのボトルネックとなるロングタスクをプロファイル
- [ ] 重いJS計算をWeb Workerに移行
- [ ] クリティカルCSSをインライン化、残りは非同期読み込み

**継続的にやること：**
- [ ] Search ConsoleでCore Web Vitalsを毎月確認
- [ ] 新機能リリース前にLighthouseのベースラインテストを実施

パフォーマンスは一度の最適化で終わりではなく、継続的なエンジニアリングの習慣です。しかし2025年の閾値厳格化は明確なシグナルを送っています：**今始めることが、後でペナルティを受けるより遥かに賢い選択です。**

---

*データソース：Google Web.dev公式ドキュメント · Nandann.com INP調査レポート · SpyceMedia 2025 Core Web Vitals分析*

