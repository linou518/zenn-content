---
title: "Flask + JavaScript でリアルタイム「ステップ進捗UI」を実装した話"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "自動化"]
published: true
---

---
title: "Flask + JavaScript でリアルタイム「ステップ進捗UI」を実装した話"
emoji: "⏳"
type: "tech"
topics: ["JavaScript", "Flask", "Python", "フロントエンド", "UX"]
published: true
---

# Flask + JavaScript でリアルタイム「ステップ進捗UI」を実装した話

Bot作成に16ステップかかる処理を、ユーザーが待ちながら見ていられるUIにした。

---

## 背景

ホームラボのBot管理ダッシュボードで「Botを作成する」ボタンを押すと、裏ではこんな処理が走る:

1. SSH接続確認
2. 既存Bot名との衝突チェック
3. ホームディレクトリ作成
4. テンプレートファイルをコピー
5. `agents.list` への登録
6. Telegram Botのバインド設定
7. `auth-profiles.json` シンボリックリンク作成
8. OpenClaw設定ファイル書き込み
9. Agent設定ファイル（`openclaw.json`）作成
10. Gateway再起動
11. 起動確認（ログ監視）
12. 開発区ディレクトリ作成
13. `TOOLS.md` 生成
14. シンボリックリンク最終確認
15. Gateway再起動待機（5秒）
16. 完了通知

全部成功しても30〜40秒かかる。`fetch` で叩いてスピナーを出すだけだと、ユーザーはフリーズしたように感じる。「何が起きているかわからない待ち時間」は体感でも実際より3倍長く感じるものだ。

---

## 設計の選択肢

リアルタイム進捗表示の実装パターンはいくつかある:

### 1. ポーリング
クライアントが定期的にサーバーに `GET /api/job-status/{id}` を叩く。シンプルだが、ジョブIDの管理、並行実行の排他制御、クリーンアップなど考えることが多い。

### 2. Server-Sent Events (SSE)
HTTP のストリーミングレスポンス。一方向（サーバー→クライアント）で十分なケースなら軽量。Flaskでは `Response(generator, mimetype='text/event-stream')` で実現できる。

### 3. WebSocket
双方向通信が必要な場合。今回は過剰。

### 4. 同期レスポンス + ステップを1ターンで全部返す
今回選んだのはこれ。

```python
@app.route('/api/create-bot-steps', methods=['POST'])
def create_bot_steps():
    steps = []
    
    # Step 1: SSH確認
    ok, out = ssh_cmd_with_retry(node, 'echo ok', retries=3)
    steps.append({'step': 1, 'label': 'SSH接続確認', 'ok': ok, 'detail': out})
    if not ok:
        return jsonify({'success': False, 'steps': steps})
    
    # Step 2: 衝突チェック
    ok, out = ssh_cmd(node, f'test -d {agent_dir} && echo exists || echo ok')
    if 'exists' in out:
        steps.append({'step': 2, 'label': '名前重複チェック', 'ok': False, 'detail': 'すでに存在します'})
        return jsonify({'success': False, 'steps': steps})
    steps.append({'step': 2, 'label': '名前重複チェック', 'ok': True, 'detail': ''})
    
    # ... 以降16ステップ続く
    
    return jsonify({'success': True, 'steps': steps})
```

同期でブロックするが、Flask がマルチスレッドで動いていれば他リクエストは詰まらない。ステップを配列に積んで最後に全部返す。

---

## クライアント側: フェイク進行アニメーション

問題はこれだと「ボタンを押した → 30秒後に全ステップが一気に表示される」になること。「リアルタイム感」がない。

解決策: **レスポンスが来てからステップを順番にアニメーション表示する**。

```javascript
async function createBot(formData) {
  showProgressModal();
  
  const res = await fetch('/api/create-bot-steps', {
    method: 'POST',
    body: JSON.stringify(formData),
    headers: {'Content-Type': 'application/json'}
  });
  const data = await res.json();
  
  // ステップを1つずつ遅延表示
  for (const step of data.steps) {
    await appendStep(step);
    await sleep(120);  // 120ms ずつ表示
  }
  
  // 最後に成功/失敗サマリー
  showResult(data.success);
}

async function appendStep(step) {
  const icon = step.ok ? '✅' : '❌';
  const el = document.createElement('div');
  el.className = `step-item ${step.ok ? 'ok' : 'fail'}`;
  el.innerHTML = `${icon} Step ${step.step}: ${step.label}`;
  if (step.detail) {
    el.innerHTML += `<span class="detail">${step.detail}</span>`;
  }
  progressLog.appendChild(el);
  el.scrollIntoView({behavior: 'smooth'});
}
```

120ms間隔で表示すると、16ステップで約2秒のアニメーション。実際の処理は30秒かかっていても、「動いている感」は最初の2秒でほぼ満たされる。

---

## SSEにしなかった理由

「同期レスポンス + クライアント側フェイクアニメーション」でほとんどのケースは十分だと気づいた。

SSEを使う必要があるのは:
- **処理が中断・キャンセルされる可能性がある**（例: ユーザーが途中でキャンセル）
- **処理時間が非常に長い**（5分以上）
- **複数クライアントがリアルタイムで同じ進捗を見る**

Bot作成は30〜40秒、ユーザーは1人、キャンセル不要。SSEは過剰だった。

「本物のリアルタイム性 vs 体感のリアルタイム性」を分けて考えると、設計がシンプルになる。

---

## 失敗ステップでの即時中断

APIは途中のステップで失敗したら即座に残ステップを省略して返す。クライアントは受け取ったステップだけをアニメーション表示して、末尾に赤いエラーメッセージを追加する。

```javascript
if (!data.success) {
  const failedStep = data.steps.find(s => !s.ok);
  appendError(`Step ${failedStep.step} で失敗: ${failedStep.detail}`);
}
```

ユーザーは「どこで失敗したか」が一目でわかる。デバッグ効率が上がった。

---

## まとめ

- **処理が長くても30秒未満かつシングルユーザー** → 同期レスポンス + クライアント側アニメーションで十分
- **ステップを配列で返す** → フロントエンドの表示ロジックがシンプルになる
- **体感の速さ** は「実際の速さ」より重要な場合がある
- **SSEやWebSocketは強力だが、シンプルな要件に持ち込むと実装コストが跳ね上がる**

ホームラボのツールこそ、「完璧な技術スタック」より「動いて維持できる実装」が正義だと改めて感じた。

