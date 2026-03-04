---
title: "ERPとECプラットフォームを統合Adminで動かす：デュアルトークン認証でログインループを直した話"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "自動化"]
published: true
---

---
title: "ERPとECプラットフォームを統合Adminで動かす：デュアルトークン認証でログインループを直した話"
emoji: "🔐"
type: "tech"
topics: ["React", "TypeScript", "認証設計", "ERP", "axios"]
published: true
category: "web-tech"
tags: ["React", "TypeScript", "認証設計", "ERP統合", "axios", "Admin開発", "バグ修正"]
---

# ERPとECプラットフォームを統合Adminで動かす：デュアルトークン認証でログインループを直した話

## 何が起きたか

今日、techsfree-platform（ERP + ECショップを一元管理するAdmin画面）の開発中に厄介なバグを踏んだ。

症状はこうだ：ログインすると商城（ECショップ）系のページは表示できるのに、売上・仕入・在庫といったERP系のページに遷移しようとすると、なぜか `/login` にリダイレクトされる。ページをクリックするたびにログイン画面に戻される無限ループ。

## 構成の背景

このプロジェクトは2つの異なるバックエンドを1つのAdmin UIから操作する構成になっている：

- **Platform backend**（port 3210） — EC機能（商品/注文/会員）、独自JWT（`tf_token`）
- **ERP backend**（port 8520）— 売上/仕入/在庫管理、これまた独自JWT（`erp_token`）

ERP部分はもともと別プロジェクトとして動いていたもので、認証トークンが完全に独立している。

## 根本原因の特定

`erpApi.ts`（ERP用のAxiosクライアント）のレスポンスインターセプターを見たら、一発でわかった：

```typescript
// こんなコードが仕込まれていた
instance.interceptors.response.use(
  response => response,
  error => {
    if (error.response?.status === 401) {
      window.location.href = '/login'; // ← コイツが犯人
    }
    return Promise.reject(error);
  }
);
```

ERP backendはERP専用のJWTしか受け付けない。Platform側の`tf_token`を送ってしまうと当然401が返る。そして401が来るたびに `/login` 強制リダイレクト。

フロントエンドはERP系ページに遷移するたびにERPへAPIリクエストを飛ばし、`tf_token`で弾かれ続けるという構図だった。

## 解決策：Dual-login実装

「ログイン時にERP側にも同時ログインして、トークンを2本持つ」という方針にした。

### バックエンド修正（Platform側）

Platform backendの`/auth/login`エンドポイントを修正。ユーザーがadminかstaffの場合、バックグラウンドでERP backendにも自動ログインして`erp_token`を取得し、レスポンスに同梱する：

```typescript
// fetchErpToken(): ERP backendにログインしてトークンを返す
async function fetchErpToken(email: string, password: string): Promise<string | null> {
  try {
    const response = await axios.post(
      `${process.env.ERP_URL}/auth/login`,
      { email, password },
      { timeout: 5000 }
    );
    return response.data?.token || null;
  } catch {
    return null; // ERP連携失敗でもPlatformログインはブロックしない
  }
}

// login endpoint
const erp_token = await fetchErpToken(email, password);
res.json({ token, erp_token, user });
```

失敗してもnullを返してログイン全体をブロックしないのがポイント。ERP側が落ちていてもEC機能は使えるべき。

### フロントエンド修正

`AuthContext.tsx`：ログイン時に`erp_token`も`localStorage`に保存：

```typescript
localStorage.setItem('tf_token', data.token);
if (data.erp_token) {
  localStorage.setItem('tf_erp_token', data.erp_token);
}
```

`erpApi.ts`のインターセプター：`tf_erp_token`を優先して使い、401時は強制リダイレクトを**やめる**：

```typescript
// リクエスト: erp_tokenを優先
instance.interceptors.request.use(config => {
  const token = localStorage.getItem('tf_erp_token')
              || localStorage.getItem('tf_token');
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

// レスポンス: 401でも/loginに飛ばさない
instance.interceptors.response.use(
  response => response,
  error => {
    if (error.response?.status === 401) {
      localStorage.removeItem('tf_erp_token'); // ERPトークンだけ消す
      // window.location.href = '/login'; ← 削除
    }
    return Promise.reject(error);
  }
);
```

もう一つの落とし穴として、ERP backendに管理者ユーザーが存在しなかったため、DB直接操作でロールをadminに昇格させる必要があった：

```sql
UPDATE users SET role='admin' WHERE email='admin@example.com';
```

## 教訓

**複数バックエンドを1つのフロントから叩く設計では、認証が一番複雑になる。**

インターセプターに`window.location.href`を書くのは「手軽だが脆い」パターン。複数APIクライアントが存在する場合はグローバルな副作用を避けて、エラーを上位に伝播させるか、認証状態はContextで一元管理するほうが安全。

ポイントをまとめると：

1. **APIクライアントごとに認証を独立させる** — 片方の401がもう片方のセッションに影響しない設計
2. **失敗してもログインをブロックしない** — ERP側が落ちてもEC機能は使えるべき
3. **インターセプターのグローバル副作用を避ける** — `window.location.href`はContextで管理

既存ERPとの統合という制約の中で、このデュアルトークンアプローチは現実的な解決策だった。

