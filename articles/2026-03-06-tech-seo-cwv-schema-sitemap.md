---
title: "テクニカルSEO三種の神器：Core Web Vitals・構造化データ・Sitemap — すべてのブログが避けて通れない土台"
emoji: "🔍"
type: "tech"
topics: ["SEO", "CoreWebVitals", "Schema", "Sitemap", "WebDev"]
published: true
---

---
title: "テクニカルSEO三種の神器：Core Web Vitals・構造化データ・Sitemap — すべてのブログが避けて通れない土台"
date: 2026-03-06
author: "学習君"
category: "web-tech"
tags: ["SEO", "Core Web Vitals", "Schema", "JSON-LD", "Sitemap", "LCP", "INP", "テクニカルSEO", "2026"]
readingTime: 12
summary: "Googleのランキング要因の中で、コンテンツ制作者が見落としがちな「土台」がある：テクニカルSEOだ。Core Web Vitalsはページ体験スコアを、構造化データは検索結果の見た目を、Sitemapはコンテンツの発見可能性を決める。三つすべてがトラフィックに直結する。"
lang: "ja"
---

残酷な現実がある。3時間かけて書き上げた質の高い記事でも、ページ読み込みに5秒かかり、検索結果にリッチな表示がなく、Googleのクローラーがその記事の存在すら知らなければ——検索流量は期待外れになるかもしれない。

それがテクニカルSEOの存在理由だ。

神秘的な黒魔術でも、大企業だけが必要とするものでもない。真剣にブログを運営するなら、3つの基礎部品は避けて通れない：**Core Web Vitals（コアウェブバイタル）、構造化データ（Schema）、XML Sitemap**。

この記事ではこの3つを分解し、「実際に何をすべきか」に焦点を当てて解説する。

---

## 一、Core Web Vitals：Googleの「体験スコア」

Googleは2021年からページ体験をランキング要因に加えた。その核心が3つの指標——Core Web Vitals（CWV）だ：

| 指標 | 何を測定するか | 合格目標 |
|------|--------------|---------|
| **LCP**（最大コンテンツ描画） | メインコンテンツが読み込まれるまでの時間 | 2.5秒未満 |
| **INP**（次のペイントまでの遅延） | ユーザー操作後のページ応答速度 | 200ms未満 |
| **CLS**（累積レイアウトシフト） | ページ要素が予期せず移動する度合い | 0.1未満 |

### LCP：最も多い失点ポイント

驚くべき統計がある：**Webサイトの40%がLCP未達**、これが3指標の中で最大の課題だ。

LCPが悪化する主な原因：
- ヒーロー画像がCSSやJavaScriptで動的に読み込まれ、ブラウザが「発見できない」
- 画像が圧縮されておらず、ファイルサイズが大きい
- サーバーレスポンスが遅い（TTFB高）

**最も効果的な改善方法**：カバー画像をHTML `<img>` タグに変更し、`fetchpriority="high"` とWebPフォーマットを追加する。

```html
<img src="/hero.webp" fetchpriority="high" loading="eager" alt="..." width="1200" height="630">
```

### INP：2024年に登場した新指標

INPは2024年3月、FIDの後継としてCore Web Vitalsの第3指標に正式採用された。ユーザー操作（クリック、キー入力）からページが応答・描画を完了するまでの時間を測定する。

技術ブログにとって、コードハイライトライブラリ（Prism.js、highlight.js）はINPの隠れた敵だ——フル読み込みすると大きなJS長タスクが発生し、メインスレッドをブロックする。

解決策：コードハイライトを遅延読み込みし、コードブロックがビューポートに入った時だけ起動する。

### CLS：気づきにくい「レイアウトジャンプ」

最も多いCLSの原因：
- 画像に `width`/`height` 属性がない
- 遅延読み込み広告や埋め込みコンテンツにプレースホルダーがない
- フォント読み込み時のFOUT（スタイルなしテキストの点滅）

**診断ツール**：[PageSpeed Insights](https://pagespeed.web.dev) — 無料で、実際のページをリアルタイム分析し、具体的な改善提案を提供する。

---

## 二、構造化データ（Schema）：Googleにコンテンツを「理解させる」

検索結果で星評価、公開日、FAQ展開、著者アイコンが表示されているページがあるのはなぜか？

それはマジックではなく、**構造化データ**の効果だ。

### 構造化データとは？

標準化されたJSONコード（JSON-LDフォーマット）を使って、このコンテンツの「正体」をGoogleに伝える——記事なのかFAQなのか、著者は誰か、いつ公開されたか、どの画像が使われているか。

この標準はSchema.orgで定義されており、Google・Microsoft・Yahooが共同で策定した。

### やる価値はあるか？

結論から言う：**ある。特に2026年は重要だ。**

理由は2つ：
1. **Rich SnippetによるCTR向上**：検索結果でパンくずリスト、著者情報、FAQ展開が表示され、研究によればクリック率が20〜30%向上する。
2. **GEO（生成的エンジン最適化）**：ChatGPT、Perplexity、Google AI Overviewsで検索する際、構造化データがあるコンテンツはAIが「理解・引用」しやすくなる。

### 技術ブログに必須の2種のSchema

**Article Schema（各記事ページ）**：

```json
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "TechArticle",
  "headline": "記事タイトル",
  "author": {
    "@type": "Person",
    "name": "あなたの名前"
  },
  "datePublished": "2026-03-06",
  "dateModified": "2026-03-06",
  "description": "記事の概要（150文字以内）",
  "image": "https://yoursite.com/og-image.jpg"
}
</script>
```

**FAQPage Schema（Q&A形式の記事末尾）**：

```json
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [{
    "@type": "Question",
    "name": "質問内容",
    "acceptedAnswer": {
      "@type": "Answer",
      "text": "回答内容"
    }
  }]
}
</script>
```

**検証は必須**：デプロイ後は必ず [Google Rich Results Test](https://search.google.com/test/rich-results) で確認する。検証なしは目隠し飛行だ。

---

## 三、XML Sitemap：クローラーがコンテンツを「見つける」ための地図

SitemapはGoogleのクロールを速くするためではなく、**見落としを防ぐ**ためにある。

新しいブログや外部リンクが少ないサイトでは、Sitemapなしのページは数週間インデックスされないこともある。Sitemap + Google Search Consoleへの送信があれば、通常1〜2日でインデックスされる。

### 重要原則：Sitemapは「クリーンに」保つ

すべてのURLを入れてはいけない。**低品質なURLをSitemapに含めると、クロール予算が希薄化され、Googleが価値のあるページに時間を使えなくなる。**

**除外すべきURLの種類**：
- noindexページ（重複コンテンツ、タグ集約ページなど）
- パラメータURL（`?sort=date&page=2` など）
- 404ページ
- 低品質の下書きページ

### ブログが大きくなったときのSitemap分割構造

```xml
sitemap-index.xml
├── sitemap-posts.xml    <!-- すべての正式記事 -->
├── sitemap-tags.xml     <!-- タグページ（低価値なら除外可）-->
└── sitemap-pages.xml    <!-- About、Contactなどの静的ページ -->
```

### よくある誤解

`<priority>` と `<changefreq>` タグは、**Googleはほぼ無視している**。設定に時間をかける必要はない。URLが正確で、`<lastmod>` がコンテンツが実際に更新された時だけ変わっていれば十分だ。

---

## 実装の優先順位：何から始めるか？

今すぐ一つだけ始めるなら、この順序で：

1. **PageSpeed Insightsを一度実行する** — まず診断。LCP/INP/CLSの現在地を把握する
2. **Article Schemaを追加する** — 記事ごとに30分以内で設定でき、検索表示への効果は即効性がある
3. **SitemapがGoogle Search Consoleに送信済みか確認する** — 10分の作業だが、忘れている人が多い
4. **LCP問題を修正する** — 大抵は画像フォーマットと読み込み順序の問題。PageSpeedの報告書に従って修正する

---

## 結論：土台が整って初めて、コンテンツが立てる

良いコンテンツを書くことはスタートラインだ。テクニカルSEOは、そのコンテンツが「発見され、理解され、表示される」ための基盤インフラだ。

Core Web VitalsはGoogleの体験スコアを決め、構造化データは検索結果での見た目を決め、Sitemapはコンテンツがそもそも候補に入るかを決める。

三つとも、エンジニアでなくてもできる。最も簡単な一歩から始めよう：PageSpeed Insightsを開いて、自分のブログURLを入力し、スコアを見る。

その数字が、次に何をすべきかを教えてくれる。
