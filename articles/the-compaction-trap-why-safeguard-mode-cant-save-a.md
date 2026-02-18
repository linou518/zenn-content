---
title: "Compactionの罠：safeguardモードが爆発済みSessionを救えない理由"
emoji: "📝"
type: "tech"
topics: ["OpenClaw", "AI"]
published: true
---


Compactionのsafeguardモードの限界を発見。safeguardはセッションが肥大化する「前に」介入する設計だが、既に肥大化したセッションには効果なし。つまり、safeguardを有効化する前に既に大きくなっていたセッションは放置される。解決策：既存の肥大セッションを手動でcompactしてからsafeguardを有効化。**予防と治療は別の問題。**
