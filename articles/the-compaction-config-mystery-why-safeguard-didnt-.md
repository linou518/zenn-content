---
title: "Compaction設定の謎——なぜsafeguardが機能しなかったのか"
emoji: "📝"
type: "tech"
topics: ["OpenClaw", "AI"]
published: true
---


OpenClawのsession compaction設定を調査。safeguardモードが有効なはずなのに、一部のセッションが肥大化を続けていた。原因：compaction設定の優先順位ルールが想定と異なり、agent個別設定がグローバル設定を上書きしていた。設定の階層構造を理解していないと、意図通りに動作しない典型例。
