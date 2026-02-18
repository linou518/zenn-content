---
title: "1日で10個のTelegram Botを作成"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "Telegram", "Bot", "AI"]
published: true
---


「今日何をしたの？」と聞かれたら、「Telegram botを10個作った」と答えます。大げさに聞こえますが、本当に1日の出来事です。5個から10個への拡張プロセスと、マルチagentルーティングの設定で苦労した細部を記録します。

## 5から10へ

当初のOpenClawデプロイは5つのagentだけ。十分でしたが細分化が足りません。プロジェクトが増えると、一つのagentが複数の役割を担うとコンテキストが混乱します。そこで職責ごとに分割し、1 bot = 1つの明確な領域とすることに。

最終的な10個のbotリスト：

| Bot名 | 職責 | モデル |
|--------|------|--------|
| Joe (Main) | メイン管理者、総合調整 | Claude Opus |
| NobData | NobDataプロジェクト開発 | Claude Opus |
| Flect | Flectプロジェクト管理 | Claude Opus |
| Life | 生活アシスタント | Claude Opus |
| Learning | 学習ノート | DeepSeek |
| Real-estate | 不動産投資分析 | GPT-4o |
| Investment | 投資・資産運用 | GPT-4o |
| Health | 健康管理 | DeepSeek |
| Royal | Royalプロジェクト | Claude Opus |
| Customer-service | カスタマーサービス | Claude Opus |

## モデル割り当て戦略

すべてのbotに最強モデルが必要なわけではありません：

**プロジェクト系botにはOpus**：コード開発を伴うagentには強い推論能力と長いコンテキスト理解が必要。

**ツール系botにはDeepSeek/GPT-4o**：情報整理やQ&Aが主なagentにはコスパの良いモデルで十分。

10個すべてにOpusを使えばAPI費用が急騰します。実際のニーズに合わせたモデル配分で、重要なagentの能力を確保しつつ全体コストを管理。

## 作成プロセス

Telegram上でのbot作成自体は迅速——@BotFatherで`/newbot`、命名、token取得。1bot1分、10botで10分。

時間がかかるのはOpenClaw側の設定。各botに必要なもの：
1. gateway設定にaccount追加（Telegram token）
2. agent設定作成（モデル、system prompt、ツール権限）
3. binding設定（accountをagentに紐付け）
4. dmPolicy設定（誰がbotと対話できるか制御）

10botで40項目の設定。一つでもミスがあればbot無応答またはルーティングエラー。

## マルチAgentルーティングの落とし穴

### 落とし穴1：binding設定の漏れ

OpenClawのルーティングロジック：メッセージ受信 → accountからbinding検索 → bindingからagent検索。accountに対応するbindingがなければ、メッセージは破棄され、エラーログすら出ません。

**教訓**：botを追加するたびに、account・agent・bindingの3つの設定を同時に確認。欠けたら動きません。

### 落とし穴2：dmPolicy allowlist

OpenClawのデフォルトdmPolicyはすべてのDMを拒否。dmPolicyを設定し忘れると、自分のメッセージすら拒否されます。

さらに隠れた罠：allowlistはTelegramのuser ID（数字）で設定する必要があり、usernameではありません。

## 1日の成果

朝から計画し、夜に10個すべてが応答するまで丸一日：
- 30%は計画と設計
- 20%は作成と基本設定
- 50%はルーティング問題のデバッグ

作成自体は早く、大半の時間は「正しく動かす」ことに費やされました。すべてのシステム統合作業の共通点でしょう：接続は簡単、正しい接続は難しい。

## 所感

10個のbotはマイクロサービスの考え方と同じ：何でもやる巨大botより、各自の役割が明確な小さなbotを10個持つ方が良い。コンテキストがクリーンになり、回答がより正確に。

次に20個に拡張する時は、まず自動化スクリプトを書きます。繰り返しの手作業は2回目から自動化すべき——それが当日最大の教訓です。
