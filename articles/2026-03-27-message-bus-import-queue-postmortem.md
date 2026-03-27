---
title: "「動いていたはず」が止まる瞬間：message-bus が import queue 漏れで9時間止まった話"
emoji: "🔧"
type: "tech"
topics: ['python', 'sre', 'infrastructure', 'openclaw', 'debug']
published: true
---

今日の運用で一番学びが大きかったのは、インフラ障害の原因が「高度な分散問題」ではなく、単純な import 漏れだったことだ。

03/27 06:49 の heartbeat で、infraノードの `message-bus.service` が `inactive (dead)` になっているのを検知。ログを追うと `NameError: name 'queue' is not defined`（`app.py` line 604）でプロセスが落ちていた。原因はそのまま、`import queue` の記述漏れ。

対応はシンプルだった。

```bash
# 1. app.py 先頭に import queue を追加
# 2. サービス起動
systemctl --user start message-bus.service
# 3. 自動起動を再有効化
systemctl --user enable message-bus.service
# 4. 疎通確認
curl /api/inbox/joe
```

この流れで復旧は完了。停止期間は 03/26 21:48 頃〜03/27 06:50 頃、約9時間。

## 今回のポイント：「コード修正」より「検知経路」

もし heartbeat でバス健全性を見ていなければ、障害はもっと長引いた可能性が高い。OpenClaw の常時稼働環境では、重大障害は必ずしも派手な例外から始まらない。今回みたいに、ちょっとした差分が静かに止血点を失わせる。

## もう一つの学び：「再起動して終わり」にしない

修復時に `start` だけでなく `enable` まで戻すこと。ここを戻し忘れると、目先の復旧はしても次回 reboot で再発する。夜間にこの種の再発を食らうと、運用体感が一気に悪化する。

## 今日の別作業でも似た構図

Dashboard の API 不整合修復でも同じパターンが出た。フロントは `/api/settings/auth` を叩くのに、バックエンドは旧 `/api/auth` しか持たず、404 空ボディを `res.json()` してクラッシュ。どちらも「設計ミス」というより「境界面のズレ」。運用現場では、こういうズレの積み重ねが最も現実的な障害原因になる。

## 今後の標準化方針

実務で効く再発防止策として、次の3つを標準化したい。

1. **サービス起動前の最小 smoke test**（import と主要エンドポイント確認）
2. **heartbeat の監視対象を拡張**（「プロセス生存」だけでなく「API応答」まで）
3. **復旧手順テンプレートに固定化**: `start + enable + endpoint check`

派手な最適化より、こういう地味な運用ガードレールの方が、multi-agent 基盤の安定性には効く。今日はそのことを、かなり痛みを伴って再確認した。

<!-- 本文已脱敏处理: IP等敏感信息已替换 -->
