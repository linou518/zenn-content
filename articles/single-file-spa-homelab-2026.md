---
title: "1ファイルで作るホームラボダッシュボード — Single-file SPAの設計と葛藤"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "自動化"]
published: true
---

---
title: "1ファイルで作るホームラボダッシュボード — Single-file SPAの設計と葛藤"
emoji: "🏠"
type: "tech"
topics: ["JavaScript", "Flask", "Python", "SPA", "個人開発"]
published: true
category: "web-tech"
---

# 1ファイルで作るホームラボダッシュボード — Single-file SPAの設計と葛藤

> 2026-03-29 | techsfree-web

---

ホームラボのダッシュボードを作るとき、最初の選択肢はいくつかある。React + Vite で本格的に作るか、Next.js で SSR も視野に入れるか。それとも……1枚のHTMLファイルに全部書くか。

うちは後者を選んだ。結果、`index.html` が 1100行を超えた。

---

## なぜ Single-file SPA なのか

理由は単純だ。**デプロイが `cp` 一発で終わる**。

本番環境（といっても自宅サーバーだが）で動かすとき、ビルドパイプラインが壊れて焦った経験は誰にでもある。`node_modules` が消えた、依存がロックされていない、パスが違う……そういうトラブルが起きるたびに「このダッシュボードにそこまで必要か？」と思う。

Flask でバックエンドを動かし、`/` でその HTML を返すだけ。CDN から Vue や Chart.js を読み込めば、それで SPA が成立する。systemd サービス 1 つで管理できる軽さは、24時間動かすホームラボ用途に向いている。

---

## 1000行越えでも維持できる構造

問題はスケールだ。機能が増えるたびにファイルが膨らむ。気づいたら `index.html` が 1100 行を超えていた。

それでも維持できている理由は、**ビュー単位でコードを整理しているから**。

```html
<!-- ===== VIEW: overview ===== -->
<div id="view-overview" class="view">
  ...
</div>

<!-- ===== VIEW: nodes ===== -->
<div id="view-nodes" class="view">
  ...
</div>
```

JavaScript も同じで、`showView(name)` という関数で全ビューを切り替える。アクティブなビューだけ `display: block`、それ以外は `display: none`。ルーターは要らない。

```javascript
function showView(name) {
  document.querySelectorAll('.view').forEach(v => v.style.display = 'none');
  document.getElementById(`view-${name}`).style.display = 'block';
  localStorage.setItem('dashboardCurrentView', name);
  // 各ビューの初期化処理
  if (name === 'nodes') loadNodes();
  if (name === 'overview') loadOverview();
}
```

`localStorage` でビューの状態を保持しているので、ページリロード後も元のビューに戻る。地味だが便利。

---

## Flask との連携パターン

バックエンドは Flask。API エンドポイントをシンプルに定義し、フロントから `fetch()` で呼ぶだけ。

```python
@app.route('/api/nodes-status')
def api_nodes_status():
    # SSH でノード状態を取得
    ...
    return jsonify(result)
```

フロント側：

```javascript
async function loadNodes() {
  const res = await fetch('/api/nodes-status');
  const data = await res.json();
  renderNodes(data);
}
```

CORS の設定もいらない。同一オリジンで動くから。このシンプルさが、単一ファイル構成の強みだ。

---

## 複雑さが増えたとき

Bot の作成・削除など、16 ステップの非同期処理が必要になったとき、Single-file SPA の限界を感じた。

ステップ進捗の表示、エラーハンドリング、SSE か Polling かの選択……結局「1 ファイル」という制約の中で解決した。

モーダルダイアログを使ったログ表示領域を設け、各ステップの結果を `<div>` に追記していく方式。WebSocket も SSE も使わず、fetch + 120ms遅延アニメーションで十分なリアルタイム感を作れた。

---

## 実際にやめたこと

- **ビルドステップ**: 不要。CDN でいい。
- **TypeScript**: 補完は欲しいが、1ファイルに型定義を書くメリットは薄い。
- **コンポーネント化**: Vue の SFC を使いたい誘惑があったが、ビルドなしで SFC は使えない。テンプレートリテラルで妥協。
- **テスト**: ……書いていない。ごめんなさい。

---

## いつ卒業するか

正直なところ、単一ファイル SPA はいつか限界が来る。うちのダッシュボードも、あと 2〜3 ビューが増えたら分割を検討するかもしれない。

その目安として考えているのは：
- `index.html` が 2000 行を超えたとき
- 複数人で編集するようになったとき
- CI/CD パイプラインを整備する余裕ができたとき

それまでは「1 ファイルで動く」シンプルさを享受する。ホームラボは楽しいことが大事だ。

---

## まとめ

| 観点 | Single-file SPA | モダンフレームワーク |
|------|-----------------|---------------------|
| デプロイ | `cp` 一発 | ビルドパイプライン必要 |
| 依存管理 | CDN のみ | node_modules 管理が必要 |
| 開発体験 | シンプル、型補完なし | 高機能 IDE サポート |
| スケール | 〜2000行が限界 | 無制限 |
| 向いてる用途 | 個人・ホームラボ | チーム開発・プロダクション |

「本番で使えるか？」と聞かれると正直微妙だ。でも、**自分だけが使う、壊れても自分が直す**ホームラボなら、この割り切りは十分合理的だと思っている。

<!-- 本文已脱敏处理: 敏感情報なし、脱敏不要 -->

