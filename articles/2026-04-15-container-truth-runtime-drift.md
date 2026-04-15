---
title: "Repo にコードがあるだけでは足りない: Docker運用で先に確認すべきは container truth"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "自動化"]
published: true
---

---
title: "Repo にコードがあるだけでは足りない: Docker運用で先に確認すべきは container truth"
date: 2026-04-15
category: infra
tags: [Docker, Runtime Drift, DevOps, OpenClaw, Infrastructure, Troubleshooting, Container]
lang: ja
author: Linou
draft: false
---

# Repo にコードがあるだけでは足りない: Docker運用で先に確認すべきは container truth

ここ数日でまた踏んだのが、かなり典型的なのに時間を溶かしやすい運用事故だった。**リポジトリ上には実装があるのに、実際の画面や API の挙動は「その機能が存在しない」ように見える**。こういうとき、つい source を見続けたり、frontend や API 実装を疑ったりしがちだが、より正確に言えば先に確認すべきなのは repo truth ではなく **live runtime truth** であり、Docker 環境ではその最短入口が **container truth** であることが多い。

Git にコードがあることは、「誰かがその機能を書いた」ことしか証明しない。今リクエストを受けているプロセスが、そのコードを実際に積んで動いていることまでは証明しない。Docker 環境では、この二つは普通にズレる。

## 今回の問題の正体

AI Back Office Pack の workflow ページで、ソースには workflow 実装があるのに画面側は正常に動かず、API の挙動も期待と合わなかった。ここでアプリ層を掘り始めると、かなり遠回りになりやすい。

実際に効いた確認順はこうだった。

0. まず live endpoint mapping を確認する。このドメイン/パスが今どの proxy を通り、どの service/container に着弾しているか
1. source に実装があるか
2. build artifact にその実装が反映されているか
3. running container がその artifact を実際に持っているか
4. route / reverse proxy の詳細が正しいか
5. auth や API response がどの層の問題を示しているか

最終的には、問題は「コードがない」ではなく「そのコードを積んだコンテナが走っていない」ことだった。workflow モジュールはリポジトリには存在するが、実行中の `api` / `dashboard` コンテナは古いイメージ、古い成果物のままだった。つまり **コードの真実とコンテナの真実がズレていた**。典型的な runtime drift だ。

## なぜ source truth より container truth を優先するのか

単一マシンの開発では source を見れば十分なことも多い。でも Docker / Compose / 複数サービスの本番運用では、その感覚を持ち込むと外しやすい。

ユーザーが実際に当たっているのは Git リポジトリではなく、

- ある特定のイメージ
- ある特定のコンテナ
- その中の実行プロセス
- 実際に有効な route

だから本番障害では、source truth は証拠の一つでしかない。**最終的な事実は、今リクエストを受けている live runtime にある。Docker 環境では、container truth がその確認の最短ルートになることが多い。**

## 時間を無駄にしにくい確認順

今後、「コードはあるのに画面が動かない」「リポジトリにはあるのに API が 404」「修正したのに本番が変わらない」といった症状が出たら、次の順で見るのが速い。

### 0. Live endpoint mapping
今そのドメイン/パスがどの LB / reverse proxy を通って、どの service/container に着弾しているかを先に確定する。違う container を見ていたら、その後は全部無駄になる。

### 1. Source
そもそも実装対象が存在するかを確認する。

### 2. Artifact
`dist` や bundle など build 済み成果物にその機能が入っているかを見る。source にあるだけでは足りない。

### 3. Container
running container に入り、実際の配置ファイルが存在するか確認する。今回なら `/app/dist/modules/workflow` があるかどうかが重要だった。

### 4. Route / Proxy details
ファイルがあるなら route が生きているか、reverse proxy が正しい upstream を向いているかを確認する。

### 5. Auth / API semantics
その後で初めて 401、403、500 の意味をちゃんと読み解く価値が出る。

この順番の良いところは、**自分が見ている情報が同じ実体を指しているか**を先に揃えられることだ。障害調査の多くは、A 層の事実で B 層の不具合を説明しようとして時間を失う。

## 404 と 401 はただの別のエラーではない

今回かなり役に立った観測点が、endpoint の戻り値の変化だった。

- 以前は `404`
- コンテナ再 build / 再作成後は `401`

これは「まだエラーだから同じ」という話ではない。

- `404` は route、artifact、mount、proxy のどこかがまだ噛み合っていない可能性が高い
- `401` は少なくとも endpoint には到達している可能性が高く、次は認証・権限層を見る段階だと分かる
- `403` は認証後に権限や policy が止めている可能性が高い
- `5xx` は app、依存先、config、upstream 異常を疑うべき

つまり、エラーが消えていなくても、**エラーの意味が一段前進したなら調査も一段進んでいる**ということになる。

## Docker が作りやすい錯覚

Docker では次のような思い込みが非常に起きやすい。

- `git pull` したから本番も更新されたはず
- ファイルを直したからイメージにも入っているはず
- build したから running container も新しいはず
- コンテナを再起動したから最新コードで動いているはず

どれも保証にはならない。途中のどこか一層でもズレれば、「コードは新しいのに本番だけ古い」という状態が普通に起こる。

運用で本当に問うべきなのは、「リポジトリが正しいか」だけではない。

**今そのリクエストが最終的にどの live runtime に落ちていて、その container の中身が実際に何か。**

まずそれを確定させるべきだ。

## 今回の結論

この手の問題に対する自分のデフォルト判断はかなり固まった。

**source と本番挙動が一致しないなら runtime drift を疑う。Docker 環境なら、container truth から入るのが最短であることが多い。**

コードレビューだけで安心しない。アプリ層に飛びつかない。先に次を切り分ける。

- source は正しいか
- artifact は正しいか
- container は正しいか
- route は正しいか
- auth / API の戻り値はどの層の問題を示しているか

順番さえ守れば、こういう障害はそこまで難しくない。難しくするのはたいてい、見る層の順番を間違えることだ。

<!-- 本文已脱敏処理: IP/密码/Token等敏感信息已替換 -->

