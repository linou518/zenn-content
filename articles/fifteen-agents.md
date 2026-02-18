---
title: "15のAgentを1つのネイティブインスタンスに統一"
emoji: "🏠"
type: "tech"
topics: ["OpenClaw", "AI", "\u30de\u30eb\u30c1Agent", "\u30a2\u30fc\u30ad\u30c6\u30af\u30c1\u30e3"]
published: true
---


## 1つのインスタンス、15の魂

Docker→ネイティブ移行完了後、T440のOpenClawネイティブインスタンスが15のagentを搭載。学習、健康、プロジェクト管理、投資、生活など多領域をカバー。

## 9つの正常稼働Agent

learning、health、docomo-pj、real-estate、investment、nobdata-pj、royal-pj、life、flect-pjが正常応答中。各agentは独立のTelegram Bot Token、system promptを持ちながら、同一のOpenClaw Gatewayプロセスを共有。**1つのエンジン、複数の人格**。

## 3つの401問題

残り3つのagentが401 Unauthorized。Token失効、設定ミス、BotFather設定変更のいずれか。BotFatherでの確認が必要。

## アーキテクチャの変貌

**以前（Docker）：** 5コンテナ × 独立OpenClawプロセス = 5倍のプロセスオーバーヘッド、5つの設定セット

**現在（ネイティブ）：** SystemDサービス → 1つのGateway → 15 agents。1プロセス、1設定、1ログ。

## SystemDによる管理

```bash
systemctl status openclaw-gateway
systemctl restart openclaw-gateway
journalctl -u openclaw-gateway -f
```

Dockerの抽象層なし、直接プロセス管理。起動高速、オーバーヘッド最小。

## リソース活用

T440（20コアXeon、62GB RAM）で15 agentは余裕。以前5つのGatewayプロセスが常駐していたのが1つに——メモリ使用量80%削減。

## まとめ

15 agent、1インスタンス、1プロセス。アーキテクチャは複雑であるほど良いのではなく、**ちょうど良いのが最良**。シンプルなインフラ、豊かなアプリケーション——それが目指す方向です。
