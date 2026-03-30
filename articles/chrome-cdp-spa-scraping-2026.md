---
title: "Chrome CDP でSPAをスクレイピングする — claude.ai 使用量を自動取得した話"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "自動化"]
published: true
---

---
title: "Chrome CDP でSPAをスクレイピングする — claude.ai 使用量を自動取得した話"
emoji: "🕷️"
type: "tech"
topics: ["Python", "scraping", "Chrome", "SPA", "asyncio"]
published: true
category: "web-tech"
---

# Chrome CDP でSPAをスクレイピングする — claude.ai 使用量を自動取得した話

> 2026-03-30 | techsfree-web

---

公式APIが存在しないWebアプリのデータを、どうやって自分のダッシュボードに取り込むか。
`requests`でHTMLを取得しようとしても、SPAは初期HTMLが空っぽ。BeautifulSoupも歯が立たない。
そんなとき、**Chrome DevTools Protocol（CDP）**が答えになった。

---

## なぜCDPを選んだか

claude.aiの使用量ページには、現在のトークン消費率・リセット時刻・追加課金状況などが表示される。複数アカウントを使っているため、毎回ブラウザで確認するのが面倒だった。

候補は3つあった：

| 方法 | 問題点 |
|------|--------|
| `requests` + BeautifulSoup | SPAなのでHTMLにデータがない |
| Selenium / Playwright | 起動コストが重い、別プロセス管理が必要 |
| **Chrome CDP** | 常時起動中のChromeを直接操作できる ✅ |

すでにChromeは`--remote-debugging-port`で起動している。使わない手はない。

---

## CDPの基本構造

CDP はWebSocketベースのプロトコルだ。デバッグポートの`/json`を叩くと、現在開いているタブ一覧がJSONで返ってくる。

```python
import urllib.request, json

tabs = json.loads(
    urllib.request.urlopen("http://127.0.0.1:9225/json").read().decode()
)
# [{title: "...", url: "...", webSocketDebuggerUrl: "ws://..."}, ...]
```

あとは対象タブの`webSocketDebuggerUrl`に接続して、コマンドを送るだけ。

```python
import asyncio, websockets

async def scrape_tab(tab):
    async with websockets.connect(
        tab['webSocketDebuggerUrl'], max_size=5*1024*1024
    ) as ws:
        # ページをリロード（最新データを取得するため）
        await ws.send(json.dumps({
            "id": 1, "method": "Page.reload",
            "params": {"ignoreCache": True}
        }))
        # ...
```

---

## SPAの壁：ロード完了 ≠ レンダリング完了

最初のハマりポイントがここだ。

`Page.loadEventFired`イベントを待っても、SPAはJavaScriptでDOMを後から描画する。イベント後すぐに`document.body.innerText`を読むと、まだローディングスピナーの状態だったりする。

解決策は**2段階待機**：

```python
# 1. Page.loadEventFired を待つ（最大10秒）
deadline = asyncio.get_event_loop().time() + 10
while asyncio.get_event_loop().time() < deadline:
    msg = json.loads(await asyncio.wait_for(ws.recv(), 2))
    if msg.get('method') == 'Page.loadEventFired':
        break

# 2. Reactのレンダリングを待つ追加sleep
await asyncio.sleep(3)  # ← これが重要
```

3秒は経験値。500ms では足りなかった。サイトによって調整が必要。

---

## JSインジェクションでデータ抽出

DOMが安定したら、`Runtime.evaluate`でJavaScriptを流し込む。

データはDOMに構造化されていないため、`document.body.innerText`を行単位で解析するアプローチをとった：

```javascript
(function() {
    const lines = document.body.innerText
        .split('\n').map(l => l.trim()).filter(l => l);

    const r = { session_pct: null, weekly_all_pct: null };

    for (let i = 0; i < lines.length; i++) {
        const line = lines[i];

        // "42% used" → パーセンテージ抽出
        const pct = line.match(/^(\d+)%\s*used$/);
        if (pct) {
            if (r.session_pct === null) r.session_pct = parseInt(pct[1]);
        }

        // "Resets in 2h 30m" → リセット時刻
        const ri = line.match(/^Resets in (\d+) hr (\d+) min$/);
        if (ri) r.session_reset = `${ri[1]}h ${ri[2]}m`;
    }

    return JSON.stringify(r);
})()
```

構造化されていないテキストへの**正規表現ゲリラ戦**だが、表示ロジックが変わらない限りは安定して動く。

---

## 複数アカウント対応

複数のアカウントを別々のChromeプロフィールで使っている。それぞれで使用量ページを開いておけば、CDPが全タブを返してくれる。

```python
usage_tabs = [
    t for t in tabs if "claude.ai/settings/usage" in t.get('url', '')
]
tasks = [scrape_tab(t) for t in usage_tabs]
results = await asyncio.gather(*tasks, return_exceptions=True)
```

`asyncio.gather`で並列実行。複数タブでも数秒で完了する。

アカウント名はページ内のテキストから取得し、マップで表示名に変換している。

---

## ダッシュボードへの統合

Flask側でAPIエンドポイントを作り、5分キャッシュを挟んでスクレイパーを呼び出す。フロントエンドはpolling不要で、ページロード時に一度取得するだけ。

```
[Chromeブラウザ] ←WebSocket→ [Pythonスクリプト] → [Flaskキャッシュ] → [Dashboard UI]
```

毎朝の「今月何%使ったっけ」確認が、ダッシュボードを開くだけで済むようになった。

---

## まとめ・注意点

CDPアプローチの要点：

- **メリット**: 既存のChromeセッションを使い回せる、認証不要、軽量
- **デメリット**: ページの内部構造変更に弱い、Chrome起動前提
- **注意**: `max_size=5*1024*1024`を設定しないとWebSocketがデフォルトサイズで切れる

公式APIがないSaaSのデータを取りたいとき、Seleniumよりもずっと軽量な選択肢として、CDPは手元のツールとして重宝している。

<!-- 本文已脱敏处理: 内部ポート番号は実装例として記載（一般的なデバッグポート） -->

