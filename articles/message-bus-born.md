---
title: "Agentメッセージバスの誕生：16のAI Agentの通信インフラ"
emoji: "📨"
type: "tech"
topics: ["OpenClaw", "Flask", "AI", "メッセージバス"]
published: true
---


AI Agentが3個から16個に増えると、通信問題は「後回し」にできなくなります。

## なぜメッセージバスが必要か

OpenClawの内蔵agentToAgent通信には致命的制限：**同一Gatewayインスタンス内でしか通信できない**。main agentが別マシンのlearning agentにメッセージを送ると即エラー。

Telegramグループでの中継やRedis Pub/Subも検討しましたが、前者はフォーマット制御不可、後者はオーバースペック。

最終的に軽量メッセージバスを自作。要件：HTTP API、マルチAgent登録、メッセージ永続化、十分にシンプル。

## 技術選定：Flask + SQLite

FlaskはPython生態系で最も慣れている軽量フレームワーク。SQLiteは追加サービス不要、単一ファイル、バックアップ簡単。T440（192.168.x.x:8091）にデプロイ。

## API設計

6つのエンドポイント：
- **POST /send** — メッセージ送信（宛先指定またはブロードキャスト）
- **GET /inbox** — 受信箱（未読フィルター対応）
- **GET /history** — 履歴（時間範囲クエリ対応）
- **POST /ack** — 既読マーク
- **GET /agents** — 登録Agent一覧
- **GET /stats** — システム統計

## 主要機能

**ブロードキャスト**：`to`フィールド未指定で全登録Agentに配信。**返信チェーン**：`reply_to`で会話コンテキストを追跡。**優先度**：normal/high/urgentの3段階。**既読通知**：ackで処理確認。

## 踏んだ罠

- **SQLite並列処理**：`journal_mode=WAL`で解決
- **メッセージ蓄積**：7日超の既読メッセージを自動アーカイブ
- **心拍検知**：5分間活動なしでofflineマーク、オンライン復帰時に未読を自動プッシュ

## 所感

**AI Agentの価値は単体の強さではなく、効率的な協働にある。** Flask + SQLiteは「原始的」に見えますが、内部ネットワークで十数個のAgentには十分。問題を解決する最小のソリューションが最良のソリューションです。
