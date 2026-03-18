---
title: "ショップ認証をphone OR emailどちらでも使えるように改修した話"
emoji: "📱"
type: "tech"
topics: ["typescript", "zod", "authentication", "ux"]
published: true
---

電話番号だけで登録してもらいたいのに、バックエンドがemail必須で弾いていた。地味なバグだけど、実際のユーザーが詰まっていたので直した記録。

## 背景

LINE連携のECサイトで、ターゲットユーザーの多くはメールアドレスを持っていないか、入力を嫌がる。登録フォームに「電話番号だけ入れてください」と書いてあるのに、送信すると `400 Bad Request` が返ってくる——という状態だった。

原因はシンプル。フロントエンドはphoneだけ送っていたが、バックエンドのバリデーションが `email: required` だった。

## 何を変えたか

### フロントエンド（Vue + Vite）

登録フォームのemail欄を「任意」に変更。ラベルも「メールアドレス（任意）」に。

```vue
<input
  v-model="form.email"
  type="email"
  placeholder="メールアドレス（任意）"
/>
```

送信時のバリデーションからemail必須チェックを削除。

### バックエンド（Node.js / ts-node）

`/api/auth/register` エンドポイントのバリデーションを修正：

```typescript
// Before
const schema = z.object({
  phone: z.string(),
  email: z.string().email(),  // ← これが問題
  password: z.string().min(6),
});

// After
const schema = z.object({
  phone: z.string().optional(),
  email: z.string().email().optional(),
  password: z.string().min(6),
}).refine(data => data.phone || data.email, {
  message: 'phone または email のどちらかは必須です',
});
```

ログインも同様に、phoneかemailかどちらかでユーザーを検索するよう変更：

```typescript
const user = await User.findOne({
  where: phone ? { phone } : { email },
});
```

## 落とし穴

**既存ユーザーとの衝突問題。**

テスト中、「電話番号がすでに登録されている」エラーが出た。よく調べるとテスト用のadminアカウントに実ユーザーの電話番号が紐付いていた。本番データが入ったDBで開発しているリスクを再認識した。

対処：該当ユーザーには「すでに登録済みです、ログインしてください」と案内する。新規登録ではなくログインフローに誘導するUIも追加。

## 学んだこと

**「必須」の判断はユーザー文脈で決まる。**
開発者視点だとemailは自然な識別子だが、現場ユーザーには電話番号のほうがずっと馴染みがある。特に高齢者やITリテラシーが低めのユーザーをターゲットにするBtoCサービスでは、認証フローのUXが離脱率に直結する。

**zod の `.refine()` は便利。**
「AまたはBのどちらかが必須」という条件は個別フィールドのバリデーションでは書けない。`.refine()` でオブジェクトレベルのチェックを追加することで、きれいに表現できた。

<!-- 本文已脱敏処理済み -->
