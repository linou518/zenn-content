---
title: "ノード管理UIのボタン地獄を解消した——アコーディオン式操作パネルの実装"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "自動化"]
published: true
---

# ノード管理UIのボタン地獄を解消した——アコーディオン式操作パネルの実装

管理画面を育てていると、必ずぶつかる問題がある。**ボタンが増えすぎる問題**だ。

TechsfreeのDashboardには、OpenClawで動く各ノード（サーバー）を管理するためのノード一覧画面がある。最初は「再起動」「バックアップ」の2ボタンだったのに、気づいたら「バックアップ」「還元」「再起動」「Bot追加」「バックアップ一覧」「診断」「ログ」「退役」「Bot削除」と9つのアクションが並んでいた。

ノードが22台。カードが22枚。1枚に9ボタン。

**縦スクロールが地獄だった。**

---

## 問題の構造

ノード管理UIはカード形式で各ノードを表示している。カードには：

- ノード名・ホスト名・ポート
- 動作中のAgentタグ一覧
- ディスク使用量などのステータス
- 操作ボタン群

この「操作ボタン群」が問題だった。頻繁に使うのは「再起動」と「バックアップ」くらい。「退役」「Bot削除」なんてほとんど触らない。それなのに常時全展開で表示されていた。

```css
/* 改善前：ボタンがそのまま並ぶだけ */
.nm-node-actions {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 6px;
}
```

---

## 解決策：アコーディオン式でデフォルト折りたたみ

シンプルに「⚙️ 操作 ▼」というトグルボタンを設けて、クリックしたときだけ操作パネルを展開する方式に変更した。

### HTML構造

```html
<button class="nm-actions-toggle" onclick="nmToggleActions('${n.id}', this)">
  <span>⚙️ 操作</span>
  <span class="nm-toggle-arrow">▼</span>
</button>
<div class="nm-actions-body" id="nmActionsBody-${n.id}">
  <div class="nm-node-actions">
    <!-- 操作ボタン群 -->
  </div>
</div>
```

### CSS

```css
.nm-actions-toggle {
  width: 100%;
  padding: 5px 0;
  background: none;
  border: none;
  border-top: 1px solid var(--border);
  font-size: 11px;
  color: var(--muted);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 4px;
  transition: background 0.15s;
}

.nm-actions-toggle:hover {
  background: var(--bg);
  color: var(--text);
}

/* デフォルトは非表示 */
.nm-actions-body {
  display: none;
  overflow: hidden;
}

/* openクラスで展開 */
.nm-actions-body.open {
  display: block;
}
```

### JavaScript

```javascript
function nmToggleActions(nodeId, btn) {
  const body = document.getElementById('nmActionsBody-' + nodeId);
  const arrow = btn.querySelector('.nm-toggle-arrow');
  if (!body) return;
  const isOpen = body.classList.contains('open');
  body.classList.toggle('open', !isOpen);
  if (arrow) arrow.textContent = isOpen ? '▼' : '▲';
}
```

たった20行の変更だが、効果は絶大だった。

---

## なぜ `display: none` を直接切り替えるのか

アコーディオンの実装はいくつか流派がある：

| 方法 | メリット | デメリット |
|------|---------|-----------|
| `display: none/block` 切り替え | シンプル、高速 | アニメーションなし |
| `max-height` アニメーション | スムーズな開閉 | 高さ計算が面倒、コンテンツ量によってズレる |
| `height` + `overflow` | 正確なアニメーション | JS で高さを取得・セットする必要あり |

管理画面の場合、**アニメーションよりも確実な動作を優先した**。`max-height` アニメーションは「コンテンツが増えたとき想定外の高さになる」バグが出やすい。ノード管理UIはBot数によってコンテンツ量が変動するため、`display` の単純切り替えが正解だった。

---

## 実装後の改善点

- カード1枚あたりの高さが**約40%削減**
- 22枚並べても縦スクロールが快適に
- 「操作したいときだけ開く」UXで誤クリックも減少
- `▼ / ▲` の矢印で状態が一目でわかる

管理画面に限らず、「情報密度が高いカード型UI」でよく使えるパターンだ。ダッシュボード、設定ページ、テーブルの行展開など。シンプルだが、実装するタイミングを適切に判断することが大事——ボタンが3個以下なら折りたたみは不要、5個を超えたら検討し始める、というのが自分の感覚的な閾値だ。

---

**タグ案:** `JavaScript`, `CSS`, `フロントエンド`, `UI設計`, `アコーディオン`, `管理画面`, `SPA`, `Techsfree`, `ノード管理`, `ダッシュボード`

