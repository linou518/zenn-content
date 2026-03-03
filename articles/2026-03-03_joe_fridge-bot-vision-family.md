---
title: "LINEボットに「冷蔵庫を見てくれ」と送ったら食材を登録してくれるやつを作った"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "自動化"]
published: true
---

# LINEボットに「冷蔵庫を見てくれ」と送ったら食材を登録してくれるやつを作った

**公開日**: 2026-03-03  
**著者**: Joe  
**タグ**: `LINE Bot` `Claude Vision API` `TypeScript` `Node.js` `家族向けアプリ` `AI実装記録`

---

## 背景

妻が「冷蔵庫に何が入ってるか管理したい」と言い出した。専用アプリを探してみたが、UI微妙・外国製・データがどこ行くか不明、という感じでしんっくりこなかった。

じゃあ作るか。

LINEはすでに家族全員使ってる。食材の写真を撮ってLINEに送ったら自動で登録、というフローが一番ストレスゼロだろうという判断で、Fridge Bot Phase 2の実装に入った。

---

## アーキテクチャ概要

```
LINE (ユーザー) → LINE Messaging API → Node.js Bot (web.34)
                                          ↓
                               Claude Vision API
                                          ↓
                              SQLite (冷蔵庫DB)
```

スタック: TypeScript + Node.js、PM2でプロセス管理、デプロイ先は自前VPS (web.34)。LINEのWebhookを受け取ってClaudeに投げ、結果をSQLiteに保存する構成。

---

## Task 3: 画像→食材自動認識

### 実装した処理フロー

1. LINEから画像メッセージを受信
2. `getMessageContent()` でバイナリをDL → base64エンコード
3. Claude Vision APIに投げる（`claude-3-5-sonnet`）
4. 返ってきたJSON（食材名・数量・賞味期限推定）をパース
5. DBに登録、ユーザーに確認メッセージを返す

### プロンプト設計がキモだった

```typescript
const prompt = `
以下の画像を分析して、見えている食材・食品を識別してください。

対応シナリオ:
- バーコード写真 → 商品名を推定
- レシート写真 → 購入品目を抽出
- 食品写真（野菜・肉・パッケージ等）→ 品目・推定数量

JSON形式で返答してください:
{
  "items": [
    { "name": "食材名", "quantity": 数量, "unit": "個/g/本等", "estimatedExpiry": "YYYY-MM-DD or null" }
  ],
  "confidence": 0.0-1.0,
  "note": "不明点があれば"
}
`;
```

最初は「食材を教えて」だけで投げてたら、バーコードだけ写ってる写真でハルシネーションが出た。シナリオを明示的に書いてから精度が安定した。

### 型定義

```typescript
interface ImageAnalysisResult {
  items: {
    name: string;
    quantity: number;
    unit: string;
    estimatedExpiry: string | null;
  }[];
  confidence: number;
  note?: string;
}
```

`confidence` が0.5未満のときはユーザーに「確認してください」メッセージを付けるようにした。

---

## Task 4: 家庭成員システム

食材管理だけだと「誰が嫌いなものをうっかり使う」問題が解決しない。家族ごとにアレルギーや苦手食材を登録できる仕組みを追加した。

### DBスキーマ（追加分）

```sql
CREATE TABLE family_members (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL,
  allergies TEXT,        -- JSON配列で保存
  dislikes TEXT,         -- JSON配列
  nutrition_goals TEXT,  -- JSON（目標カロリー等）
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

### LINEコマンド

テキストベースのコマンドで操作できるようにした:

| コマンド | 動作 |
|----------|------|
| `添加家人 太郎` | 家族メンバー追加 |
| `过敏 太郎 えび` | アレルギー登録 |
| `不吃 花子 なす` | 苦手食材登録 |
| `删除家人 太郎` | メンバー削除 |
| `家庭成员` | 一覧表示 |

コマンド名が中国語なのは、メイン開発者（自分）の母語だから。LINEは日本語UIだが内部コマンドはわりと何でも動く。

### 栄養目標計算

`familyService.ts` に栄養計算ロジックを入れた。将来的にはレシピ提案時に「太郎はえびアレルギーあるよ」と注意を出す予定。

---

## ハマったところ

**TypeScriptのビルドエラーが地味につらかった**

`client.ts` から `getMessageContent` をexportし忘れていてビルドが通らなかった。`tsc` のエラーメッセージは親切なので、焦らず読めば解決できる。でも何回「zero errors」見たら気が済むんって気持ちにはなる。

**LINEの画像URLは一時的**

`getMessageContent()` で取得できる画像URLはLINEのCDN上の一時URLで、時間が経つと無効になる。受信即取得・即処理が必須。

---

## 今日時点でのデプロイ状況

- web.34 にPM2で稼働中
- port 3420でListen
- SQLite DB: `/var/www/fridge-bot/fridge.db`

Phase 2残タスク: Task 5（レシピ強化）→6（棚卸）→7（買い物/予算）→8（通知）。今週中にTask 5まで終わらせたい。

---

## まとめ

「写真を送ったら食材登録」というシンプルなUXは、Claude Vision + LINE Webhookの組み合わせで意外とサクッと実装できた。プロンプトのシナリオ明示とconfidenceによる確認フローが品質のキモ。

家族向けアプリは「使ってもらえるか」が全てなので、UIをゼロにしてLINEに乗っかる判断は正解だったと思ってる。

---

*この記事は実際の開発作業ログをベースに書いています。*

