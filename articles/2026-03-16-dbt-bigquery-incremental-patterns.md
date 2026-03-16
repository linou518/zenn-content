---
title: "dbt + BigQuery 本番データパイプライン設計パターン：Incremental戦略・Partition最適化・Data Contract実践"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "自動化"]
published: true
---

---
title: "dbt + BigQuery 本番データパイプライン設計パターン：Incremental戦略・Partition最適化・Data Contract実践"
date: 2026-03-16
author: "学習君"
category: "data-engineering"
tags: ["dbt", "BigQuery", "データエンジニアリング", "Incremental Model", "Data Contract", "ETL", "データパイプライン"]
readingTime: 10
summary: "2TBのテーブルを全量スキャンして数MBの新データだけ取得、請求額が急増、ダッシュボードが遅延——聞き覚えがある？dbtのBigQuery向け3大Incremental戦略、Partition+Clusteringの最適化、そしてData ContractによるCIでのschema変更検知を実践的に解説する。"
lang: "ja"
draft: false
---

ナイジェリアのフィンテック企業Kudaが直面した問題は、あなたのチームでも起きているかもしれない。

データ量が増加した後、フルリフレッシュモデルの実行時間が数分から数時間に膨れ上がった。さらに痛かったのは、毎時間2TBのテーブルをスキャンして、取得するのはわずか数MBの新データだけだったこと。BigQueryは処理量課金のため、月次請求額が急増。ダッシュボードは遅延し、金融業務のリアルタイム性を全く満たせなくなった。

これはアーキテクチャ設計の失敗ではなく、データパイプラインパターンの選択ミスだ。

**本記事の核心：dbtでBigQuery上に本番グレードの増分データパイプラインを構築し、Data Contractでschema変更による連鎖崩壊を防ぐ。**

---

## なぜフルリフレッシュはスケールしないのか？

フルリフレッシュのロジックはシンプルだ：毎回実行時にテーブル全体を再計算する。

データ量が少ないうちは問題ない。しかしデータには成長曲線がある：

- 今日100万行：30秒
- 1年後1000万行：5分
- 2年後1億行：1時間、コスト×100

BigQueryのオンデマンド課金はスキャン量に応じて課金されるため、大きなテーブルへのフルリフレッシュはコストのブラックホールになる。

**解決策**：dbt Incremental Models——新規または変更されたレコードのみを処理し、処理量とコストを大幅に削減する。

---

## 3大Incremental戦略：シナリオが選択を決める

dbtはBigQuery上で3つのIncrementalストラテジーをサポートする。「どれが最良か」ではなく「データの特性に何が合うか」だ。

### 戦略①：Append（追記）

**追加のみで変更のないデータ**に適している：ログストリーム、クリックイベント、支払い記録（履歴を修正しない場合）。

```sql
{{ config(materialized = 'incremental') }}
SELECT * FROM {{ source('core', 'transactions') }}
{% if is_incremental() %}
WHERE transaction_date > (SELECT MAX(transaction_date) FROM {{ this }})
{% endif %}
```

**注意**：BigQueryはネイティブ`append`戦略をサポートしないため、タイムスタンプフィルターで手動実装が必要。上流が重複データを送信する可能性がある場合、重複排除ロジックが必須。

### 戦略②：Insert Overwrite（パーティション上書き）★ BigQuery推奨

**日付パーティション化された集計テーブル**に適している。日次データは変化するが、特定の日付範囲に限定される。

```sql
{{ config(
  materialized = 'incremental',
  incremental_strategy = 'insert_overwrite',
  partition_by = { 'field': 'transaction_date', 'data_type': 'date' }
) }}
```

パーティション全体を置換するため、行単位のマージを避けられる。遅延データも当日パーティションの再計算でキャプチャできる。

**BigQuery環境でコストパフォーマンスが最も高い入門パターン。**

### 戦略③：Merge（アップサート）

**主キーを持つディメンションテーブル**に適している。時間とともに変化するデータ——顧客プロフィール、口座残高、ユーザーステータス。

```sql
{{ config(
  materialized = 'incremental',
  incremental_strategy = 'merge',
  unique_key = 'customer_id'
) }}
```

dbtはBigQuery上でネイティブMERGE文にコンパイルする。`unique_key`の選択が重要——間違えると重複データが発生するため、本当にユニークなフィールドをビジネスサイドと確認すること。

---

## BigQuery専用チューニング：Partition + Clustering

Incremental戦略は「どれだけのデータを処理するか」の問題を解決する。PartitionとClusteringは「いかに効率よくスキャンするか」の問題を解決する。

### Partition（パーティション）設定

```sql
{{ config(
  materialized = 'table',
  partition_by = {
    "field": "created_at",
    "data_type": "timestamp",
    "granularity": "day"
  },
  require_partition_filter = true,
  partition_expiration_days = 90
) }}
```

重要な設定の意味：

- **`require_partition_filter = true`**：すべてのクエリにパーティションフィルター条件を強制。エンジニアがうっかり全テーブルスキャンを書くのを防ぐガードレール。100GB以上のイベントテーブルには必須。
- **`partition_expiration_days = 90`**：90日以上のパーティションを自動削除、手動クリーンアップ不要でストレージコストを直接削減。
- パーティションフィルターはリテラル値を使用すること。サブクエリを使うとpartition pruningが機能せず、全テーブルスキャンと同等になる。

### Clustering（クラスタリング）

パーティションの上に、高頻度フィルターカラム（`user_id`、`event_type`、`status`）でソートし、WHERE条件と集計操作を高速化する。

**最適な組み合わせ**：日付パーティション + user_idクラスタリング = 「特定ユーザーの特定日の行動」クエリ時に極めて小さいデータサブセットのみスキャン。

---

## Data Contract：CI/CDでschema破壊を事前阻止

すべてのデータチームが経験したことのあるシナリオがある：

プロダクトチームが`user_status`フィールドを`STRING`から`INTEGER`に変更した。データエンジニアリングチームへの通知なし。翌朝、dbtモデルがクラッシュ、LookerダッシュボードがすべてRED、MLパイプラインが断線。原因特定に3時間かかった。

**Data Contractの目的は、これが起きる前に止めることだ。**

```yaml
models:
  - name: fct_user_activity
    contract:
      enforced: true
    columns:
      - name: user_status
        data_type: string
        constraints:
          - type: not_null
      - name: user_id
        data_type: integer
        constraints:
          - type: not_null
          - type: unique
```

`contract.enforced: true`を有効にすると、dbtはコンパイル時にモデル出力のschemaがYAML定義と一致するかを検証する。一致しなければCI/CDが失敗し、破壊的変更は本番に出られない。

これは**データプロデューサーとコンシューマーの間の正式な約定**であり、schema変更を「サプライズ」から「制御可能な対話」に変える。

---

## パイプライン階層設計：Medallionパターン

```
Raw Layer → Staging Layer → Intermediate Layer → Mart Layer
（生データ）  （軽度クレンジング）  （ビジネスロジック）  （分析向け）
```

推奨ディレクトリ構造：

```
models/
  staging/       # sourceシステムごとにサブディレクトリ
  intermediate/  # クロスsource join、ビジネス中間層
  marts/         # ビジネスドメイン別（finance/product/operations）
```

各層のマテリアライゼーション推奨：
- Staging：`view`または`ephemeral`（ストレージ不使用、クエリ時に計算）
- Intermediate：`table`（再利用頻度の高い中間結果）
- Mart：`incremental`（最終的なクエリ向け大テーブル）

---

## まとめ

データパイプラインのスケールアップは、より大きなマシンではなく、よりスマートなデータ処理戦略で実現する。

dbtのIncremental Modelsは「必要なデータのみ処理する」問題を解決し、BigQueryのPartition + Clusteringは「効率的なスキャン」の問題を解決し、Data Contractは「クロスチームのschema変更による連鎖崩壊」の問題を解決する。

この3つを組み合わせて初めて、BigQuery上の現代的なデータプラットフォームが本来の力を発揮できる。

---

*データソース：dbt Developer Blog（Kudaフィンテック事例）；dbt BigQuery Configs公式ドキュメント；Data Contracts with dbt+BigQuery実践*

