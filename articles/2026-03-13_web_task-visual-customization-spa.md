---
title: "SPAのタスク一覧に「色」と「太字」を足す設計判断"
emoji: "🎨"
type: "tech"
topics: ["javascript", "spa", "frontend", "localstorage", "uiux"]
published: true
---

# SPAのタスク一覧に「色」と「太字」を足す設計判断

プロジェクト管理ダッシュボードを運用していると、「このタスク、目立たせたい」という要望が来る。今回は自作SPAのタスク一覧に、タスクごとの色設定と太字切り替えを追加したときの設計判断を整理する。

---

## 要件：「重要なタスクを視覚的に区別したい」

タスク管理ページで数十個のタスクが並ぶと、どれが重要か一目でわからない。ユーザーから「色を付けたい」「太字にしたい」というリクエストが来た。

一見シンプルだが、設計上の選択肢がいくつかある。

## 選択肢1：タスクデータに直接プロパティを持たせる

```json
{
  "id": "task-001",
  "title": "APIドキュメント更新",
  "done": false,
  "color": "#e74c3c",
  "bold": true
}
```

**メリット：** シンプル。データと表示が一体化。
**デメリット：** バックエンドのAPIとデータ構造を変更する必要がある。既存のタスクデータに破壊的変更が入る。色やフォントは「表示の都合」であって「タスクの本質」ではない。

## 選択肢2：表示設定を別レイヤーで管理する

```javascript
// localStorage に表示設定だけ保存
const taskStyles = JSON.parse(localStorage.getItem('taskStyles') || '{}');

// 例: { "task-001": { color: "#e74c3c", bold: true } }
```

**メリット：**
- バックエンドを触らない。APIの変更ゼロ
- タスクデータの純粋性を保てる（完了/未完了だけがサーバー側の関心事）
- ユーザーごとのブラウザに閉じた設定（チーム利用時に他人に影響しない）

**デメリット：**
- ブラウザを変えると設定が消える
- 端末間同期がない

今回は個人利用のダッシュボードなので、選択肢2を採用した。

## 実装：コンテキストメニューで色を選ぶ

タスク行を右クリック（またはモバイルで長押し）するとカスタムコンテキストメニューが出る、という方式にした。理由は：

- タスクごとにカラーピッカーを常時表示するとUIが煩雑
- 右クリックメニューは「そこにあるけど邪魔にならない」UIパターン
- ブラウザ標準のコンテキストメニューを上書きすることに抵抗がなければ、一番スマート

```javascript
taskRow.addEventListener('contextmenu', (e) => {
  e.preventDefault();
  showStyleMenu(e.clientX, e.clientY, task.id);
});

function showStyleMenu(x, y, taskId) {
  const menu = document.createElement('div');
  menu.className = 'task-style-menu';
  menu.innerHTML = `
    <div class="color-row">
      ${['#e74c3c','#e67e22','#2ecc71','#3498db','#9b59b6','#1abc9c',''].map(c =>
        `<span class="color-dot${c === '' ? ' clear' : ''}" 
              style="background:${c || '#ccc'}" 
              data-color="${c}"></span>`
      ).join('')}
    </div>
    <label class="bold-toggle">
      <input type="checkbox" ${getTaskStyle(taskId).bold ? 'checked' : ''}>
      <b>B</b> 太字
    </label>
  `;
  menu.querySelectorAll('.color-dot').forEach(dot => {
    dot.onclick = () => {
      setTaskStyle(taskId, { color: dot.dataset.color });
      applyTaskStyle(taskId);
      menu.remove();
    };
  });
  document.body.appendChild(menu);
  menu.style.cssText = `position:fixed;left:${x}px;top:${y}px;z-index:9999`;
}
```

## ハマった点：CSSの優先度

タスクの色をインラインstyleで設定すると、既存のCSSクラス（`.done`で打ち消し線+グレー化するなど）と競合する。

```css
.task-item.done .task-title {
  text-decoration: line-through;
  color: #999;
  opacity: 0.6;
}
```

解決策は、完了タスクの場合はユーザー設定色にopacityをかぶせること。色は残るが「終わった感」は出る。

```javascript
function applyTaskStyle(taskId) {
  const el = document.querySelector(`[data-task-id="${taskId}"] .task-title`);
  const style = getTaskStyle(taskId);
  const isDone = el.closest('.task-item')?.classList.contains('done');
  
  if (style.color) {
    el.style.color = style.color;
    if (isDone) el.style.opacity = '0.5';
  } else {
    el.style.color = '';
    el.style.opacity = '';
  }
  el.style.fontWeight = style.bold ? 'bold' : '';
}
```

## もう一つのハマり：localStorageの肥大化

タスクを削除してもlocalStorageのスタイル設定は残り続ける。ゴミが溜まる。

対策として、タスク一覧のレンダリング時に、存在しないタスクIDのエントリを掃除する。

```javascript
function cleanupOrphanStyles(currentTaskIds) {
  const styles = JSON.parse(localStorage.getItem('taskStyles') || '{}');
  const idSet = new Set(currentTaskIds);
  let changed = false;
  for (const id of Object.keys(styles)) {
    if (!idSet.has(id)) {
      delete styles[id];
      changed = true;
    }
  }
  if (changed) localStorage.setItem('taskStyles', JSON.stringify(styles));
}
```

## 設計の振り返り

| 判断 | 理由 |
|------|------|
| 表示設定をlocalStorageに分離 | バックエンドAPI変更不要。データの責務分離 |
| コンテキストメニュー方式 | UI汚染ゼロ。パワーユーザー向き |
| 完了タスク＋カスタム色の共存 | opacityで「終わった感」を出しつつ色は残す |
| 不要エントリの自動掃除 | localStorage肥大化の予防 |

大げさな機能ではないが、「データの純粋性を守りつつ表示をカスタマイズする」という設計思想は、SPA全般に応用できる。APIが返すデータとUIの見た目は、別のレイヤーで管理したほうが後々楽になる。
