---
title: "localStorage でSPAの状態を"覚えさせる"——3つの実装パターンと落とし穴"
emoji: "💾"
type: "tech"
topics: ["JavaScript", "localStorage", "SPA", "フロントエンド", "WebStorage"]
published: true
---

ビルドツールなしの素のJavaScript SPAで、ページをリロードしても「さっきの画面」に戻りたい。React なら zustand + persist、Vue なら pinia-plugin-persistedstate があるけど、フレームワークを使っていない場合は自前で `localStorage` を叩くことになる。

ホームラボのダッシュボード（Single-file SPA、約3000行）を運用する中で、`localStorage` を3つの目的で使い分けている。それぞれのパターンと、実装して初めて気づいた落とし穴を書く。

---

## パターン1: ビュー状態の保持

SPAで一番ありがちな悩み——F5を押すとトップページに戻される。

```javascript
function showView(viewName) {
  document.querySelectorAll(".view").forEach(v => v.classList.remove("active"));
  document.getElementById(viewName + "View").classList.add("active");

  // 現在のビューを保存
  localStorage.setItem('dashboardCurrentView', viewName);
}

document.addEventListener("DOMContentLoaded", function() {
  const savedView = localStorage.getItem('dashboardCurrentView');
  if (savedView) {
    const el = document.getElementById(savedView + "View");
    if (el) {
      showView(savedView);
    } else {
      showView('simple-tasks'); // フォールバック
    }
  }
});
```

**落とし穴: 存在しないビューIDが保存されている場合。** ビューを削除・リネームした後にアクセスすると `getElementById` が `null` を返して死ぬ。保存値を信用せず、**必ずDOM上の存在確認を挟む**のが鉄則。フォールバック先もハードコードで持っておく。

---

## パターン2: テーマの永続化

5色のテーマを切り替えるボタンがある。localStorage がなければ「日付ベースのランダム選択」にフォールバックする設計にした。

```javascript
const themes = ["", "theme-blue", "theme-green", "theme-pink", "theme-purple"];
let currentTheme = 0;

function switchTheme() {
  document.body.className = "";
  currentTheme = (currentTheme + 1) % themes.length;
  if (themes[currentTheme]) document.body.classList.add(themes[currentTheme]);
  localStorage.setItem("dashboardTheme", currentTheme);
}

function loadTheme() {
  const saved = localStorage.getItem("dashboardTheme");
  // 保存値がなければ日付でランダム選択
  currentTheme = saved !== null
    ? parseInt(saved)
    : new Date().getDate() % themes.length;
  if (themes[currentTheme]) document.body.classList.add(themes[currentTheme]);
}

loadTheme(); // DOMContentLoaded を待たず即実行
```

**落とし穴: FOUC（Flash of Unstyled Content）。** テーマの読み込みを `DOMContentLoaded` の中に入れると、一瞬デフォルトテーマが見えてからパッと切り替わる。`loadTheme()` はスクリプト読み込み時に即実行し、`<script>` タグをできるだけ `<body>` の先頭に近い位置に置くのが対策。完全に消すなら `<html>` タグに `style="visibility:hidden"` をつけて JS で解除する手もあるが、やりすぎ感がある。

---

## パターン3: APIレスポンスのキャッシュ（Stale-While-Revalidate）

複数台のサーバー状態を取得するAPIは、内部でSSHを叩くので数秒かかる。その間ユーザーは白い画面を見ることになる。

解決策は **Stale-While-Revalidate パターン**。前回のレスポンスを localStorage に突っ込んでおき、次回アクセス時はまずキャッシュを描画→裏でAPIを叩いて差し替える。

```javascript
const CACHE_KEY = 'nodes_cache';

async function fetchNodes() {
  // Step 1: キャッシュがあればまず描画（白屏回避）
  const cached = localStorage.getItem(CACHE_KEY);
  if (cached) {
    try {
      nodes = JSON.parse(cached);
      renderAll();  // 古いデータでもいいから見せる
    } catch(e) {}
  }

  // Step 2: 裏で最新データを取得して上書き
  try {
    const r = await fetch("/api/nodes-status");
    if (r.ok) {
      const data = await r.json();
      nodes = data.nodes;
      localStorage.setItem(CACHE_KEY, JSON.stringify(data.nodes));
      renderAll();  // 最新データで再描画
    }
  } catch(e) {
    console.error("API error", e);
  }

  // Step 3: キャッシュもAPIもダメならハードコードデータ
  if (!cached) {
    nodes = getLocalNodeData();
    renderAll();
  }
}
```

**落とし穴: localStorage の5MB制限。** JSON.stringify したノードデータが1回あたり数KBでも、他の用途（タスクデータ、プロジェクト情報など）と合わせると意外と早く上限に近づく。`try/catch` で `QuotaExceededError` を拾う処理を入れておかないと、キャッシュ書き込み失敗でアプリ全体が止まるリスクがある。

もうひとつ、**キャッシュの鮮度管理を入れていない** ので、ブラウザを1週間開きっぱなしにすると1週間前のデータがまず表示される。タイムスタンプを一緒に保存して、古すぎたらキャッシュを無視する処理は検討中。

---

## 共通の設計判断: なぜ sessionStorage ではなく localStorage か

`sessionStorage` はタブを閉じると消える。ダッシュボードは「朝開いて1日つけっぱなし、たまにリロード」という使い方なので、タブを閉じても設定が残る `localStorage` のほうが自然だった。

逆にセキュリティ的に残したくない情報（APIトークンなど）は localStorage に入れてはいけない。今回のダッシュボードは LAN 内専用なので割り切っているが、公開サービスなら `httpOnly` Cookie やサーバーサイドセッションが正解。

---

## まとめ

| パターン | 保存対象 | 注意点 |
|---------|---------|--------|
| ビュー状態 | 文字列（ビュー名） | DOM存在チェック必須 |
| テーマ | 数値（インデックス） | FOUC対策で即実行 |
| APIキャッシュ | JSON文字列 | 5MB制限 + 鮮度管理 |

フレームワークの persist プラグインがやっていることを手動でやっているだけだが、**素のJSだと「何がどう永続化されているか」が全部見える**のは利点でもある。ブラックボックスのミドルウェアより、3行の `getItem/setItem` のほうがデバッグは楽だ。
