---
title: "TechsFreeプラットフォームを2台のサーバーから1台に集約した話"
emoji: "🔧"
type: "tech"
topics: ["インフラ", "nginx", "mysql", "astro", "devops"]
published: true
---

# TechsFreeプラットフォームを2台のサーバーに分散させていたのを1台に集約した話

自宅GMKクラスター（9台構成）を運用していると、いつの間にか「どのサービスがどこで動いているか」が曖昧になってくる。今日はその棚卸しと、TechsFreeプラットフォームの全面移行を一気にやったので記録しておく。

---

## 背景：infraとwebに散らばったカオス

インフラ構成は大まかにこう：
- **infraサーバー**: T440サーバー、メッセージバス・各種開発環境・一時的なサービス置き場
- **webサーバー**: 宝塔CentOS、Nginx+SSL込みの本番Webサーバー

問題はinfraサーバーが「一時的なサービス置き場」として機能しすぎていたこと。

今日の棚卸しでinfraサーバーのポートを列挙したら、こんな惨状だった：

| ポート | サービス | 状態 |
|--------|----------|------|
| :3001 | ERP API Backend（旧online-shop） | PM2管理、127回再起動済み |
| :5174 | Shop Frontend（vite preview） | PM2管理 |
| :5175 | ERP Frontend（vite dev） | PM2管理外のプロセス |
| :3400~3409 | linebot実験プロセス10個 | PM2管理外 |
| :8080-8081 | likeshop Docker | infraとwebの両方に存在 |
| :8530 | techsfree-homepage python http.server | 臨時起動の放置 |

`python3 -m http.server 8530` が本番同然で動いていたのは流石に笑えない。

---

## 作業1：infraサーバーのゴミ掃除

まずinfraサーバーから不要プロセスを全部落とした：

```bash
# linebot実験プロセス10個をkill
ps aux | grep linebot | awk '{print $2}' | xargs kill

# likeshop Dockerコンテナ停止
docker stop nginx node php mysql redis

# Python http.serverも停止
pkill -f "http.server 8530"
```

PM2の `restart: 127` は笑い話だが、それだけ長期間放置されていた証拠でもある。

---

## 作業2：TechsFreeプラットフォームをwebサーバーに全面移行

TechsFreeプラットフォームは:
- **Admin（ERP管理画面）**: React SPA
- **Shop-PC**: Nuxt3 SSR
- **Shop-H5**: Vue3 SPA（モバイル）
- **Backend**: Node.js (Express + Prisma)
- **DB**: MySQL（Dockerで管理）

### DBマイグレーション

MySQLはlikeshopのDocker bind mountの中に紛れ込んでいた。直接コピーはDocker内部形式が違うのでNG。一時コンテナ経由でdumpした：

```bash
# 一時MySQLコンテナで接続してdump
docker run --name tmp-mysql-dump \
  -v /path/to/mysql:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=<DB_PASSWORD> \
  -p 13307:3306 mysql:5.7 -d

mysqldump -h 127.0.0.1 -P 13307 -u root -p techsfree_platform > dump.sql

# webサーバーに転送してインポート
scp dump.sql user@web-server:/tmp/
ssh user@web-server "mysql -h 127.0.0.1 -P 3308 -u root -p techsfree_platform < /tmp/dump.sql"
```

37テーブル、インポート成功。

### フロントエンドのビルドとNginx設定

Admin SPAは旧サーバーIPをハードコードしていたので修正して再ビルド。Nginxのproxy_passを更新：

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:3210/api/;
}
location / {
    root /www/wwwroot/erp.techsfree.com;
    try_files $uri $uri/ /index.html;
}
```

### 管理者パスワードのリセット

移行後、管理者パスワードが不明に。bcryptjsで新しいhashを生成してDBを直接UPDATE：

```bash
node -e "
const bcrypt = require('./node_modules/bcryptjs');
console.log(bcrypt.hashSync('new-password', 10));
"
mysql -e "UPDATE tf_users SET password='\$2a\$10\$...' WHERE email='admin@techsfree.com'"
```

---

## 作業3：Astro製ブログのデプロイとハマりポイント

### NFS上でnpm installができない

解決：ローカルの`/tmp/`にコピーしてビルド：

```bash
cp -r /mnt/shared/projects/04_techsfree-blog/ /tmp/blog-build/
cd /tmp/blog-build && npm install && npm run build
```

### curlのテストが誤判定

`curl http://127.0.0.1/blog/` が404を返してきたが、`Host: 127.0.0.1`でデフォルトサーバーに拾われていたせい。

```bash
# OK: Hostヘッダーを明示する
curl -H "Host: your-domain.com" http://127.0.0.1/blog/
```

---

## 今日の教訓

1. **一時的なものは必ず期限を決める** — `python3 -m http.server`に本番の役割を与えてはいけない
2. **NFS上でnpm installはしない** — `/tmp/`にコピーしてビルドする
3. **curlテストはHostヘッダーを意識する** — Nginx複数vhost環境では`-H "Host: xxx"`を忘れずに
4. **DB移行は一時コンテナ経由が安全** — bind mountを直接触るより、mysqldump経由の方がポータビリティが高い

インフラの棚卸しは定期的にやらないと、気づいたら127回再起動してるゾンビプロセスが量産される。
