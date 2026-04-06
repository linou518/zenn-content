---
title: "Mattermostでagentが無言に見えた本当の原因は replyToMode ではなく本文の reply tag だった"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "自動化"]
published: true
---

---
title: "Mattermostでagentが無言に見えた本当の原因は replyToMode ではなく本文の reply tag だった"
category: infra
tags:
  - OpenClaw
  - Mattermost
  - AI Agent
  - Troubleshooting
  - ChatOps
---

# Mattermostでagentが無言に見えた本当の原因は replyToMode ではなく本文の reply tag だった

Mattermost DM で agent が返事していないように見える現象を追っていたところ、主因は OpenAI 側の不調でも gateway の一時エラーでもなく、assistant の最終出力本文に混ざった `[[reply_to_current]]` だった。

この環境では Mattermost の thread replies が無効化されている。だから送信処理は 400 で弾かれるのに、利用者から見ると単に「agent が黙った」にしか見えない。ここが一番ややこしい。

## 何が罠だったのか

最初は `replyToMode: off` を入れているのだから thread reply にはならないはずだ、と考えていた。ところが実際には、設定で thread reply を切っていても、本文そのものに `[[reply_to_current]]` や `[[reply_to:<id>]]` が残っていれば送信側が thread と解釈してしまう。つまり設定値より、最終的に送られた payload のほうが強い。

このため、設定ファイルだけ確認して「off だから問題ない」と判断すると外す。実運用では、最終出力本文、チャネル制約、サーバーログを同時に見ないと原因に届かない。

## 症状

- ユーザー視点では agent が無反応に見える
- ログには `thread_replies_disabled` が出る
- 送信はクラッシュしていないが 400 で失敗している
- 別の auth や model のノイズも混ざり、原因の切り分けを難しくする

この手の障害は「無言」という症状に引っ張られると負ける。通信断やモデル障害を疑いたくなるが、実際には本文中の reply tag という地味な原因で詰まっていた。

## 実際に効いた対策

今回効いたのは次の4点だった。

1. AGENTS.md に「Mattermost では `[[reply_to_current]]` と `[[reply_to:<id>]]` を絶対に出さない」と明記する
2. 問題の session を退避して初期化し、古い会話文脈を捨てる
3. heartbeat や残存設定に残っていた古い model 指定を現行モデルへ統一する
4. 一部 agent にあった `auth-profiles.json` の symlink 欠落を補修する

ただし本丸は reply tag だった。auth や model の不整合も直したほうがよいが、それだけでは「無言」現象は止まらなかった。

## 運用上の教訓

Mattermost 連携で「たまに返事が消える」「特定環境だけ黙る」が起きたら、まず次を確認したほうが早い。

- サーバーログに `thread_replies_disabled` があるか
- assistant の最終出力本文に reply tag が混ざっていないか
- 設定だけでなく、実際の送信 payload がどうなっているか

設定は正しく見えても、本文が thread を要求していれば自爆する。こういう事故ほど、UI 症状ではなく payload とログを見る基本が効く。

<!-- 本文已脱敏处理: IP/密码/Token等敏感信息已替换 -->

