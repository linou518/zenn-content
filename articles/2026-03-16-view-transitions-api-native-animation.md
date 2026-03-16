---
title: "View Transitions API：ブラウザネイティブのページ遷移アニメーション、Framer Motionとお別れ"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "自動化"]
published: true
---

---
title: "View Transitions API：ブラウザネイティブのページ遷移アニメーション、Framer Motionとお別れ"
date: 2026-03-16
author: "学習君"
category: "web-tech"
tags: ["View Transitions API", "CSSアニメーション", "フロントエンド", "Web API", "SPA", "MPA", "ブラウザネイティブ"]
readingTime: 8
summary: "2025年10月にBaselineへ正式仲間入り。Chrome/Safari/Edge/Firefoxが全面サポート。ゼロ依存・ゼロバンドル・優雅なフォールバック——View Transitions APIがネイティブページ遷移アニメーションをFramer Motion不要で実現。仕組みから実装の落とし穴まで5分で習得。"
lang: "ja"
draft: false
---

ReactアプリでFramer Motionを丸ごとインストールして、やりたかったのはページ切り替えのフェードインだけ、という経験はないだろうか。

あるいはマルチページアプリで`requestAnimationFrame`を手書きし、スナップショットの管理や合成まで自分でやって、それでも結果がネイティブアプリより不自然だった、という経験は？

2025年10月、公式の解決策が登場した。

**View Transitions APIがBaseline Newly Availableに正式追加**——同一ドキュメント内の遷移はChrome 111+、Edge 111+、Firefox 133+、Safari 18+で全面サポート。ブラウザがネイティブでスナップショット・合成・アニメーションを処理し、開発者は「DOMが変わる」と伝えるだけでいい。

---

## 仕組み：3フェーズのスナップショット、ブラウザがすべて担当

View Transitions APIの設計は非常にエレガントだ。遷移アニメーションを3つのフェーズに分解する：

1. **旧状態のスクリーンショット** — ブラウザが現在のページのスナップショットを撮影
2. **DOM変更の実行** — コールバック関数が実行され、DOMが更新される
3. **新状態のスクリーンショット → 差分計算 → アニメーション生成** — ブラウザが自動的に補間

これはすべて1フレーム内で発生し、ユーザーには瞬間的な切り替えは見えず、滑らかな遷移だけが見える。

最も重要なポイント：**ブラウザがスナップショットと合成を担当し、開発者はロジックだけに集中できる。**

---

## 最小実装：1行のコード

### SPA（シングルページアプリケーション）

```javascript
// 変更前
updateTheDOMSomehow();

// 変更後
document.startViewTransition(() => updateTheDOMSomehow());
```

この1行だけ。デフォルト効果はクロスフェードで、CSSを一切書く必要がない。

### MPA（マルチページアプリケーション）

```css
/* サイト全体のページ遷移を2行のCSSで解決 */
@view-transition {
  navigation: auto;
}
```

グローバルCSSに追加するだけで、すべての`<a>`リンクのページ遷移に自動でアニメーションが付く。ROIが極めて高い。

---

## カスタムアニメーション：CSSの疑似要素で制御

デフォルトのフェードでは物足りない？遷移中にブラウザは2つのCSS疑似要素を公開する：

```css
/* 旧状態のスナップショット——左にスライドアウト */
::view-transition-old(root) {
  animation: slide-out-left 0.3s ease-out;
}

/* 新状態——右からスライドイン */
::view-transition-new(root) {
  animation: slide-in-right 0.3s ease-out;
}

@keyframes slide-out-left {
  to { transform: translateX(-100%); }
}

@keyframes slide-in-right {
  from { transform: translateX(100%); }
}
```

通常のCSSアニメーションと同じ感覚で、時間・イージング・アニメーション効果をCSSで制御できる。

---

## 命名遷移：要素レベルのMorphアニメーション

これがView Transitionsの最も強力な機能——**同名の異なる状態の要素間でMorphを自動実行する。**

「リストのサムネイルをクリックして詳細の大きな画像にスムーズに展開する」を実現したい？

```css
/* リストページのサムネイル */
.thumbnail {
  view-transition-name: product-image;
}

/* 詳細ページの全画面画像 */
.fullscreen-image {
  view-transition-name: product-image;
}
```

ブラウザが異なるページ/状態での2つの要素の位置とサイズの差を自動計算し、補間アニメーションを生成する。座標の手動計算もabsolute配置のテクニックも不要。

**適したシナリオ：**
- リスト→詳細ページ遷移（iOSスタイルのカード展開）
- 画像ギャラリーの全画面展開
- ショッピングカートのフライイン
- データテーブルの行展開

---

## 既存ソリューションとの比較

| ソリューション | バンドルサイズ | フレームワーク依存 | MPA対応 | ブラウザサポート |
|--------------|-------------|----------------|---------|--------------|
| Framer Motion | ~50KB | React | ❌ | 全ブラウザ |
| React Spring | ~25KB | React | ❌ | 全ブラウザ |
| GSAP | ~70KB | なし | 手動 | 全ブラウザ |
| **View Transitions API** | **0KB** | **なし** | **✅** | Chrome/Edge/Safari/Firefox(SPA) |

---

## Promise API：タイミングの精密制御

```javascript
const transition = document.startViewTransition(() => updateDOM());

// アニメーション開始を待つ（疑似要素が作成され、アニメーション直前）
await transition.ready;
// ここでWeb Animations APIでさらに制御可能

// アニメーション完了を待つ
await transition.finished;
// クリーンアップや次の操作をここで行う
```

チェーンアニメーションの編成、アニメーション完了後のアナリティクスイベント発火、遷移後のルーター状態更新などに使える。

---

## 実装の落とし穴リスト

**1. スナップショットはビットマップ、リアルタイムDOMではない**

アニメーション中、要素はスクリーンショット（画像）になる。`font-size`アニメーションや`clip-path`の変化はアニメーション中に反映されない。

**2. 同名要素はページ内で唯一でなければならない**

各`view-transition-name`は同一ページで1つの要素のみが使用できる。リストアイテムのMorphには動的な名前割り当てが必要（`view-transition-name: item-${id}`）。

```javascript
item.style.viewTransitionName = `item-${item.dataset.id}`;
document.startViewTransition(() => navigateToDetail());
```

**3. クロスページのMorphはアスペクト比を維持する必要がある**

2つのページの同名要素のアスペクト比が一致しないと、Morphアニメーションが変形する。

**4. FirefoxのMPAサポートはまだ未対応**

クロスドキュメントナビゲーション（MPAページ遷移）のView TransitionsはFirefoxが未対応（2026年対応予定）。`@supports`で段階的強化：

```css
@supports (view-transition-name: none) {
  @view-transition {
    navigation: auto;
  }
}
```

**5. INPメトリクスへの注意**

遷移アニメーションがメインスレードを長時間占有すると、INP（Interaction to Next Paint）メトリクスに影響する。アニメーションはシンプルに保ち、コールバック内で複雑な同期処理は避ける。

---

## 優雅なフォールバック：非対応ブラウザに影響なし

```javascript
function showView(name) {
  if (document.startViewTransition) {
    document.startViewTransition(() => _doShowView(name));
  } else {
    _doShowView(name);  // フォールバック：アニメーションなしで直接切り替え
  }
}
```

ゼロブレイキング、プログレッシブエンハンスメント、既存ユーザーに影響なし。

---

## まとめ

View Transitions APIは十分に成熟しており、新プロジェクトのデフォルト選択に値する。

SPAでは`startViewTransition`の1行でFramer Motionの大量のボイラープレートを置き換え。MPAでは2行のCSSでサイト全体のページ遷移に効果を付与。iOSレベルのカード展開が必要なら`view-transition-name`のMorphで対応可能。

バンドルサイズ0KB、フレームワーク依存なし、フォールバックリスクゼロ。このAPIは過小評価されている。

---

*データソース：Trevor Lasn's View Transitions API Guide；Cyd Stumpel's Practical CSS View Transitions Guide；MDN Web Docs Baseline 2025*

