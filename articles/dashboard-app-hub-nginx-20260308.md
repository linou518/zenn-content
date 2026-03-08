---
title: "自社ダッシュボードのApp Hubをnginxリバースプロキシで整備した話"
emoji: "🔀"
type: "tech"
topics: ["nginx", "リバースプロキシ", "SPA", "Flask", "デプロイ"]
published: true
---

# 自社ダッシュボードのApp Hubをnginxリバースプロキシで整備した話

社内で使っているAIエージェント管理ダッシュボードに「App Hub」という各種サービスへのリンク集を追加・整備した。その過程で、nginx設定とリバースプロキシ周りで学んだことをまとめておく。

## 背景

自社ではFlask製のSingle Page Application（SPA）でダッシュボードを運用している。今回の整備では以下を実施：

- 退役サービス3個を削除
- 新規サービス3個（TechsFree Shop、TechsFree ERP、会計AI）を追加
- URLが変わったサービスの修正

## 問題：IPアクセス時のHostヘッダー

内部IPで直接サービスにアクセスしようとしたところ、nginxが正しいvhostに振り分けてくれない問題が発生した。nginx側の構成はサブドメインを前提にしており、IPアクセスの場合は`server_name`にマッチしない。

### 解決策：default serverにlocationを追加

```nginx
server {
    listen 80 default_server;

    location /shop/ {
        root /www/apps/techsfree-shop;
        try_files $uri $uri/ /shop/index.html;
    }

    location /erp/ {
        proxy_pass http://127.0.0.1:3101/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

`default_server`のvhostにパスベースのlocationを追加するのがポイント。

## 静的ファイルとリバースプロキシの使い分け

| サービス | 構成 | nginx設定 |
|---------|------|-----------|
| TechsFree Shop | Viteビルド済み静的ファイル | `root` ディレクティブ |
| TechsFree ERP | バックエンドAPI + フロント | `proxy_pass` |

**`proxy_pass`のtrailing slashに注意：**

```nginx
# NG: /erp/ のプレフィックスがバックエンドにも渡ってしまう
location /erp/ {
    proxy_pass http://127.0.0.1:3101;  # trailing slash なし
}

# OK: /erp/ を取り除いてバックエンドに転送
location /erp/ {
    proxy_pass http://127.0.0.1:3101/;  # trailing slash あり
}
```

## systemdとpm2の共存管理

同じサービスが2つの環境で動いている状態になったとき、変更が反映されない問題が発生。原因はsystemdが古いプロセスをkillせずに新しいプロセスを起動していたため。

```bash
# 古いプロセスを確認してkill
ps aux | grep server.py
kill <PID>
systemctl --user restart task-dashboard.service
```

## タスク管理機能のバグ修正

JSONフィールド名の不整合によるバグ：

```python
# バグあり: done で参照していた
active_tasks = [t for t in tasks if not t.get('done')]

# 正しい: completed が正しいフィールド名
active_tasks = [t for t in tasks if not t.get('completed')]
```

## まとめ

1. **nginxのdefault serverにlocationを追加**してIPアクセスに対応
2. **静的ファイルとリバースプロキシを用途で使い分け**（SPAには`try_files`必須）
3. **`proxy_pass`のtrailing slashに注意**（有無でパスの扱いが変わる）
4. **デプロイ後の重複プロセス確認**を習慣化
5. **データフィールド名の一貫性**はプロジェクト全体で守る
