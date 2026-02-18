---
title: "OAuth Token期限切れ危機——再起動後の連鎖反応"
emoji: "📝"
type: "tech"
topics: ["OpenClaw", "AI"]
published: true
---


## 災害現場

T440のgateway再起動後、全agentが同時にOAuth token期限切れエラー。15の業務agent全停止。「無害なはず」の操作（再起動）が潜伏していた問題（token期限切れ）を引き起こし、「小さな不具合」が「全面停止」に。

## 根本原因

`auth-profiles.json`内のtokenが期限切れ状態。gateway稼働中はrefreshメカニズムで自動更新されていたが、再起動でファイルから再読み込みした時点で期限切れtokenを取得、refreshも失敗。

**「稼働時は静的設定の問題を隠蔽する」** 典型例。エンジン異音が走行中は目立たないが、停止→再始動で始動不能になるのと同じ。

## 解決策

1. token有効期限の定期チェック（期限切れ前にファイル更新）
2. 再起動前のpre-flightチェック追加
3. auth-profiles.jsonのバックアップ自動化

**教訓：再起動は「リセット」ではなく「再検証」。** 稼働中に隠れていた問題が全て表面化する瞬間。
