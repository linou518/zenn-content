---
title: "ERPシステムにAIエージェントを組み込む：OpenClaw Gateway × LINE Bot × 権限連携の実装記録"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "自動化"]
published: true
---

# ERPシステムにAIエージェントを組み込む：OpenClaw Gateway × LINE Bot × 権限連携の実装記録

## 背景

TechsFreeで構築しているERP/Shopシステムに、今日一気に3つの機能を実装した。

1. **Pi4 AIエージェントをチャットAPIに接続**
2. **LINE Bot を OpenClaw Gateway 経由で稼働**
3. **LINE ユーザーと ERPアカウントの権限バインドシステム**

それぞれ「つながった瞬間」の気持ちよさがあったので、実装のポイントと詰まった部分をまとめておく。

---

## 1. AIエージェントをチャットAPIに繋ぐ：消息总線 vs HTTP直接呼び出し

最初、ERP/Shopのチャット機能をAIエージェントに接続するとき、社内のメッセージバス（非同期キュー方式）を使おうとした。しかしリアルタイムチャットには根本的に向かないと気づいた。

**メッセージバス方式の問題点：**
- 送信 → エージェント処理 → レスポンスをポーリングで取得
- 待ち時間30秒以上になることがある
- チャットUIでこれは使えない

**解決策：OpenAI互換HTTPエンドポイント**

Pi4 gateway（OpenClaw）がOpenAI互換のAPIを提供していることを確認し、直接HTTP呼び出しに切り替えた。

```typescript
// chatController.ts（要点抜粋）
const AGENT_GATEWAYS = {
  pi4: {
    url: process.env.PI4_GATEWAY_URL,
    token: process.env.PI4_GATEWAY_TOKEN,
    model: 'openclaw:pi4',
  }
};

const response = await fetch(`${gateway.url}/v1/chat/completions`, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${gateway.token}`,
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({
    model: gateway.model,
    messages: [{ role: 'user', content: userMessage }],
    max_tokens: 2000,
  }),
  signal: AbortSignal.timeout(30000),
});
```

これで `{"reply": "こんにちは！私は **Pi4**...", "source": "agent"}` のレスポンスが返ってくるようになった。シンプルで速い。

**教訓：** AIエージェントとの通信方式は「非同期通知」か「同期API」かを最初に決めること。用途によって全く違う設計が必要になる。

---

## 2. LINE Bot × OpenClaw：2つの接続モード

LINE Botの接続方式は大きく2つある。

### モード1：OpenClaw ネイティブ LINE Channel

```json
// openclaw.json（Pi4 gateway設定）
{
  "channels": {
    "line": {
      "channelAccessToken": "xxx",
      "channelSecret": "yyy",
      "enabled": true
    }
  },
  "bindings": [
    { "channel": "line", "agent": "pi4" }
  ]
}
```

nginx の設定：
```nginx
location /pi4/webhook {
  proxy_pass http://localhost:18789/line/webhook;
}
```

OpenClaw が LINE webhook を受け取り、そのまま agent session として扱う。LINE メッセージが Slack DM と同じ感覚で agent に届く。**汎用AIエージェントとして使いたい場合**に向いている。

### モード2：独立 Express サーバー（冰箱管家方式）

```javascript
// Express + @line/bot-sdk で独立実装
app.post('/webhook', middleware(lineConfig), (req, res) => {
  const events = req.body.events;
  // 独自ロジック処理
});
```

PM2でデーモン化、nginx で `/fridge/webhook` にルーティング。**専用機能bot（食材管理、タスク管理など）** に向いている。

今日は両方を本番稼働させた。用途によって使い分けるのが正解だと実感した。

---

## 3. LINE × ERP 権限バインドシステム

LINE Bot から ERP の情報を取得するとき、「誰がLINEを叩いているのか」を ERP のユーザー管理と紐付ける必要があった。

### データベース変更

```sql
ALTER TABLE users 
ADD COLUMN line_user_id VARCHAR(64) UNIQUE NULL,
ADD COLUMN line_display_name VARCHAR(255) NULL;
```

### 権限グループ設計

| ロール | 許可操作 |
|--------|---------|
| admin | 全機能（売上/仕入/在庫/財務/スタッフ管理） |
| staff（user） | 在庫/商品/仕入/売上/出荷 |
| 未バインド | バインド案内のみ |

### バインドAPI

```typescript
// POST /api/line/bind
// 手機番号でERPアカウントにLINEユーザーを紐付け
const user = await User.findOne({ where: { phone } });
if (!user) return res.status(404).json({ error: 'User not found' });

await user.update({
  line_user_id: lineUserId,
  line_display_name: displayName,
});

return res.json({ 
  success: true,
  permissions: ROLE_PERMISSIONS[user.role],
});
```

LINE Bot 側は `GET /api/line/user/:line_user_id` で権限を確認し、それに応じた応答を返す。

---

## 今日の詰まりポイント

**1. `chatCompletions` エンドポイントがデフォルト無効**  
Pi4 gateway では `/v1/chat/completions` がデフォルトで無効設定だった。`gateway.json` の `features.chatCompletions: true` を追加して再起動で解決。エラーが「APIエンドポイント不明」ではなく「SPA HTMLが返ってくる」という形だったので気づくのに時間がかかった。

**2. LINE channelAccessToken の大文字/小文字問題**  
401エラーが続き、トークン自体の問題かと思ったが、コピペ時に1文字目が `E` → `e` に変わっていた。LINE Developer Console で新規発行して解決。トークンの検証スクリプトを作っておけばよかった。

**3. ルーターのポートフォワーディング未設定**  
`linebot.techsfree.com` へのHTTPSアクセスが502になる原因が、nginx でも Pi4 でもなく、外部IP:443 がルーターでブロックされていただけ。外からcurlして確認する手順を最初にやれば30分は節約できた。

---

## まとめ

ERPシステムにAIを組み込む構成が今日で一通り動くようになった。

- **チャット機能** → Pi4 OpenAI互換API（同期HTTP）
- **LINE通知/対話** → OpenClaw LINE Channel（ネイティブ）
- **権限管理** → LINEユーザー × ERPロール の手機番号バインド

次は Pi4 エージェントが `GET /api/sales/recent` などを自律的に叩いて、LINEから「今週の売上は？」と聞けるようにしたい。Tool useの実装が次のステップ。

---

**タグ案:** `ERP` `LINE-Bot` `OpenClaw` `AI-Agent` `TypeScript` `Sequelize` `TechsFree` `実装メモ`

