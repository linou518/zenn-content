---
title: "電話番号だけで登録できるShopを、emailカラム必須のERPと同居させた話"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "自動化"]
published: true
---

---
title: "電話番号だけで登録できるShopを、emailカラム必須のERPと同居させた話"
emoji: "📱"
type: "tech"
topics: ["TypeScript", "React", "backend", "認証"]
published: true
category: web-tech
tags: [backend, auth, typescript, react, database, workaround]
lang: ja
date: 2026-03-15
---

# 電話番号だけで登録できるShopを、emailカラム必須のERPと同居させた話

既存のERPシステムにShopフロントを後付けしたとき、認証まわりで地味にハマったことがある。

ユーザーテーブルの `email` カラムが `NOT NULL + UNIQUE`。ERP側はメールアドレスでアカウントを管理していた。ところがShopのターゲットユーザーは卸売業者の得意先で、「スマホとLINEしか使わない」層が多い。メールアドレスなんて持っていないか、持っていても登録画面で入力したくない。

そこで選んだのが**プレースホルダーemailによる回避策**だ。

## 構成の概要

```
ERP Backend (Node.js + TypeScript)
  └─ User モデル: email (unique, not null), phone, name, role
  └─ /api/shop/auth/register  ← Shopから叩く
  └─ /api/shop/auth/login

Shop Frontend (React + TypeScript)
  └─ Register.tsx: phone + name + password (emailは任意)
  └─ Login.tsx: phone + password
```

ShopとERPは同じUserテーブルを共有している。ERPユーザーは `role: 'staff'` または `'admin'`、Shopのお客さんは `role: 'customer'` で区別している。

## バックエンド: プレースホルダーemailで制約をくぐる

登録エンドポイント `shopRegister` の核心はここだ。

```typescript
// emailが空 or 未指定の場合、phoneからプレースホルダーを生成
const email = req.body.email || `shop_${phone}@shop.local`;

// phoneの重複チェックはphoneで
const existingUser = await User.findOne({ where: { phone } });
if (existingUser) {
  res.status(400).json({ error: 'Phone number already registered' });
  return;
}

// プレースホルダーemailの重複チェックも一応する
const existingEmail = await User.findOne({ where: { email } });
if (existingEmail) {
  res.status(400).json({ error: 'Phone number already registered' });
  return;
}

const user = await User.create({ email, phone, name, password, role: 'customer' });
```

`shop_08012345678@shop.local` という形式で生成する。`.local` ドメインなので実際には存在しないメールアドレスだが、DBの一意制約は満たせる。フロントがemailを渡してきた場合はそちらを優先する。

## フロントエンド: emailは「任意」として表示

```tsx
// Register.tsx (抜粋)
const handleSubmit = async (e: React.FormEvent) => {
  await register(phone, password, name, email || undefined);
};

// emailフィールドのラベル
<label>
  メールアドレス <span className="text-gray-400 text-xs">（任意）</span>
</label>
<input type="email" value={email} onChange={(e) => setEmail(e.target.value)}
  placeholder="example@email.com" /* required なし */
/>
```

`required` を外すだけ。emailが空文字のときはバックエンドがプレースホルダーに差し替えてくれる。フロントで `email || undefined` と渡すことで、APIペイロードに空文字を混ぜない。

## ログイン: phoneで検索するだけ

```typescript
export const shopLogin = async (req: Request, res: Response): Promise<void> => {
  const { phone, password } = req.body;
  const user = await User.findOne({ where: { phone } });
  if (!user || !(await user.validatePassword(password))) {
    res.status(401).json({ error: 'Invalid phone number or password' });
    return;
  }
  // ... JWT発行
};
```

loginはphoneで検索するだけなのでシンプル。emailは参照しない。

## 実際にハマったこと

最初は `email` カラムを「任意」にするDBマイグレーションをかけようとした。が、ERP側のコードがメールアドレスを前提にしている箇所が多く、マイグレーションのコストが大きかった。

次に考えたのは「Shopユーザーは別テーブル」案。これはこれでJWT認証ミドルウェアを2本管理する羽目になる。

プレースホルダー方式は「ダサい」という直感はあるが、**既存コードへの影響がゼロ**というのは強い。ERP側は `@shop.local` のアドレスをメール送信先として使うことがないし、Shopの画面にはemailを表示しない。実害は今のところない。

## まとめ

- 既存DB制約を壊さずにフロント要件を吸収するときは、プレースホルダーデータで逃げる選択肢がある
- ただし「このフィールドは意味のあるデータが入っているとは限らない」という前提が、後からコードを読む人に伝わるようにしておくこと
- コメントか、フィールド名（`email_or_placeholder`みたいな）か、ドキュメントか——何かで明示しないと技術的負債になる

今回のケースは暫定で入れたが、将来的にemailカラムをnullable化するかどうかは継続検討中。「動いてるならそのまま」になりそうな予感はしている。

