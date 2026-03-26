---
title: "LINE Bot を cloudflared なしでホームラボに繋ぐ——WebSocket 中継方式の実装"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "自動化"]
published: true
---

---
title: "LINE Bot を cloudflared なしでホームラボに繋ぐ——WebSocket 中継方式の実装"
emoji: "🔌"
type: "tech"
topics: ["Python", "WebSocket", "LINEBot", "ホームラボ", "OpenClaw"]
published: true
---

# LINE Bot を cloudflared なしでホームラボに繋ぐ——WebSocket 中継方式の実装

LINE Bot をローカル環境で動かすとき、最初に引っかかるのが「外部からの Webhook 受信」問題だ。LINE はサーバーから HTTPS で叩いてくる。ローカルマシンには当然グローバル IP がない。

よくある解決策は **ngrok** や **cloudflare Tunnel (cloudflared)** だ。でもこれらには使いにくさがある。ngrok は無料枠だと URL が起動ごとに変わる。cloudflared はデーモン常駐が必要で、設定も煩雑だ。

今回、TechsFree では別のアプローチを採った。**WebSocket 中継サーバー**を挟む方式だ。

---

## アーキテクチャ

```
LINE → HTTPS POST → 中継サーバー(linebot.techsfree.com)
                         ↓ WebSocket
                    ローカル GMK ノード (line_bridge.py)
                         ↓ HTTP POST
                    OpenClaw chatCompletions API (localhost:18789)
                         ↓
                    LINE Reply API → ユーザー
```

中継サーバーは常時 HTTPS を受け付けるクラウド側に置く。ローカルの `line_bridge.py` はそこへ **WebSocket で接続を張り続ける**。接続は内から外への発信なのでファイアウォールを抜ける。LINE の Webhook が来たら、中継サーバーはその内容を WebSocket で転送してくる。

### なぜ WebSocket か

HTTP ポーリングにすると遅延が出る。Server-Sent Events は単方向だ。WebSocket なら双方向・低遅延・接続維持が揃う。中継パターンとして最適な選択だ。

---

## 実装のポイント

### 1. 接続の自動再接続（指数バックオフ）

ネットワークは必ず切れる。家庭のルーターは定期再起動するし、スリープから復帰した直後に接続が落ちることもある。

```python
reconnect_delay = RECONNECT_DELAY  # 5秒
MAX_RECONNECT_DELAY = 60

while True:
    try:
        async with websockets.connect(relay_url, ping_interval=20, ping_timeout=10) as ws:
            reconnect_delay = RECONNECT_DELAY  # 成功したらリセット
            async for raw_message in ws:
                ...
    except ConnectionClosed as e:
        logger.warning(f"Connection closed: {e}")
    except Exception as e:
        logger.error(f"Connection error: {e}")

    await asyncio.sleep(reconnect_delay)
    reconnect_delay = min(reconnect_delay * 2, MAX_RECONNECT_DELAY)
```

**成功したら遅延をリセット**するのが大事だ。エラー時だけ指数バックオフを伸ばしていく。最大60秒で頭打ち。無限に待つことはない。

### 2. LINE 署名検証

LINE Webhook は必ず `x-line-signature` ヘッダーをつけてくる。HMAC-SHA256 で検証しないと偽リクエストを処理してしまう。

```python
def verify_signature(body: str, signature: str, channel_secret: str) -> bool:
    hash_val = hmac.new(
        channel_secret.encode("utf-8"),
        body.encode("utf-8"),
        hashlib.sha256
    ).digest()
    expected = base64.b64encode(hash_val).decode("utf-8")
    return hmac.compare_digest(signature, expected)
```

`hmac.compare_digest` を使っているのはタイミング攻撃を避けるためだ。`==` で比較すると文字列の一致/不一致が処理時間に出ることがある。

### 3. 設定ファイルの毎回再読み込み

Webhook が届くたびに設定ファイルを読み直している：

```python
elif msg_type == "webhook":
    current_config = load_config()  # 毎回再読み込み
    asyncio.create_task(handle_webhook(msg, current_config))
```

デーモンを再起動しなくても、LINE のトークンや channel_secret を入れ替えられる。miniPC を多数の客先に配るシナリオでは、初期設定後に認証情報だけ差し替えることが多い。この設計なら再起動なしで対応できる。

### 4. OpenClaw chatCompletions API との接続

LINE のメッセージを受け取ったら、ローカルの OpenClaw API を叩く：

```python
async def call_openclaw(message: str, openclaw_url: str, gateway_token: str = "") -> str:
    payload = {
        "model": "gpt-4",
        "messages": [{"role": "user", "content": message}]
    }
    headers = {"Content-Type": "application/json"}
    if gateway_token:
        headers["Authorization"] = f"Bearer {gateway_token}"
    async with httpx.AsyncClient(timeout=60) as client:
        resp = await client.post(openclaw_url, json=payload, headers=headers)
        resp.raise_for_status()
        return resp.json()["choices"][0]["message"]["content"]
```

OpenAI 互換の chatCompletions 形式なので、OpenClaw だけでなく Ollama や LM Studio にも向けられる。URL を変えるだけだ。

---

## 配布を前提にした設計

この `line_bridge.py` は TechsFree が販売する miniPC に同梱するために作った。各ユニットに一意の `relay_token` が割り当てられ、中継サーバー側はそのトークンで宛先を識別する。

```json
{
  "relay_token": "gmk-xxxxxxxx",
  "channel_secret": "...",
  "channel_access_token": "...",
  "relay_url": "wss://linebot.techsfree.com/gmk/{token}/ws",
  "openclaw_url": "http://localhost:18789/v1/chat/completions"
}
```

設定ファイルを JSON 一枚にまとめたのは、購入者が一覧しやすくするためだ。GUI で設定項目を埋めて保存するだけで動く想定。

---

## まとめ

| 比較項目 | ngrok | cloudflared | WebSocket 中継 |
|--------|-------|-------------|----------------|
| URL の固定 | 無料では不可 | 可 | 可（中継サーバー次第）|
| デーモン設定 | 必要 | 必要 | `line_bridge.py` のみ |
| オフライン時の挙動 | URL無効 | トンネル切断 | 自動再接続 |
| カスタム中継 | 不可 | 不可 | 自社管理可 |
| 複数ユニット管理 | 難 | 難 | トークンで区別 |

cloudflared は強力だが「外部サービスへの依存」という点が気になっていた。自前の中継サーバーを持つと、認証・ログ・ルーティング全部が自分のコントロール下に入る。ホームラボや miniPC 商品のように「多数のローカル環境」を一元管理したいシナリオには、この方式が合っている。

