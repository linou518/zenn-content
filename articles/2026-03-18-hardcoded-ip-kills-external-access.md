---
title: "ハードコードIPがプロダクション環境を殺した話 — Vite + Nginx構成の落とし穴"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "自動化"]
published: true
---

---
title: "ハードコードIPがプロダクション環境を殺した話 — Vite + Nginx構成の落とし穴"
emoji: "🔧"
type: "tech"
topics: ["Vite", "Nginx", "フロントエンド", "デプロイ"]
published: true
category: "web-tech"
lang: "ja"
---

# ハードコードIPがプロダクション環境を殺した話 — Vite + Nginx構成の落とし穴

フロントエンドにハードコードされたプライベートIPが、外部アクセスを完全に壊していた。原因特定から修正・デプロイまでの記録。

## 背景

社内向け業務システム（Vite + React フロントエンド + Express バックエンド）を、`erp.example.com` で外部からもアクセスできるようにしたかった。社内LAN内では問題なく動いていたのに、外部からアクセスすると何も動かない。

## 発見した構成の問題

調べてみると、こんな状態だった：

- `erp.example.com` のnginx設定：
  - `/api/` → `127.0.0.1:3210`（バックエンド）✅
  - `/` → `127.0.0.1:3101`（フロントエンド）❌ — **このポートでは何も動いていなかった**
- フロントの実体は別サーバーの vite dev server (port 3101) で配信 — LAN内のみアクセス可

つまり、開発用のvite dev serverがそのまま「本番」として使われていた。

## 本当の致命傷：ハードコードIP

さらに問題だったのが、フロントエンドのソースコード全体に散らばったプライベートIP：

```typescript
// erpApi.ts
const BASE_URL = 'http://192.168.x.x:8520/api';

// 各ページコンポーネント
fetch('http://192.168.x.x:8520/api/sales/report/ledger', ...)
fetch('http://192.168.x.x:3210/api/v1/arrivals/slip?ids=1', ...)
```

LAN内のブラウザなら `192.168.x.x` に直接到達できるから動く。外部からは当然到達できない。vite buildしてnginxで配信しても、ビルド済みJSの中にこのIPがそのまま焼き込まれる。

**結果：外部からアクセスすると、全APIコールが `ERR_CONNECTION_TIMED_OUT` で全滅。**

## 修正内容

### 1. ハードコードIPを相対パスに置換

```typescript
// Before
const BASE_URL = 'http://192.168.x.x:8520/api';

// After
const BASE_URL = '/api';
```

対象ファイル：`erpApi.ts`, `erpCompat.ts`, 各一覧ページ（SaleList, KaikakeList, NyukinList 他）。grep で `192.168.x` を全件洗い出して潰した。

### 2. `.env.production` の作成

```env
VITE_API_URL=/api/v1
VITE_ERP_URL=/api
```

Viteの環境変数で開発時と本番時のAPIベースURLを切り替えられるようにした。

### 3. Nginx設定の変更

```nginx
# Before: vite dev server にプロキシ（動いてない）
location / {
    proxy_pass http://127.0.0.1:3101;
}

# After: ビルド済み静的ファイルを直接配信
location / {
    root /www/apps/erp-admin;
    try_files $uri $uri/ /index.html;
}

location /api/ {
    proxy_pass http://127.0.0.1:3210;
}
```

SPAなので `try_files` で全パスを `index.html` にフォールバックさせる定番構成。

### 4. ビルド＆デプロイ

```bash
cd /mnt/shared/projects/platform/erp-admin
npm run build    # dist/ にハッシュ付きファイル生成
scp -r dist/* admin@webserver:/www/apps/erp-admin/
ssh admin@webserver "nginx -t && nginx -s reload"
```

## Vite dev proxy の罠

ついでにハマったポイント：`vite.config.ts` の `server.proxy` 設定は **開発時（`npm run dev`）のみ有効** で、`npm run build` の成果物には一切影響しない。

```typescript
// vite.config.ts — これはdev serverの設定であってbuildには無関係
server: {
  proxy: {
    '/api/v1': 'http://192.168.x.x:8520',
  }
}
```

つまり「devでは動くのにbuildすると壊れる」パターンの原因になりがち。本番のプロキシはnginx側で設定する必要がある。

## 教訓

1. **フロントエンドにプライベートIPをハードコードするな。** 相対パス (`/api`) か環境変数 (`import.meta.env.VITE_API_URL`) を使え。
2. **vite dev serverを本番配信に使うな。** `npm run build` → nginx静的配信が正解。
3. **`server.proxy` はdev専用。** 本番のリバースプロキシはWebサーバー側で設定する。
4. **LAN内で「動いてる」は検証にならない。** 外部アクセスをテストしないと、ハードコードIPは見つからない。

当たり前のことばかりだけど、「とりあえず動いてるからいいや」で積み重なると、いざ外部公開しようとしたときに全部壊れる。grepで `192.168.x` を探すところから始めよう。

---

**Tags:** #vite #nginx #frontend #deployment #spa #erp #hardcoded-ip #devops

<!-- 本文已脱敏処理 -->
