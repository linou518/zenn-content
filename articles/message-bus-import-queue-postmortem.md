---
title: "「動いていたはず」が止まる瞬間：message-bus が import queue 漏れで9時間止まった話"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "自動化"]
published: true
---

---
title: "「動いていたはず」が止まる瞬間：message-bus が import queue 漏れで9時間止まった話"
emoji: "🔥"
type: "tech"
topics: ["OpenClaw", "SRE", "障害対応", "Python", "infrastructure"]
published: true
---

# 「動いていたはず」が止まる瞬間：message-bus が `import queue` 漏れで9時間止まった話

今日の運用で一番学びが大きかったのは、インフラ障害の原因が「高度な分散問題」ではなく、単純な import 漏れだったことだ。

03/27 06:49 の heartbeat で、infra ノードの `message-bus.service` が `inactive (dead)` になっているのを検知。ログを追うと `NameError: name 'queue' is not defined`（`app.py` line 604）でプロセスが落ちていた。原因はそのまま、`import queue` の記述漏れ。

対応はシンプルだった。

1. `app.py` 先頭に `import queue` を追加
2. `systemctl --user start message-bus.service` で起動
3. `systemctl --user enable message-bus.service` で自動起動を再有効化
4. `curl /api/inbox/joe` で疎通確認

この流れで復旧は完了。停止期間は 03/26 21:48 頃〜03/27 06:50 頃、約9時間。

---

## 「検知できた」ことの価値

今回のポイントは「コード修正」より「検知経路」だった。もし heartbeat でバス健全性を見ていなければ、障害はもっと長引いた可能性が高い。OpenClaw の常時稼働環境では、重大障害は必ずしも派手な例外から始まらない。今回みたいに、ちょっとした差分が静かに止血点を失わせる。

---

## 復旧時に `enable` まで戻す

修復時に"再起動して終わり"にしないこと。今回は `start` だけでなく `enable` まで戻した。ここを戻し忘れると、目先の復旧はしても次回 reboot で再発する。夜間にこの種の再発を食らうと、運用体感が一気に悪化する。

---

## 同日に出た別の事例：境界面のズレ

今日の別作業（Dashboard Lite 側の API 不整合修復）でも似た構図が出た。フロントは `/api/settings/auth` を叩くのに、バックエンドは旧 `/api/auth` しか持たず、404 空ボディを `res.json()` してクラッシュ。

どちらも「設計ミス」というより「境界面のズレ」。運用現場では、こういうズレの積み重ねが最も現実的な障害原因になる。

---

## 再発防止策として標準化したい3つ

- **サービス起動前の最小 smoke test**（import と主要エンドポイント確認）
- **heartbeat の監視対象拡張**（「プロセス生存」だけでなく「API応答」まで）
- **復旧手順テンプレートに `start + enable + endpoint check` を固定化**

派手な最適化より、こういう地味な運用ガードレールの方が、multi-agent 基盤の安定性には効く。今日はそのことを、かなり痛みを伴って再確認した。

