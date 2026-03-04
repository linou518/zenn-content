---
title: "メッセージバスのウェイク通知バグを直した話：全AgentをSlackへ統一"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "自動化"]
published: true
---

---
title: "メッセージバスのウェイク通知バグを直した話：全AgentをSlackへ統一"
emoji: "🔧"
type: "tech"
topics: ["openclaw", "multiagent", "slack", "python", "自動化"]
published: true
category: programming
tags: ["openclaw", "multiagent", "slack", "python", "automation"]
---

# メッセージバスのウェイク通知バグを直した話：全AgentをSlackへ統一

## TL;DR

- 16個のAI Agentが動くホームサーバー環境で、**メッセージバスのHTTP 500エラー**が頻発していた
- 根本原因: `wake_agent()` 関数がTelegram通知をハードコードしており、Slack使用Agentで失敗していた
- 修正: Agentのチャンネル情報をメッセージバスに登録し、Agent別に通知先を切り替えるよう変更
- 副産物として、登録漏れだった6つのAgentも発見・追加

---

## 背景：16 Agent、2メッセージチャンネル

自宅サーバークラスタには現在16個のAI Agentが動いている。Joe（総管理）、Jack（個人系サービス統括）をはじめ、各ドメイン専門のAgentたちだ。

Agent間の通信には**自作メッセージバス**（Python Flask、`http://192.168.x.x:8091`）を使っている。Agentは定期的にバスをポーリングして自分の受信箱をチェックし、必要に応じて返信する。

ただし問題があった。**AgentがSlackで動いているものとTelegramで動いているものが混在していた**のだ。

| Agentグループ | 通知チャンネル |
|---|---|
| joe, jack | Slack |
| pi4 | Slack |
| その他多数 | Telegram |

---

## バグの症状

朝のヘルスチェックで、バスへのメッセージ送信後にHTTP 500が返ってくることに気づいた。

```
POST /api/send
→ 500 Internal Server Error
→ ログ: "Telegram sendMessage failed: chat not found"
```

特定のAgentへメッセージを送ろうとすると確実に失敗する。しかもエラーがバス全体の処理を止めてしまっていた。

---

## 根本原因の特定

メッセージバスの `app.py` を調べると、`wake_agent()` という関数がこう書かれていた：

```python
def wake_agent(agent_id: str):
    """Agentに新着メッセージを通知してウェイクさせる"""
    telegram_chat_id = AGENT_CHAT_MAP.get(agent_id)
    if telegram_chat_id:
        send_telegram_message(telegram_chat_id, f"📬 新着メッセージ: {agent_id}")
```

**全AgentにTelegram通知を送っていた**。joe・jack・pi4のようにSlackで動いているAgentにもTelegramで通知しようとするため、必然的に失敗する。

さらに `AGENT_CHAT_MAP` を確認すると、最近追加されたはずのAgentが複数登録されていなかった：

```python
# 登録漏れ確認
missing_agents = ["pi4", "windows", "monitor", "shadow", "erp-chat-bridge", "learn-infra-05"]
```

---

## 修正内容

### 1. Agentレジストリにチャンネル情報を追加

```python
AGENT_REGISTRY = {
    "joe":             {"channel": "slack",    "slack_channel": "#joe"},
    "jack":            {"channel": "slack",    "slack_channel": "#jack"},
    "pi4":             {"channel": "slack",    "slack_channel": "#pi4"},
    "health":          {"channel": "telegram", "chat_id": "..."},
    "investment":      {"channel": "telegram", "chat_id": "..."},
    # ... 全16 Agent
}
```

### 2. wake_agent() をチャンネル別に分岐

```python
def wake_agent(agent_id: str):
    info = AGENT_REGISTRY.get(agent_id, {})
    channel = info.get("channel", "telegram")
    
    if channel == "slack":
        slack_channel = info.get("slack_channel", f"#{agent_id}")
        send_slack_message(slack_channel, f"📬 新着メッセージ: {agent_id}")
    else:
        chat_id = info.get("chat_id")
        if chat_id:
            send_telegram_message(chat_id, f"📬 新着メッセージ: {agent_id}")
```

### 3. 未登録Agentを追加

6つの登録漏れAgentを `AGENT_REGISTRY` に追加。pi4はSlack、残り5つはTelegram系として登録した。

---

## 修正後の確認

```bash
# バスに登録されているAgent一覧を確認
curl -s http://192.168.x.x:8091/api/agents \
  -H "X-Bus-Token: <TOKEN>" | jq '.agents[].id'

# "joe", "jack", "pi4" にメッセージ送信テスト
curl -X POST http://192.168.x.x:8091/api/send \
  -H "Content-Type: application/json" \
  -H "X-Bus-Token: <TOKEN>" \
  -d '{"from":"test","to":"jack","subject":"test","body":"wake test"}'
→ 200 OK ✅
```

HTTP 500エラーは完全に解消。Slack AgentにはSlack通知、Telegram AgentにはTelegram通知が届くようになった。

---

## 教訓

### 1. インフラコードに通知チャンネルをハードコードしない

Agentの通知チャンネルは「設定値」であって「実装」ではない。今回のように環境が変わったときにバグが出る典型例だ。

### 2. Agent追加時はバス登録をセットで行う

新しいAgentを立ち上げるとき、メッセージバスへの登録を忘れないようチェックリストに入れる必要がある。今回は6つも漏れていた。

### 3. エラーを黙って飲み込まない

`wake_agent()` のTelegram失敗がHTTP 500として外部に伝播していたのは問題だった。ウェイク通知の失敗はメッセージ配送の失敗とは別で、**ログだけ吐いて処理を続行するべきだった**。

---

## まとめ

マルチAgent環境はコンポーネントが増えるほど「設定の陳腐化」が起きやすい。今回のバグはその典型で、1行のハードコードが16 Agentシステム全体の通知を壊していた。

インフラの設定変更は **全コンポーネントへの影響確認** がセットで必要だと改めて実感した。

<!-- 本文已脱敏处理: IP/Token等敏感信息已替换 -->

