---
title: "OpenClawにLINEを繋ぐ——公式プラグインが動かなかったので独自Bridgeを作った話"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "自動化"]
published: true
---

# OpenClawにLINEを繋ぐ——公式プラグインが動かなかったので独自Bridgeを作った話

公式プラグインが使えなかった。だから作った。それだけの話だが、詰まったポイントが多かったので記録しておく。

## 背景

GMK AI Mini PCにOpenClawをインストールして販売する準備を進めている。日本ユーザー向けにTelegram・Slack・LINEの3チャネルで動作することが要件だった。TelegramとSlackは素直に動いたが、LINEだけハマった。

## 公式LINEプラグイン方式の失敗

OpenClawには公式のLINEプラグインがある（Pi4での検証時にも使おうとした）。今回も最初これを試した。

結果から言うと、`bindings` 設定が正しくなくGatewayがクラッシュする、`accounts.default` が必須なのにドキュメントが薄い、などの問題で断念。openclaw.jsonの設定を何度変えても再起動のたびにcrash loopになった。

時間をかけて公式プラグインと戦うより、**自分で作った方が早い**と判断した。

## 独自LINE Bridge方式

採用したアーキテクチャはシンプル：

```
LINEサーバー → cloudflared tunnel → line_bridge.py (Flask) → OpenClaw chatCompletions API → AI返信
```

### line_bridge.py のポイント

LINEのWebhookはPOSTで受け取り、ユーザーのメッセージをOpenClawのchatCompletions APIに投げて、返ってきた回答をLINE Reply APIで返す。

```python
@app.route("/webhook", methods=["POST"])
def webhook():
    body = request.get_json()
    for event in body.get("events", []):
        if event["type"] == "message":
            user_msg = event["message"]["text"]
            reply_token = event["replyToken"]
            
            # OpenClaw chatCompletions API呼び出し
            ai_resp = call_openclaw(user_msg)
            
            # LINE Reply
            line_reply(reply_token, ai_resp)
    return "OK"
```

OpenClawのchatCompletions APIはデフォルトで無効になっている。openclaw.jsonに以下を追加しないと動かない：

```json
"gateway": {
  "http": {
    "endpoints": {
      "chatCompletions": {
        "enabled": true
      }
    }
  }
}
```

**ハマりポイント**: `gateway.chatCompletions: true` という書き方をすると「Unrecognized key」でGatewayがクラッシュする。正しいパスは `gateway.http.endpoints.chatCompletions.enabled`。

### cloudflared でWebhook受信

LINE BotのWebhookにはhttpsのURLが必要。自宅サーバーでポートフォワーディングなしにこれを実現するにはcloudflaredが便利：

```bash
cloudflared tunnel --url http://localhost:3900
```

起動するとランダムなURLが発行される（例: `https://substance-deeper-pharmacy-councils.trycloudflare.com`）。これをLINE DevelopersコンソールのWebhook URLに設定すれば完成。

### systemdサービス化

line_bridge.pyをsystemd user serviceとして常駐させる：

```ini
[Unit]
Description=LINE Bridge for OpenClaw

[Service]
ExecStart=/usr/bin/python3 /home/openclaw/line_bridge.py
Restart=always

[Install]
WantedBy=default.target
```

## 動作確認結果

LINE → line_bridge → OpenClaw → AI返信のフルフローが通った。Webhook Verifyも成功。日本語でのやり取りも問題なし。

## 今日の教訓

1. **OpenClaw公式プラグインが使えない場合、chatCompletions APIで自前Bridgeを作るのが現実的**
2. **chatCompletions APIの有効化パスを間違えるとGatewayがcrashする**（`gateway.http.endpoints.chatCompletions.enabled`）
3. **cloudflaredは自宅サーバーのhttps公開に最速の選択肢**（証明書もポートフォワーディングも不要）
4. **streaming: partial はLINEで使うと1文字ずつ届いて体験が最悪**。`off` にすること

公式の道が詰まったとき、APIレベルで繋ぎ直す——これが一番早いことが多い。

---

**タグ案**: `OpenClaw`, `LINE Bot`, `cloudflared`, `Python`, `Flask`, `自宅サーバー`, `AI Agent`, `GMK Mini PC`

