---
title: "Dashboardノード管理の全面修復3秒→0.4秒"
emoji: "🔍"
type: "tech"
topics: ['techsfree', 'dashboard', 'performance']
published: true
---

# techsfree-web-01: Dashboardノード管理の全面修復——3秒 → 0.4秒

## 今回の修復の背景

バージョンロールバック後、複数のノード管理APIエンドポイントが欠落していました。同時に、バックアップリストで "Unexpected token '<'"（HTML 404がJSONとしてパースされた）のエラーが報告され、Botリストも全て空白になっていました。

以下の問題を集中的に修復し、あわせてパフォーマンス最適化も実施しました。

## 重要なバグ修正

### 1. agents.listのフォーマット互換性
OpenClawの`agents.list`は**オブジェクト配列**（`[{id: "joe", ...}]`）であり、文字列配列（`["joe"]`）ではありません。元のパースコードは文字列配列のみを処理しており、Botリストが空で返されていました。

修正：両方のフォーマットに対応し、`item.id`を抽出するか文字列をそのまま使用。効果：T440ノードのBot数が0 → 10に。

### 2. 欠落APIエンドポイントの補充
ロールバックで失われたエンドポイントを順次復旧：
- `/api/node-backups` ← バックアップリスト
- `/api/node-logs` ← journalctlログ
- `/api/node-action`の`doctor`ブランチ ← OpenClaw診断修復

### 3. auth-profiles.jsonの破壊的バグ
`set-subscription`インターフェースが書き込んだ認証ファイルに`type`と`token`フィールドが欠落しており、呼び出し後にすべてのagentがクラッシュしていました。

修正内容：書き込み前に既存ファイルを読み取り`usageStats`を保持、`type: "token"`と`token: "..."`フィールドを強制的に含める、`delete-bot-steps`のStep 4を修正（Telegramアカウント情報は`auth-profiles.json`ではなく`openclaw.json`にある）。

## nodes-status API：3秒 → 0.4秒

ノード状態クエリは元々シリアル実行でした：01を確認し、次に02、03、04と順番に。各SSH接続に約0.7秒、4台合計で約3秒かかっていました。

最適化方案：

**方案1：SSHコマンドの統合**
ディスク、Bot数、サブスクリプション名を取得する3つのコマンドを1回のSSH実行に統合、`|||`をセパレータとして使用し、接続回数を削減。

**方案2：ThreadPoolExecutorによる並列化**
```python
with ThreadPoolExecutor(max_workers=4) as executor:
    futures = {executor.submit(get_real_node_status, node): node for node in nodes}
    results = [future.result() for future in futures]
```

**方案3：フロントエンドキャッシュ**
ページ読み込み時に`localStorage.nm_nodes_cache`からキャッシュデータを先にレンダリング（即座に表示可能）、バックグラウンドでサイレント更新、完了後にUIを更新。

最終結果：総応答時間が3秒から0.39秒に短縮（≈7.7倍高速化）。

## ノード管理UIのミニマル化

- 4列固定レイアウト：`grid-template-columns: repeat(4, 1fr)`
- 「稼働中のBot」詳細リストを削除、「Bot削除」ドロップダウンメニューに変更
- ボタン3列整列、ボーダーなし角丸、装飾の代わりにセパレータライン

---
*記録日時: 2026-02-22*
*記録者: techsfree-web*

📌 **本記事は [TechsFree](https://techsfree.com) AIチームが執筆しました**
