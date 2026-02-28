---
title: "View Transitions APIで脱・アニメーションライブラリ——5行でSPAをネイティブアプリ並みに"
emoji: "✨"
type: "tech"
topics: ["javascript", "css", "frontend", "webdev", "animation"]
published: true
---

2025年10月、`View Transitions API` がBaseline入りした。Chrome・Edge・Firefox・Safari全対応。依存ゼロ、コスト最小、効果は本物だ。

今日、自分で管理しているダッシュボード（SPA + Flask）に適用するにあたって調べたことをまとめる。

---

## そもそも何をしてくれるのか

ページの「before」と「after」を自動でスクリーンショットして、ブラウザが差分アニメーションを生成してくれる仕組みだ。

処理フローはシンプルで3ステップ:

1. **旧状態をキャプチャ** — ブラウザが現在の画面をスナップショット
2. **DOMを更新** — 開発者のコールバックが動く
3. **新状態をキャプチャ → アニメーション** — 差分を計算して自動補間

これが単一フレーム内で起きるから、ユーザーには「なめらかな切り替え」だけが見える。

---

## 最小構成: 本当に5行で終わる

### SPA（シングルページアプリ）

```javascript
document.startViewTransition(() => updateTheDOMSomehow());
```

これだけ。デフォルトはクロスフェードだが、CSSで好きなアニメーションに変えられる。

自分のダッシュボードに適用するなら:

```javascript
function showView(name) {
  if (document.startViewTransition) {
    document.startViewTransition(() => _doShowView(name));
  } else {
    _doShowView(name); // 非対応ブラウザはそのまま動く
  }
}
```

既存の `showView()` を5行ラップするだけで、「概要 → 家族 → ノード管理」間の切り替えがスライドアニメーションになる。

### MPA（マルチページ / SSR）

```css
@view-transition {
  navigation: auto;
}
```

CSSを2行追加するだけ。Next.jsやLaravelなど、SPAでないプロジェクトでも即日使える。

---

## カスタムアニメーション: CSSで自由に

デフォルトのフェードに飽きたら、伪元素でアニメーションを上書きできる。

```css
@keyframes slide-out {
  to { transform: translateX(-100%); opacity: 0; }
}
@keyframes slide-in {
  from { transform: translateX(100%); opacity: 0; }
}

::view-transition-old(root) {
  animation: slide-out 0.3s ease-out;
}
::view-transition-new(root) {
  animation: slide-in 0.3s ease-out;
}
```

ページ種別によってアニメーションを変えたいときは `data-direction` 属性などと組み合わせれば細かく制御できる。

---

## 本命: 要素レベルのMorphアニメーション

`view-transition-name` を使うと、「サムネイル → フルスクリーン展開」のような要素間 morph アニメーションが実現できる。

```css
.thumbnail { view-transition-name: product-image; }
.detail-image { view-transition-name: product-image; }
```

同じ名前を持つ要素の間で、ブラウザが位置・サイズの差を自動補間する。Framer Motionで手書きしていた `layoutId` アニメーションが、ピュアCSSで書ける。

適用が映える場面:
- リスト → 詳細ページ遷移
- 画像ギャラリーのズームイン
- カートに飛ぶアニメーション
- タスクカードの展開

---

## 他ライブラリとの比較

| 手段 | バンドルサイズ | フレームワーク依存 | MPA対応 |
|------|--------------|-----------------|--------|
| Framer Motion | ~50KB | React必須 | ❌ |
| React Spring | ~25KB | React必須 | ❌ |
| GSAP | ~70KB | なし | △（手動実装） |
| View Transitions API | **0KB** | **なし** | ✅ |

新規プロジェクトで「それほど複雑なアニメーションは要らない」という要件なら、今日から View Transitions を使わない理由がない。

---

## 踏んでおきたい落とし穴

**1. スナップショットは「画像」**  
アニメーション中の要素はビットマップになる。`clip-path` や `overflow: hidden` は動画中に効かないケースがある。

**2. `view-transition-name` はページ内で一意**  
同名が複数存在すると最後の要素が勝つ。動的リストで使うときは要注意（`view-transition-name: item-${id}` パターンで対応）。

**3. 縦横比が異なるとモーフが崩れる**  
遷移元と遷移先の同名要素の縦横比が大きく違うと、アニメーション中に変形する。テキスト折り返しで起きやすい。

**4. Firefox の MPA 非対応はまだ継続中**  
Same-document（SPA内）は対応済みだが、Cross-document（MPA跨ページ）はFirefox未対応。2026年中に対応予定とされているが、MPA + Firefox を切れないプロジェクトは `@supports` で段階的導入を。

---

## 今日実際にやったこと

ダッシュボードの `showView()` に `startViewTransition` を仕込んで、ビュー切り替えを試験した。コードの変更は6行。既存の動作は一切壊れない（非対応ブラウザはelse分岐がそのまま走る）。

アニメーション実装にこれだけコストが下がったのは正直驚いた。「ライブラリ入れるほどではないが、パキっとした切り替えが欲しい」という場面に完全にはまる。

---

## まとめ

- SPAなら `startViewTransition` で1行、MPAならCSSで2行
- 依存ゼロ、降級もゼロコスト
- 要素レベルのmorphはFramer Motionの `layoutId` 相当
- Firefox MPA はまだ待ち、それ以外は本番投入して問題なし

Chrome DevToolsの「Animations」パネルでView Transition中のタイムラインを確認できるので、チューニングもしやすい。次のプロジェクトでは最初から組み込むつもりだ。

---

**タグ案:** `#JavaScript` `#CSS` `#WebAPI` `#フロントエンド` `#アニメーション` `#パフォーマンス` `#ViewTransitions` `#SPA` `#Web標準`
