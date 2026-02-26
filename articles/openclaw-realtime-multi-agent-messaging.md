---
title: "OpenClawマルチエージェント環境でリアルタイムメッセージングを実現した話"
emoji: "⚡"
type: "tech"
topics: ["OpenClaw", "Python", "SSE", "MultiAgent", "systemd"]
published: true
---

## TL;DR

9台のミニPC上で19個のAIエージェントを運用している環境で、エージェント間メッセージの配信遅延を**最大30分→約1秒**に短縮した。SSE（Server-Sent Events）とsystemdサービスを組み合わせた軽量なブリッジアーキテクチャの実装記録。

## 背景: なぜ遅延が問題だったか

OpenClawのエージェントは基本的にイベント駆動で動く。ユーザーからのメッセージには即座に反応するが、**エージェント同士の連携**となると話が変わる。

我々の環境では自前の消息総線（メッセージバス）を運用している。Flask + Gunicornで構築したシンプルなHTTP APIで、エージェントがメッセージを投函し、宛先エージェントが取りに来る仕組みだ。

問題は「取りに来る」タイミング。OpenClawの `cron.wake` でheartbeat（定期ポーリング）を使っていたが、heartbeatの間隔は最大30分。つまり：

- エージェントAがメッセージを投函 → 0秒
- エージェントBの次回heartbeat → **最大30分後**
- Bがメッセージを発見して処理 → さらに数秒

リアルタイム協調が必要な場面では致命的だった。

## 最初の試み: sessions_send（失敗）

OpenClawには `sessions_send` というAPIがある。別セッションに直接メッセージを注入できる機能だ。

```
sessions_send(sessionKey="agent:some-agent:main", message="新しいタスクです")
```

これなら即座に届く！…と思ったが、落とし穴があった。

**`sessions_send` は main/webchat セッションにしか注入できない。** Telegramセッション（我々のエージェントの主要チャネル）には届かない。

つまり、メッセージは届いているのにエージェントが気づかない。ボツ。

## 解決策: SSE + bus-watcher ブリッジ

発想を変えた。メッセージバス側にSSEエンドポイントを追加し、各ノードで監視プロセスを走らせる構成にした。

### アーキテクチャ

```
[Agent A] → POST /api/send → [消息総線] → SSE /api/stream
                                                    ↓
                                            [bus-watcher.py]
                                                    ↓
                                          cron.wake(mode=now)
                                                    ↓
                                              [Agent B 起動]
                                                    ↓
                                         heartbeat → GET /api/inbox
                                                    ↓
                                            [メッセージ処理]
```

### 消息総線側: SSE endpoint追加

Flask側に `/api/stream` を追加。新着メッセージをリアルタイムで配信する。

```python
@app.route('/api/stream')
def stream():
    def generate():
        last_id = 0
        while True:
            # 新着メッセージをチェック
            new_msgs = get_messages_after(last_id)
            for msg in new_msgs:
                yield f"data: {json.dumps(msg)}\n\n"
                last_id = msg['id']
            time.sleep(1)
    return Response(generate(), mimetype='text/event-stream')
```

**ハマりポイント**: Gunicornのワーカー数。最初2ワーカーで起動していたが、SSEのsubscriberがワーカー間で分散してしまい、片方のワーカーに届いたメッセージがもう片方のsubscriberに届かない問題が発生。`gevent` ワーカー1プロセスに統一して解決。

### 各ノード: bus-watcher.py

```python
#!/usr/bin/env python3
"""SSE → cron.wake ブリッジ"""
import urllib.request, json, subprocess

def watch():
    url = "http://192.168.x.x:8091/api/stream"  # 内部メッセージバスサーバー
    req = urllib.request.Request(url)
    with urllib.request.urlopen(req) as resp:
        for line in resp:  # readline ベース
            line = line.decode().strip()
            if line.startswith("data:"):
                msg = json.loads(line[5:])
                if msg["to_agent"] in LOCAL_AGENTS:
                    # cron.wake で即座にエージェントを起こす
                    subprocess.run([
                        "openclaw", "cron", "wake",
                        msg["to_agent"], "--mode=now"
                    ])
```

**もう一つのハマりポイント**: 最初 `urllib` の `read()` を使っていたが、バッファリングされてSSEイベントがリアルタイムに届かない。`readline()` ベースに切り替えて解決。

### systemdサービス化

各ノードに `bus-watcher.service` をデプロイ。自動起動・自動再接続を保証。

```ini
[Unit]
Description=Message Bus Watcher
After=network.target

[Service]
ExecStart=/usr/bin/python3 /path/to/bus-watcher.py
Restart=always
RestartSec=5

[Install]
WantedBy=default.target
```

7ノード全てにデプロイし、19エージェント全員のテストを通過。

## 結果

| 指標 | Before | After |
|------|--------|-------|
| メッセージ配信遅延 | 最大30分 | ~1秒 |
| 追加インフラ | なし | SSE endpoint + 軽量watcher |
| CPU/メモリ増加 | - | ほぼゼロ |
| 依存追加 | - | なし（標準ライブラリのみ） |

テスト時、エージェントたちがテストメッセージに即座に自動返信してきた時は感動した。複数のエージェントが次々反応する様子は、まさに「生きたエージェントネットワーク」だった。

## 学んだこと

1. **sessions_sendの制限を知っておく**: OpenClawのセッション注入はチャネルを選ぶ。万能ではない。
2. **SSEは過小評価されている**: WebSocketより圧倒的にシンプルで、この用途には十分。
3. **Gunicorn + SSE = ワーカー数に注意**: gevent + 1ワーカーがSSEには最適。
4. **urllibのバッファリング**: ストリーミング用途では `readline()` を使え。
5. **cron.wakeのmode=now**: OpenClawの隠れた便利機能。heartbeatを待たずに即座にエージェントを起動できる。

## まとめ

大げさなメッセージキュー（Redis, RabbitMQ等）を導入せずとも、SSEと数十行のPythonスクリプトで実用的なリアルタイムメッセージングが実現できた。

マルチエージェント環境において「エージェント間の通信速度」はシステム全体の応答性を左右する。30分と1秒の差は、単なる数字以上にユーザー体験を変える。
