---
title: "20+エージェントのチャット基盤をTelegramからSlackへ一日で全面移行した話"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "自動化"]
published: true
---

# 20+エージェントのチャット基盤をTelegramからSlackへ一日で全面移行した話

## TL;DR

9ノード・20+エージェントのOpenClawマルチエージェント環境を、TelegramからSlackへ一日で完全移行した。ハマりポイントと得られた教訓を記録する。

## 背景

自宅の GMK mini PC 9台クラスタで、OpenClawベースのAIエージェント群を運用している。仕事用・趣味用・家庭用など用途別にノードを分け、各ノードに複数エージェントが住んでいる構成だ。

これまで Telegram Bot で各エージェントと会話していたが、以下の理由でSlackへ移行することにした：

- Telegram は bot 単位の管理が煩雑（20+個の bot トークン管理）
- Slack ならワークスペース内でチャンネル・DMを統一管理できる
- ack reaction（👀）や streaming 表示など、UX 面で Slack の方が柔軟

## 作業内容

### Step 1: 全ノードからTelegram設定を一括削除

9ノードのOpenClaw設定から `channels.telegram`、関連 plugin、bindings を削除。SSH で各ノードに入り、`openclaw.yaml` を編集して gateway 再起動。

### Step 2: Slack App作成とトークン配置

ノードごとに Slack App を作成し、Bot Token と App Token を配置。ここで最初の罠にハマった。

### Step 3: 動作確認と修正の連続

ここからが本番。

## ハマったポイント5選

### 1. `assistant:write`スコープを追加するとDM送信不可になる

Slack App Manifest に OAuth scope として `assistant:write` を追加したところ、bot からの DM 送信が完全にブロックされた。Slack の「AI Assistant」モードが自動で有効化され、通常の `chat:write` による DM 送信が無効になる仕様だった。

**対策**: `assistant:write` を scope から除外して App 再作成。

### 2. `ackReactionScope: "group-mentions"`はDMで発火しない

OpenClaw の `messages.ackReactionScope` を `"group-mentions"` に設定していたところ、グループでのメンション時は 👀 リアクションが付くのに、DM では一切反応しなかった。

**対策**: `ackReactionScope: "all"` に変更。DM でも確実にリアクション。

### 3. マルチアカウント設定時のagent list漏れ

1ノードに複数エージェント（複数 Slack App）を配置する場合、`agents.list` に全エージェントIDを列挙する必要がある。漏れがあると binding が無効になり、デフォルトエージェントにフォールバックする。

### 4. Pi4でgateway起動失敗 — `controlUi.allowedOrigins`

`gateway.bind: "lan"` 設定の Pi4 ノードで、OpenClaw アップグレード後に gateway 起動が失敗。`controlUi.allowedOrigins` 未設定が原因。

**対策**: `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback: true` を追加。

### 5. Slack App Manifest更新にはApp Configuration Tokenが必要

通常の Bot Token では `app.manifest.update` API を叩けない。App Configuration Token（ワークスペース単位、12時間有効）を取得する必要がある。有効期限が短いので、自動化するなら refresh token の管理も必要。

## 移行結果

| 項目 | Before | After |
|------|--------|-------|
| チャット基盤 | Telegram Bot ×20+ | Slack App ×20+ |
| ack表示 | なし | 👀 リアクション |
| streaming | typing...が延々表示 | partial streaming 対応 |
| 管理画面 | Telegram BotFather | Slack App 管理 + API |
| 移行時間 | — | 約3時間（修正込み） |

## 教訓まとめ

1. **Slack App のスコープは慎重に** — `assistant:write` は副作用が大きい
2. **OpenClaw の ackReactionScope は `"all"` 一択**（DM 対応が必要なら）
3. **マルチエージェント構成では `agents.list` の漏れに注意**
4. **トークンと認証情報は `.env` ファイルに集約** — 設定ファイルに直書きしない
5. **一日で移行できるが、ハマりポイントは事前に知っておくと半日で済む**

## 環境

- OpenClaw 2026.2.x
- Ubuntu 24.04（GMK mini PC ×9台クラスタ）
- Slack Free Plan

---

Tags: `#OpenClaw` `#Slack` `#Telegram` `#マルチエージェント` `#インフラ` `#移行` `#lessons-learned`

<!-- 本文已脱敏处理: バージョン番号・プラットフォーム名のみ一般化 -->

