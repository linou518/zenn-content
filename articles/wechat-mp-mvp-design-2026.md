---
title: "WeChat Mini Programで飲食店注文システムを設計した話 — MVP戦略と技術選定の実録"
emoji: "🍜"
type: "tech"
topics: ["WeChat", "MiniProgram", "MVP", "Taro", "React"]
published: true
---

実際にWeChat Mini Program（微信小程序）で飲食店向け注文システムを設計した記録。技術選定からMVP範囲の切り方、ハマりポイントまで、リアルな話を書く。

## 背景

クライアントは日本法人（合同会社）が運営する中華料理店。既にERP（販購存管理システム）のバックエンドがあり、商品マスタや在庫はAPIで取れる状態。ここにWeChat Mini Programを載せて、中国人顧客がスマホから注文できるようにしたい。

## 技術選定: Taro + React + TypeScript

Mini Program開発のフレームワーク選定で検討したのは3つ:

| 候補 | メリット | デメリット |
|------|---------|-----------|
| **WeChat純正（WXML/WXSS）** | 公式ドキュメント通り | 独自構文、再利用性低い |
| **uni-app (Vue系)** | マルチプラットフォーム | Vue前提、エコシステムが中国寄り |
| **Taro (React系)** | React/TS対応、型安全 | ビルド設定がやや複雑 |

**Taroを選んだ理由:**
- チームがReact/TypeScriptに慣れている
- 将来的にH5（Webアプリ）版も同一コードベースから出力可能
- 型定義でAPI連携のバグを減らせる

## MVP範囲の切り方が一番大事だった

最初の要件は「注文＋決済＋配達管理」と盛りだくさんだったが、ここで一番時間をかけたのがスコープの削減。

### Phase 1（MVP）: 閲覧 + 注文のみ
- 商品一覧表示（カテゴリ別）
- カート機能
- 注文送信（店舗側で受注確認）
- **決済はオフライン**（店頭支払い / 既存の決済手段）

### Phase 2: WeChat Pay統合
### Phase 3: 配達管理・ポイント

**WeChat Payを外した理由がブロッカーそのもの:**

WeChat Pay（微信支付）の商戸号申請には中国大陸の銀行口座が必要。日本法人の場合「境外商戸」として申請するルートがあるが、審査に時間がかかる上に、法人の銀行口座状況の確認が先。

つまり技術的にはすぐ実装できるが、**ビジネス側の手続きがボトルネック**。これをPhase 1に入れると全体が止まる。決済なしでも「メニュー閲覧→注文→店頭支払い」のフローは成立するので、まずそこで回す判断をした。

## shop.techsfree.com — APIエンドポイントの構築

Mini ProgramからERP APIを叩くために、公開エンドポイントが必要。既存のERP backendは内部ネットワーク（192.168.x.x:8520）で動いているので、Nginxでリバースプロキシを立てた。

```nginx
server {
    listen 443 ssl;
    server_name shop.techsfree.com;

    ssl_certificate /etc/letsencrypt/live/shop.techsfree.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/shop.techsfree.com/privkey.pem;

    location /api/ {
        proxy_pass http://127.0.0.1:8520/api/;
        add_header Access-Control-Allow-Origin *;
        add_header Access-Control-Allow-Methods "GET, POST, OPTIONS";
    }
}
```

SSL証明書はLet's Encrypt（certbot）で取得。Mini Programは**HTTPSのみ**許可なので、これは必須。

### ハマりポイント: LAN内NATヘアピン

開発中、LAN内からshop.techsfree.comにアクセスするとタイムアウト。ルーターがNATヘアピン（NAT loopback）に対応していなかった。開発時は内部IP直指定で回避。テスト時に地味にハマる。

## AppSecret検証 — access_tokenの取得確認

Mini Program開発で最初にやるべきは、AppID/AppSecretでaccess_tokenが取れることの確認。

```bash
curl "https://api.weixin.qq.com/cgi-bin/token?grant_type=client_credential&appid=YOUR_APPID&secret=YOUR_SECRET"
```

これが通れば、WeChat APIとの疎通はOK。逆にここでコケると、AppIDの種類違い（公众号 vs 小程序）やIP白名単の設定ミスなど、根本的な問題がある。**最初の5分でやるべきスモークテスト。**

## まとめ

- **MVPは「何を入れるか」より「何を外すか」が9割。** 決済という明らかな機能をPhase 1から外す判断が、プロジェクト全体を前に進めた。
- **ビジネス手続きのリードタイムを技術設計に織り込む。** WeChat Payの審査は数週間〜数ヶ月。待っている間にUIとフローを固める。
- **スモークテストは最初の5分。** access_token取得、SSL証明書、API疎通。基盤が動くことを確認してから設計に入る。
