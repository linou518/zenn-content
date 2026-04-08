---
title: "Mattermostでagentが「無反応」に見えた本当の原因は `thread_replies_disabled` だった"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "自動化"]
published: true
---

---
title: Mattermostでagentが「無反応」に見えた本当の原因は `thread_replies_disabled` だった
category: infra
tags:
  - OpenClaw
  - Mattermost
  - Multi-Agent
  - LLMOps
  - Troubleshooting
---

# Mattermostでagentが「無反応」に見えた本当の原因は `thread_replies_disabled` だった

OpenClaw を Mattermost 上で運用しているとき、複数ノードの agent が DM に返事していないように見える障害に遭遇しました。最初は接続断、モデル障害、認証切れを疑いましたが、実際の根因はもっと厄介でした。

結論から言うと、Mattermost 側では thread reply が無効になっていた一方で、agent の最終出力に `[[reply_to_current]]` が混ざり、OpenClaw が thread reply として投稿しようとして 400 で落ちていました。ユーザーから見ると「agent が黙っている」ように見えますが、内部では送信エラーが起きていた、という構図です。

## 何が起きていたのか

ログには次のようなエラーが出ていました。

```text
thread_replies_disabled: replying to threads is disabled on this server
```

この時点で重要なのは、問題が「LLMが返答を作れていない」ことではなく、「返答は作れているが Mattermost への投稿方式がサーバー設定と衝突している」ことです。

さらにややこしかったのは、`channels.mattermost.replyToMode` を `off` にしても、すぐには直らないケースがあったことです。原因は session 側の癖でした。agent が過去文脈を引きずり、本文に `[[reply_to_current]]` を自分で出してしまっていたため、設定だけ直しても再発していました。

## 今回の修復で効いた4つの対応

今回の復旧は、次の4点をセットでやって安定しました。

1. **各 agent の指示を修正**  
   Mattermost では `[[reply_to_current]]` や `[[reply_to:<id>]]` を絶対に出さない、と明文化しました。

2. **問題のある session をリセット**  
   バックアップ後に該当 session を削除し、返信タグを吐く癖をリセットしました。

3. **heartbeat model を統一**  
   一部に残っていた古い model 設定を `openai-codex/gpt-5.4` に揃えました。

4. **fallback 用の認証参照を修復**  
   `auth-profiles.json` の symlink 欠落を補い、fallback 時の別障害を防ぎました。

## 学び

この障害で分かったのは、「返事が来ない」=「接続障害」とは限らない、ということです。実際には次の3層が絡んでいました。

- **Mattermost のサーバー制約**
- **agent の出力癖**
- **session に残った過去の振る舞い**

見た目の現象は同じでも、どこが壊れているかで直し方は変わります。今回のケースでは、設定1個を直すだけでは不十分で、**ルール・session・model・auth をまとめて整える**必要がありました。

Mattermost を OpenClaw の実運用に使うなら、今後は `thread_replies_disabled` を最優先の確認項目に入れるべきです。無反応に見える障害ほど、送信経路のログを先に疑った方が早いです。

<!-- 本文已脱敏处理: IP/密码/Token等敏感信息已替换 -->

