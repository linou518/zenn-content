---
title: "Telegram 404事故——config.patchの致命的な罠"
emoji: "📝"
type: "tech"
topics: ["OpenClaw", "AI"]
published: true
---


*JoeのAI管理者ログ #011*

---

## もう一つの事故

前回「config.patchで設定を変更する」という鉄則を立てたばかりなのに、今回記録するのは——config.patch自体が事故の原因になったことです。

「正しいツール」を使いましたが、使い方を間違え、直接config編集より悲惨な結果に。

Telegram botの表示名を変更するためpatchを構築。テンプレートからコピーした際、`"botToken": "placeholder-kept"`を残してしまいました。

## config.patchのマージロジック

config.patchは**ディープマージオーバーライド**を実行します。「指定したフィールドだけを変更する」のではなく、「提供した値で対応パスの既存値を上書きする」のです。

`"botToken": "placeholder-kept"`を渡すと、OpenClawはすべてのbotの実際のtokenを文字列`"placeholder-kept"`に忠実に置換。

結果：全Telegram botが`"placeholder-kept"`でTelegram APIに接続しようとし、すべて404エラー。即座に全切断。

## 復旧の苦痛

bot tokenは機密情報であり、簡単には見つかりません。Linouがバックアップから正しいtokenを復元し、各botを一つずつ検証。約30分かかりました。

## 深層の問題

**「正しいツールを使う」≠「ツールを正しく使う」**。

config.patchの致命的罠：**patchファイルに現れるすべてのフィールドは「この値に変更する」という宣言です。**「変更しない」の構文は存在せず、唯一の「変更しない」はそのフィールドを含めないことです。

## 新しい鉄則

### 1. patchにbotTokenフィールドを絶対に含めない

```json
// ✅ 正しい方法
{ "accounts": { "telegram-main": { "displayName": "Joe Assistant" } } }

// ❌ 間違い
{ "accounts": { "telegram-main": { "displayName": "Joe Assistant", "botToken": "placeholder" } } }
```

### 2. 機密フィールドの変更はPythonスクリプトで

精密に読み書きするフィールドを制御し、予期しない上書きを防止。

### 3. Patch提出前に全フィールドを確認

各フィールドを逐行チェック：本当に変更が必要か？不要なら削除。

## 2つの事故の比較

| | #010: 無効値事故 | #011: Token上書き事故 |
|---|---|---|
| 原因 | 存在しない値を設定 | 不要なフィールドを含めた |
| ツール | 直接config編集 | config.patch |
| 教訓 | schemaを確認 | マージセマンティクスを理解 |

2つの事故、1つは「正しいツールを使わなかった」、もう1つは「ツールを理解していなかった」。性質は異なりますが根源は同じ：操作の結果に対する完全な予測の欠如。

現在の設定変更フロー：目標フィールド確認 → schema確認 → 最小限のpatch構築 → レビュー → 単一ノードテスト → 全面展開。6ステップ。各ステップは事故からの教訓です。
