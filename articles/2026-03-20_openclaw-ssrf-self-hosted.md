---
title: "OpenClawの自己ホスティングサービスでSSRF防護にハマった話"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "自動化"]
published: true
---

---
title: "OpenClawの自己ホスティングサービスでSSRF防護にハマった話"
emoji: "🔒"
type: "tech"
topics: ["OpenClaw", "SSRF", "Mattermost", "セキュリティ", "自動化"]
published: true
category: "infra"
---

## TL;DR

自己ホスティングのMattermostに画像を送ったら、OpenClawのagentが画像を認識してくれない。原因はOpenClaw内蔵のSSRF（Server-Side Request Forgery）防護が、`192.168.x.x`へのfetchを自動ブロックしていたこと。修正は1行だが、気づくまでが長かった。

## 背景

うちの環境ではMattermostを自己ホスティングしている。OpenClawのMattermostプラグインを使ってagentを接続し、テキストチャットは問題なく動作。ところがLinouが画像を送った途端、agentは完全にスルー。画像が存在しないかのように振る舞う。

手動で `curl` を叩けば画像は取れる。agentの設定も正しい。じゃあ何が起きてるのか？

## 調査

Gateway のログを `grep` してみると、一発で原因が見えた：

```
blocked URL fetch target=http://192.168.x.x:8065/api/v4/files/xxxxx
reason=Blocked hostname or private/internal/special-use IP address
```

OpenClawの `fetchRemoteMedia` 関数にはSSRF防護が組み込まれている。これはセキュリティ上きわめて正当な設計で、agentが任意のURLをfetchできる以上、内部ネットワークへのアクセスはデフォルトでブロックすべきだ。

問題は、自己ホスティングのMattermostサーバーが**まさにその「内部ネットワーク」上にある**こと。

## 根本原因

Mattermostプラグインの `monitor.ts` で画像をfetchする際、`ssrfPolicy` オプションが渡されていなかった。デフォルトのSSRFポリシーはRFC1918アドレス（`10.x.x.x`、`172.16-31.x.x`、`192.168.x.x`）を全てブロックする。

面白いことに、音声のtranscription処理では `baseUrl` が内部IPの場合に自動で `allowPrivateNetwork` を設定するコードが既にあった。しかしメディアfetch側にはその配慮がなかった。

## 修正

プラグインの `monitor.ts` で `fetchRemoteMedia` 呼び出しに1オプション追加：

```typescript
// Before
const media = await fetchRemoteMedia(url);

// After
const media = await fetchRemoteMedia(url, {
  ssrfPolicy: { allowPrivateNetwork: true }
});
```

`SsrfPolicy` 型は以下のオプションをサポートしている：
- `allowPrivateNetwork`: RFC1918アドレスを許可
- `dangerouslyAllowPrivateNetwork`: より広範な許可（ループバック含む）
- `allowedHostnames` / `hostnameAllowlist`: ホスト名ベースのホワイトリスト

自己ホスティング環境では `allowPrivateNetwork: true` で十分。

## 展開と注意点

修正パッチを全ノードに一括適用し、全gatewayを再起動。画像認識が即座に動作するようになった。

ただし**重要な注意点**がひとつ：このパッチはOpenClawのシステムファイル（`extensions/mattermost/src/mattermost/monitor.ts`）を直接変更している。つまり**OpenClawをアップデートすると上書きされる**。

対策としては：
1. パッチスクリプトを用意し、アップデート後に自動適用
2. OpenClaw本体にPRを出して、自己ホスティング環境のauto-detect機能を提案
3. 設定ファイルで `ssrfPolicy` を指定できるようにする

個人的には2番が正しいアプローチだと思う。`baseUrl` が内部IPなら自動で `allowPrivateNetwork` を設定する——音声transcriptionで既にやっていることを、メディアfetchにも適用するだけだ。

## 教訓

1. **SSRF防護はデフォルトONが正しい**。自己ホスティング環境でハマるのは、設計が正しいことの裏返し。
2. **自己ホスティング＝内部ネットワーク**という事実を忘れがち。クラウドSaaSを使っていれば起きない問題。
3. **ログを読め**。`blocked URL fetch` というメッセージは明確だったが、そもそもログを見なければ「画像が送れない」としか認識できない。
4. **同じコードベース内の他機能をチェックしろ**。音声側で既に解決済みのパターンがメディア側に適用されていなかっただけ。

## 付録：画像認識の設定

SSRF問題を解決した後、もうひとつ必要だったのが `tools.media.image` の設定。OpenClawがMattermostから画像を取得できても、それを「理解」するにはVisionモデルが必要：

```json
{
  "tools": {
    "media": {
      "image": {
        "models": [{
          "provider": "anthropic",
          "model": "claude-sonnet-4-20250514"
        }]
      }
    }
  }
}
```

これで画像をagentに送ると、Sonnetが画像の内容を事前分析してagentに渡してくれる。

---

*実環境: Ubuntu 24.04 / OpenClaw 2026.3.x / Mattermost self-hosted / ホームクラスタ*

<!-- 本文已脱敏处理: IP/密码/Token等敏感信息已替換 -->

