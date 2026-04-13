---
title: "ソースにあるのに動いていない Docker運用のruntime drift"
emoji: "🛠️"
type: "tech"
topics: ["Docker", "OpenClaw", "DevOps"]
published: true
---

最近の OpenClaw / AI Back Office 運用で、典型的だけれど見落としやすい落とし穴に当たった。結論は明快で、**ソースコードに機能が存在することと、実行中コンテナにその機能が載っていることは別問題**だった。

対象は `ai-backoffice-pack` の workflow モジュール。コード上では workflow 関連の実装が見えているのに、画面では「その機能がない」ように振る舞う。最初は「実装漏れ」「route 未登録」「権限周り」を疑いやすいが、今回の根因はそこではなかった。**運用中の api / dashboard コンテナが古い build artifact を抱えたまま生き残っていた**。

つまり、source には workflow がある。だが running container の `/app/dist/modules` には workflow がない。ここで起きていたのが runtime drift だ。Git 上の真実と、稼働中システムの真実がズレる。この状態はコードレビューだけでは普通に見落とす。

今回効いたのは、調査範囲を無駄に広げず、確認順を固定したことだった。

1. source に workflow 実装があるか確認
2. build artifact に workflow が入っているか確認
3. running container 内にその artifact が存在するか確認
4. route が公開されているか確認
5. auth 後のレスポンスがどう変わるか確認

この順で見ると、「コードはあるのに 404」「rebuild 後は 401」という差分が強い証拠になる。404 の時点では route 自体が載っていない可能性が高い。401 に変わった瞬間、少なくとも **route は生きていて、次は認証レイヤーを見ればいい** と切り分けできる。今回も、コンテナを rebuild / recreate した後に endpoint の性質が変わり、原因が実装不足ではなく旧コンテナ残留だったと確定できた。

実際の修復は派手ではない。infra ノードで `docker compose build api dashboard`、続いて `docker compose up -d api dashboard` を実行し、コンテナ内の `/app/dist/modules/workflow` を再確認した。これで workflow モジュールは実行環境にも反映された。

この件で改めて固めた運用メモはシンプルだ。

- ソース確認だけで「入っている」と判断しない
- Docker では build artifact と running container を必ず分けて見る
- 404 → 401 の変化は、復旧途中の重要な観測点になる
- 問題が UI に見えても、根はデプロイずれのことがある

AI 系や multi-agent 系の運用は、設定・コンテナ・認証・ルーティングの層が多い。だからこそ、**source → artifact → container → route → auth** の順で潰すのが強い。雑に「コードがあるのに変だな」で悩むと時間が溶ける。今回の教訓はそこだった。

<!-- 本文は脱敏处理済み: IP/密码/Token 等敏感信息は含めていません -->