---
title: "個人で10体のAI Agentを運用して生活を自動化した話"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "Agent", "自動化", "LLM"]
published: false
---

## はじめに

「AIエージェントって、結局チャットボットでしょ？」

半年前の自分もそう思っていました。でも今、自宅の PC 3台で **10体の AI Agent** が24時間稼働し、朝のブリーフィングから会議管理、投資モニタリング、子供の宿題サポートまで——生活のあらゆる場面を自動化しています。

こんにちは、Linou です。日本で IT エンジニアとして働いて10年以上、主にデータプラットフォーム領域にいます。この記事では、[OpenClaw](https://docs.openclaw.ai) というオープンソースの AI Agent フレームワークを使って構築した「個人マルチエージェントシステム」の全貌を、ハマりポイント込みで共有します。

:::message
この記事は「やってみた系」ですが、実運用で踏んだ地雷の話が本題です。公式ドキュメントに載っていないリアルな知見を中心に書きます。
:::

## なぜ10体も必要なのか

最初は1体（総管の Joe）だけでした。でも運用していくうちに、こんな問題が出てきます。

- **コンテキストの肥大化**: 投資の話と子供の宿題が同じセッションに混ざると、トークン消費が爆発する
- **プロンプトの専門化**: プロジェクト管理と技術ニュース収集では、求められるペルソナがまったく違う
- **障害の分離**: 1体が壊れても他に影響しない

結果として、**関心の分離（Separation of Concerns）** という設計原則そのままに、役割ごとに Agent を分けるのが最適解でした。

### Agent 一覧

| Agent | 役割 | 概要 |
|-------|------|------|
| **Joe（総管）** | 統括 | 全体調整、日程管理、heartbeat監視、障害対応 |
| **CaiZhi** | 投資管理 | ポートフォリオ監視、市場トレンド、月次レビュー |
| **学思** | 個人学習 | 学習計画、技術トレンド収集、毎日の技術ニュース |
| **学習助理** | 子供教育 | 宿題サポート、学習進捗管理 |
| **Life Helper** | 生活全般 | 買い物、予約、翻訳、生活情報 |
| **PJ-A〜D** | プロジェクト | 各案件の進捗・会議管理（4案件分） |
| **Fudosan** | 不動産 | 物件管理、不動産市場分析 |

プロジェクト系が4体あるのは、案件ごとにコンテキストとスケジュールが完全に独立しているからです。混ぜると「A社の会議」と「B社の朝会」の区別がつかなくなります。

## アーキテクチャ

全体構成はこんな感じです。

```
┌─────────────┐     Telegram Bot API      ┌──────────────────┐
│  Telegram    │  ◄──── 複数Bot Account ──► │  PC-A (メイン)    │
│  (スマホ)    │                            │  OpenClaw Gateway │
└─────────────┘                            │  Joe + 全Agent    │
                                           │  8GB RAM          │
                                           └────────┬─────────┘
                                                    │ メモリ同期(5分)
                                           ┌────────▼─────────┐
                                           │  PC-B (スタンバイ) │
                                           │  Joe-Standby      │
                                           │  自動フェイルオーバー│
                                           └──────────────────┘
                                           
                                           ┌──────────────────┐
                                           │  T440 (Docker)    │
                                           │  学習系Agent      │
                                           │  Flask Dashboard  │
                                           └──────────────────┘
```

### ポイント

- **通信**: Telegram Bot API を複数アカウントで運用。OpenClaw の binding routing で、各 Bot → 対応 Agent にルーティング
- **HA（高可用性）**: `watchdog.py` が30秒間隔で PC-A を監視。ダウン検知から約90秒で PC-B に自動切替
- **バックアップ**: GitHub private repo に GPG 暗号化して自動 push。Healthchecks.io で監視
- **コスト**: 自宅の既存 PC 3台を流用。追加ハードウェア投資はほぼゼロ

:::message alert
8GB RAM で10体の Agent を動かせるのは、OpenClaw が Agent 自体をローカルで推論するわけではなく、API コール + セッション管理 + ワークスペース管理がメインだからです。LLM の推論は Anthropic API 側で行われます。
:::

## 実践事例

### 事例①：朝の自動ブリーフィング

毎朝、Telegram で「おはよう」と一言送るだけで、こんなレポートが届きます。

```
🌅 おはようございます。2月12日(水)のブリーフィングです。

📋 昨日のサマリー
- CaiZhi: 日経平均 +1.2%、保有銘柄に大きな変動なし
- PJ-A: PRレビュー2件完了、Sprint進捗85%
- PJ-B: 明日のデモに向けてステージング環境デプロイ済み

📅 今日の予定
- 10:00 A社 定例（Teams）
- 14:00 B社 技術レビュー（Zoom）
- 16:00 1on1（上司）

📰 技術ニュース Top 3
- [1] OpenAI が新モデル発表...
- [2] Rust 2025 Edition の変更点...
- [3] AWS re:Invent 2025 まとめ...
```

**仕組み**: cron job で毎朝の処理をキック → `sessions_spawn` で各 Agent にレポート生成を依頼 → Joe が集約して Telegram に配信。

**効果**: 以前は朝の情報収集に **60分** かかっていたのが、**5分** で完了するようになりました。コーヒーを淹れている間に全部読めます。

### 事例②：会議リマインド＆重複検知

複数案件を掛け持ちしていると、会議の重複が地味に致命的です。

- 毎朝5時に **MS Graph API** でカレンダーを取得
- 重複する会議を自動検知して警告（例：10:00 A社定例 と 10:30 B社朝会が被っている）
- 各会議の **2時間前** にリマインド + Telegram のインラインボタン（開始 / 完了 / 延期）
- T440 上の Flask Dashboard でタイムライン表示

```
⚠️ 会議重複を検知しました！
  10:00-11:00 A社 定例
  10:30-11:30 B社 朝会
→ B社朝会の時間変更を提案しますか？ [はい] [いいえ] [無視]
```

インラインボタンで「はい」を押すと、該当プロジェクトの Agent が自動でリスケ候補を提示してくれます。

### 事例③：技術トレンド自動収集

学思 Agent が毎朝4:30に動きます。

1. **12のデータソース**（Hacker News, Reddit, arXiv, Zenn, Qiita, GitHub Trending など）から **1,000件以上** の記事を収集
2. AI でスコアリング（関連度 × 新規性 × 影響度）
3. **Top 10** を Telegram に配信
4. OpenClaw 関連の論文やセキュリティ情報は別枠で自動検知

起きたらもう読むべき記事が整理されている。これだけでも Agent を運用する価値があります。

## ハマりポイント 5選（ここが本題）

ここからが本当に伝えたい内容です。公式ドキュメントには載っていない、**実運用で踏んだ地雷** を共有します。

### 🔥 1. Session汚染

**症状**: 複数の Telegram Bot を同一 Gateway で動かすと、Agent A に送ったメッセージの返答が Agent B から返ってくる。

**原因**: 複数 Bot で `deliveryContext` が上書きされる問題。デフォルトの `dmPolicy: pairing` だと、最後に pairing した Bot が全セッションを奪ってしまう。

**解決策**:

```yaml
# dmPolicy を allowlist に変更
dmPolicy: allowlist

# 各 Telegram account に明示的な binding を設定
telegram:
  accounts:
    - name: joe-bot
      binding: agent:main
    - name: caizhi-bot
      binding: agent:caizhi
```

:::message alert
**教訓**: マルチ Bot 構成では `dmPolicy: allowlist` + 明示的 binding が必須。これを知らずに3日溶かしました。
:::

### 🔥 2. Config事故（最も深刻）

**症状**: Gateway が起動しない。PC-A も PC-B も両方とも。

**原因**: `streamMode` に無効値 `"full"` を設定してしまった。正しい値は `"off"` / `"partial"` / `"block"` のみ。config ファイルを直接 `sed` で編集したため、バリデーションが効かなかった。

**しかも HA構成の両系に同じ config を同期していたので、2台同時に死亡。** 復旧に1時間以上かかりました。

**教訓**:

```bash
# ❌ 絶対にやってはいけない
vim ~/.openclaw/config.yaml

# ✅ 必ず config.patch を使う（バリデーション付き）
openclaw config.patch '{"streamMode": "streaming"}'
```

`config.patch` はバリデーションを通してから適用するので、無効値は弾いてくれます。**直接ファイル編集は絶対NG**。これは鉄則です。

### 🔥 3. Bot Token 衝突

**症状**: Bot の応答が異常に遅い。同じメッセージに2回返答する。ログに `409 Conflict` が大量に出る。

**原因**: 同一の Bot Token で複数プロセスが Telegram の polling を行っていた。PC-A と PC-B で同じ Token を使って同時に起動してしまったケース。

**解決策**: **1 Token = 1 プロセス** の鉄則を守る。HA構成では、スタンバイ側は Active になるまで polling を開始しない設計にする。

### 🔥 4. Telegram Bot 404事故

**症状**: 全 Bot が突然応答しなくなる。Telegram 上で Bot にメッセージを送っても既読にすらならない。

**原因**: `config.patch` で Telegram accounts の設定を更新する際、うっかり `botToken` フィールドに placeholder 文字列を含めてしまった。結果、**全 Bot の本物の Token が placeholder で上書き**され、全 Bot が断線。

```bash
# ❌ こうやってしまった
openclaw config.patch '{
  "telegram": {
    "accounts": [{
      "name": "joe-bot",
      "botToken": "YOUR_TOKEN_HERE",  # ← これが全Botに適用された
      "binding": "agent:main"
    }]
  }
}'
```

**教訓**: `config.patch` で `telegram.accounts` を触る時は、**`botToken` フィールドを絶対に含めない**。binding や名前だけを更新すること。

:::message alert
この事故の復旧には、各 Bot の Token を BotFather から再取得して1つずつ再設定する必要がありました。10体分です。二度とやりたくない。
:::

### 🔥 5. モデル廃止

**症状**: ある日突然、特定の Agent が応答しなくなる。

**原因**: Claude 3 Opus が廃止され、API が `model_not_found` を返すようになった。OpenClaw の fallback 設定はあったが、**自動切替が期待通りに動かなかった**。

**教訓**: モデル指定は定期的に見直す。できれば `claude-sonnet-4-20250514` のような固定バージョンではなく、`anthropic/claude-sonnet-4` のようなエイリアスを使う。それでも廃止時は手動対応が必要になることがある。

## コスト

気になるコストをまとめます。

| 項目 | 月額 |
|------|------|
| Anthropic API（10 Agent） | $200〜300 |
| ハードウェア（自宅PC 3台） | ¥0（既存機流用） |
| Telegram Bot API | 無料 |
| Healthchecks.io | 無料枠 |
| GitHub Private Repo | 無料枠 |
| **合計** | **約 $200〜300/月** |

Claude Sonnet と Haiku をメインに使い、Opus は総管や複雑な判断が必要な場面に限定しています。10体で月 $200〜300 は安くはないですが、毎月50時間以上の時間短縮を考えると十分にペイします。

**初期構築は約2週間**。現在はほぼ自動運用で、heartbeat + 自動修復により、手動介入はほとんど不要です。

## 構築のステップ

「10体はさすがにハードル高いな……」と思った方、安心してください。最初から10体作る必要はありません。

### Step 1: まず1体から

```bash
# OpenClaw のインストール
npm install -g openclaw

# Gateway 起動
openclaw gateway start

# Telegram Bot を BotFather で作成して接続
openclaw config.patch '{
  "telegram": {
    "accounts": [{
      "name": "my-first-bot"
    }]
  }
}'
```

最初の1体で「朝の天気を教えて」「今日の予定は？」くらいから始めれば OK です。

### Step 2: 役割を分ける

1体で不便を感じたら分割する。「このコンテキストは分けたいな」と思った瞬間が、2体目を作るタイミングです。私の場合、掛け持ちしている案件ごとに Agent を分けたのが最大の効率改善でした。

### Step 3: 自動化を足す

cron job、heartbeat、sessions_spawn を組み合わせて、「聞かなくても教えてくれる」状態を目指す。ここまで来ると、もう Agent なしの生活には戻れません。

## まとめ

10体の AI Agent による個人管家系統を半年運用してきて、確信していることがあります。

**AI Agent の真価は「チャット」ではなく「自律的に動く」こと。**

聞かれたら答えるだけのチャットボットと、自分から情報を集めて報告してくれるエージェントは、まったく別物です。OpenClaw はその「自律性」を個人レベルで実現できる稀有なフレームワークだと感じています。

まずは1体から。「おはよう」の一言で一日の準備が整う体験、ぜひ味わってみてください。

## リンク

- 📖 [OpenClaw 公式ドキュメント](https://docs.openclaw.ai)
- 🐙 [GitHub](https://github.com/openclaw/openclaw)
- 💬 [Discord コミュニティ](https://discord.com/invite/clawd)

:::message
次回は「AI Agent 間の連携パターン」について書く予定です。Joe が他の Agent にタスクを委譲する仕組み（sessions_spawn）や、Agent 間のメモリ共有について深掘りします。お楽しみに！
:::
