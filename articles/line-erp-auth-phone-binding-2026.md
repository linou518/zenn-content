---
title: "LINE Bot × ERPシステムのユーザーバインド設計——電話番号マッチングで作る認証連携"
emoji: "🔗"
type: "tech"
topics: ["linebot", "erp", "typescript", "sequelize", "backend"]
published: true
---

## はじめに

LINEで在庫を確認したい、注文状況を聞きたい——ERPに接続したチャットbotを作るとき、最初の壁になるのが「このLINEユーザーはERP上の誰なのか」という同一性確認だ。

最近、社内向けの食品卸売ERPシステムにLINE Botを連携させた。実装の中核になったのが「電話番号マッチング + LINE IDバインド」という認証設計で、意外とシンプルな仕組みで実用的な権限管理が実現できた。実装過程を整理しておく。

## 問題設定：LINEとERPのID空間は別物

LINEのユーザーIDは `U2eec19b8c3961a95f436ca8eb185e7f0` のような文字列で、ERPの内部ユーザーID（DBの整数ID）とは別管理になる。

チャットbotでERP操作を許可するには、「このLINE IDがどのERPアカウントか」を事前に紐付ける必要がある。

よくある選択肢:
1. **ログインフロー**: botからERP認証ページのURLを送りLINEログインさせる → リッチだが実装コスト高
2. **招待コード方式**: 管理者が一時コードを発行してLINEで入力 → セキュアだが手間
3. **電話番号マッチング**: ERPに登録済みの電話番号でバインド → シンプル、社内利用に最適

今回は社内スタッフのみ対象だったので、**電話番号マッチング**を採用した。

## DB設計：usersテーブルに2カラム追加するだけ

```sql
ALTER TABLE users 
  ADD COLUMN line_user_id VARCHAR(64) UNIQUE NULL,
  ADD COLUMN line_display_name VARCHAR(255) NULL;
```

既存の `users` テーブルに2カラム追記するだけ。Sequelizeの `sync()` で自動適用できる。

```typescript
// User.ts (Sequelize Model)
@Column({ type: DataType.STRING(64), unique: true, allowNull: true })
line_user_id!: string | null;

@Column({ type: DataType.STRING(255), allowNull: true })
line_display_name!: string | null;
```

## APIエンドポイント設計

LINE Bot側から呼び出すAPIを3本用意した:

```
GET  /api/line/user/:line_user_id   # LINE IDで身元照会
POST /api/line/bind                  # 電話番号でバインド
POST /api/line/unbind                # バインド解除
```

### バインドAPI

```typescript
// lineController.ts
export const bindUser = async (req: Request, res: Response) => {
  const { line_user_id, line_display_name, phone } = req.body;
  
  // 電話番号でERPユーザーを検索
  const user = await User.findOne({ where: { phone } });
  if (!user) {
    return res.status(404).json({ error: 'Phone number not registered in ERP' });
  }
  
  // LINE IDをバインド
  await user.update({ line_user_id, line_display_name });
  
  return res.json({
    success: true,
    role: user.role,
    permissions: getPermissions(user.role)
  });
};
```

### 身元照会API

```typescript
export const getLineUser = async (req: Request, res: Response) => {
  const { line_user_id } = req.params;
  
  const user = await User.findOne({ where: { line_user_id } });
  if (!user) {
    return res.json({ bound: false });
  }
  
  return res.json({
    bound: true,
    role: user.role,
    name: user.name,
    permissions: getPermissions(user.role)
  });
};
```

## 権限グループの整理

ERPのロールは `admin` / `staff(user)` の2種類に絞った。顧客（customer）はLINE Botでの操作対象外とした。

```typescript
const getPermissions = (role: string): string[] => {
  if (role === 'admin') {
    return ['sales', 'purchases', 'inventory', 'products', 
            'customers', 'suppliers', 'payments', 'reports', 'staff'];
  }
  if (role === 'user') {
    return ['inventory', 'products', 'purchases', 'sales', 'shipments'];
  }
  return []; // 未バインドまたは不明なロール
};
```

シンプルだが、「在庫は見られるが財務レポートは見られない」という業務的な棲み分けはこれで十分。

## LINE Bot側の処理フロー

```
LINEメッセージ受信
    ↓
GET /api/line/user/:line_user_id で身元照会
    ↓
bound: false → 「電話番号を送信してください」と案内
                POST /api/line/bind でバインド実行
    ↓
bound: true  → permissions に基づいてERP APIを呼び出し
              → 結果をLINEで返信
```

バインド済みかどうかを毎メッセージごとに確認するため、余計なセッション管理が不要になる。LINE Bot はステートレスに設計できる。

## 実装してわかったこと

**電話番号マッチングのリスク**: ERPに登録されていない電話番号でメッセージを送ると永遠にバインドできない。スタッフのERP登録を先にやっておく必要がある（当然だが見落としやすい）。

**TypeScriptの型エラーとの格闘**: `User.findOne()` の返り値が `User | null` なのに、中で更新しようとすると型エラーが出る。`if (!user) return` のガード節を挟んでからOperationsするのが基本だが、Sequelizeのモデル型はたまに混乱する。

**ステートレス設計の快適さ**: LINE Bot をステートレスに保てたのは大きい。botのプロセスが再起動してもユーザーがもう一度バインドし直す必要がない。セッションファイルの管理が不要。

## まとめ

LINEとERPの連携で一番シンプルな認証方式として「電話番号マッチング + LINE IDバインド」を実装した。

- DBへの追加は2カラムだけ
- APIは3本（照会/バインド/解除）
- Bot側はステートレスに保てる

社内向けの業務botならこのくらいシンプルで十分機能する。認証が複雑になると使われなくなる——これが一番のリスクだと思っている。
