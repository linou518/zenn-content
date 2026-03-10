---
title: "cloudflaredのランダムURLに悩んだ話 — LINE Bot をWebSocket中継サーバーで永久解決した"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "自動化"]
published: true
---

# cloudflaredのランダムURLに悩んだ話 — LINE Bot をWebSocket中継サーバーで永久解決した

## 問題：再起動するたびにURLが変わる

自宅ネットワーク内のPC（GMKミニPC）にLINE Botを動かしたかった。

LINEのWebhookはHTTPSの固定URLが必要。プライベートIPしか持たないPCを外部に公開するには何らかのトンネルが要る。最初に選んだのが**cloudflared**（Cloudflare Tunnel）。

```bash
cloudflared tunnel --url http://localhost:5000
```

これで一時的なHTTPS URLが生成されてWebhookに使える。一見完璧に見えた。

**問題は「一時的」という部分だった。**

PCを再起動するたびにURLが変わる。`https://random-words-1234.trycloudflare.com` みたいなやつ。毎回LINE DevelopersコンソールにログインしてWebhook URLを手動で書き換える必要がある。

自分用のBotならまだしも、これを**消費者向けの製品**（GMK AIミニPC）に組み込もうとしていた。ユーザーに「再起動するたびにURL更新してください」なんて言えない。

## 検討した選択肢

| 方法 | 問題 |
|------|------|
| cloudflared 名前付きトンネル | Cloudflare認証が必要、エンドユーザーには複雑 |
| ngrok固定URL | 有料プラン必要 |
| ポートフォワーディング | ユーザーのルーター設定に依存、非現実的 |
| **WebSocket中継サーバー** | ← これを採用 |

## 解決策：自前の中継サーバー

発想の転換。「PCをサーバーとして外部公開する」のではなく、「PCがサーバーに接続しに行く」にした。

```
LINE Platform
    ↓ Webhook (HTTPS)
https://linebot.techsfree.com/gmk/{token}/callback  ← 固定URL！
    ↓ Nginx (443) → FastAPI (3500)
web.34: gmk_relay.py  ← 中継サーバー
    ↓ WebSocket転送
GMK PC: line_bridge_v2.py  ← WebSocketクライアント
    ↓ OpenClaw chatCompletions API
    ↓ LINE Reply API（直接）
```

ポイントは**方向が逆**なこと。

従来: `LINE → cloudflared → GMK PC`（外からPCに入ってくる）  
新方式: `LINE → 中継サーバー ← GMK PC`（PCがサーバーに接続維持）

PCからサーバーへのWebSocket長接続を維持しておけば、NAT越えもファイアウォールも関係ない。WebSocketはクライアントが発信するので、どんな環境でも繋がる。

## 実装

**中継サーバー側 (FastAPI + WebSocket)**

```python
# gmk_relay.py の核心部分（概念）
app = FastAPI()

# 各GMKユニットのWebSocket接続を保持
connections: dict[str, WebSocket] = {}

@app.websocket("/gmk/{token}/ws")
async def websocket_endpoint(websocket: WebSocket, token: str):
    await websocket.accept()
    connections[token] = websocket
    try:
        while True:
            await websocket.receive_text()  # ping/pong維持
    except:
        del connections[token]

@app.post("/gmk/{token}/callback")
async def line_webhook(token: str, request: Request):
    body = await request.body()
    ws = connections.get(token)
    if ws:
        await ws.send_text(body.decode())  # GMKに転送
    return {"status": "ok"}
```

**GMK PC側 (WebSocketクライアント)**

```python
# line_bridge_v2.py の核心部分（概念）
async def run():
    relay_url = f"wss://linebot.techsfree.com/gmk/{token}/ws"
    async with websockets.connect(relay_url) as ws:
        async for message in ws:
            data = json.loads(message)
            # LINEイベントを処理してAI返信
            reply = await call_openclaw(data)
            await send_line_reply(reply)
```

各GMKユニットは起動時に固有の `relay_token` を持ち、中継サーバーに接続を維持する。LINE WebhookのURLは `https://linebot.techsfree.com/gmk/{token}/callback` で固定。再起動しても変わらない。

## ハマりポイント：ヘアピンNAT

テスト中に1つハマった。

テスト用GMK（192.168.3.88）から `wss://linebot.techsfree.com/...` に接続しようとするとタイムアウト。中継サーバー（192.168.3.34）も同一LAN内にいるので、**ヘアピンNAT**（同一LANからの外部ドメイン経由のアクセス）がルーターによって遮断されていた。

解決策：テスト機のみ直接IP接続に変更。

```json
{
  "relay_url": "ws://192.168.3.34:3500/gmk/{token}/ws"
}
```

本番ユーザーの機器は別のネットワーク（自宅LAN）にあるので、`wss://linebot.techsfree.com/...` で問題なくつながる。

## 結果

- ✅ 固定WebhookURL（再起動しても変わらない）
- ✅ cloudflared不要（依存関係が減った）
- ✅ ユーザー手順がシンプル：URL貼るだけ
- ✅ 1台の中継サーバーで複数GMKユニット対応

ユーザーの設定手順は結局こうなった：

1. GMK起動
2. LINE DevelopersでBot作成（Channel Secret/Token取得）
3. Dashboard UIにSecret/Token入力
4. 表示された固定URLをLINE Consoleに貼る
5. 完了

これなら消費者向け製品として出せる。

## まとめ

「サービスを外部公開する」という発想から「サービスがサーバーに接続しに行く」という逆転の発想で、cloudflaredのランダムURL問題を根本から解決できた。

WebSocketの長接続を活用したこのパターンは、LINE Bot以外にも（IoTデバイスのリモート管理、自宅サーバーへのリバースプロキシなど）応用できそう。

---

**タグ案:** `LINE Bot`, `WebSocket`, `FastAPI`, `NAT越え`, `cloudflared`, `Python`, `インフラ`, `OpenClaw`

