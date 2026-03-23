---
title: "CSVをPostgreSQLに安全に取り込む——マルチテナントETLパイプラインの実装パターン"
emoji: "📊"
type: "tech"
topics: ["Python", "FastAPI", "PostgreSQL", "ETL", "SaaS"]
published: true
---

ユーザーが任意のCSVをアップロードして分析できるSaaSを作るとき、一番厄介なのが「スキーマが事前にわからない」問題だ。普通のRDBMSは列名と型を決めてからテーブルを作る。でもユーザーのCSVは10列かもしれないし100列かもしれない。列名も`売上金額`だったり`revenue_amount`だったり、スペース入りだったり記号が混じってたりする。

この問題に正面から向き合ったのが、最近実装したCSV分析プラットフォームのETLパイプラインだ。実装を通じて学んだパターンを整理しておく。

---

## 全体フロー

```
S3（アップロード済みCSV）
  ↓
パーサー：型推論＋カラム名正規化
  ↓
ステージングテーブル（動的生成）
  ↓
DWHテーブル（会社別スキーマ）
```

認証はAWS Cognito、ストレージはS3、DBはRDS PostgreSQL（非同期接続はSQLAlchemy + asyncpg）。バックエンドはFastAPIで、ETLはPOST `/api/etl/{upload_id}/run` で起動する。

---

## ポイント1：カラム名の正規化

ユーザーが持ってくるCSVのヘッダーは何でも来る。`売上金額`、`Revenue (JPY)`、`  col  `（前後スペース）、空文字、重複名。これをそのままSQLのカラム名にするとINJECTIONリスクや文法エラーになる。

```python
def _normalize_column_name(name: Any, index: int, seen: set[str]) -> str:
    normalized = re.sub(r"[^a-z0-9]+", "_", str(name).strip().lower()).strip("_")
    if not normalized:
        normalized = f"column_{index + 1}"
    if not normalized[0].isalpha():
        normalized = f"col_{normalized}"
    # PostgreSQL識別子の63文字制限に合わせる
    normalized = normalized[:63].rstrip("_") or f"column_{index + 1}"
    # 重複カラム名はサフィックスで解決
    candidate = normalized
    suffix = 1
    while candidate in seen:
        suffix_text = f"_{suffix}"
        base = normalized[:63 - len(suffix_text)].rstrip("_")
        candidate = f"{base}{suffix_text}"
        suffix += 1
    seen.add(candidate)
    return candidate
```

`seen: set[str]` を引き回して重複を検出し、`_1`, `_2`…とサフィックスを付ける。PostgreSQLの識別子上限63文字も厳密に守る。

---

## ポイント2：型推論

全列を文字列で読み込んでから、「これはnumericか？dateか？それともtextか？」を判定する。

```python
def _infer_column_type(series: pd.Series) -> tuple[str, pd.Series]:
    non_null_mask = series.notna()
    if not non_null_mask.any():
        return "text", series

    numeric_series = pd.to_numeric(series, errors="coerce")
    if numeric_series[non_null_mask].notna().all():
        return "numeric", numeric_series

    datetime_series = pd.to_datetime(series, errors="coerce", utc=True)
    if datetime_series[non_null_mask].notna().all():
        return "date", datetime_series

    return "text", series
```

ロジックはシンプル：「全non-null値がnumericに変換できれば数値」「全non-null値がdatetimeに変換できれば日付」「それ以外はtext」。NULL混じりの列でも `non_null_mask` でフィルタするので誤判定しない。

型マッピングはPostgreSQL側：

```python
_TYPE_MAP = {
    "numeric": "DOUBLE PRECISION",
    "date": "TIMESTAMPTZ",
    "text": "TEXT",
}
```

---

## ポイント3：テーブル名の動的生成とSQLインジェクション対策

動的DDLでは生SQLを組み立てるしかない場面がある。`CREATE TABLE {table_name} (...)` の部分はプレースホルダが使えない。ここが一番怖い。

対策：**ホワイトリスト検証を挟んでからSQLに埋め込む**。

```python
_VALID_SQL_IDENTIFIER = re.compile(r"^[a-z][a-z0-9_]*$")

def _validate_table_name(name: str) -> str:
    if not _VALID_SQL_IDENTIFIER.match(name):
        raise ValueError(f"Invalid SQL table name: {name}")
    return name
```

テーブル名の生成規則：`c{company_id_hex8}_{safe_filename}_{timestamp}`。会社IDのプレフィックスでマルチテナント分離も兼ねている。

---

## ポイント4：エンコーディング自動検出

日本語CSVでありがちな文字化け問題。UTF-8を試して失敗したらlatin-1にフォールバックする。

```python
try:
    df = _read_csv(csv_bytes, "utf-8")
except UnicodeDecodeError:
    df = _read_csv(csv_bytes, "latin-1")
```

Shift-JISはlatin-1では文字化けするが、パースエラーにはならない。本当はchardetで判定するのが正確だが、対象ユーザーの多くがExcel出力CSVをアップロードするという前提で、まずはUTF-8/latin-1の2択で運用している。

---

## ポイント5：ステータス管理で処理の見える化

ETL処理は非同期で時間がかかる。ユーザーがリロードしても「今どこ？」がわかるよう、`uploads`テーブルにステータスカラムを持たせる。

```
pending → processing → completed
                     ↘ failed
```

ETL開始時に`PROCESSING`にセットし、完了・失敗時に対応するステータスに更新する。フロントエンドはポーリングまたはWebSocketでステータスを監視する設計。

---

## まとめ

「ユーザーが自由にCSVを持ち込める」機能は一見シンプルに見えて、**カラム名の正規化・型推論・SQLインジェクション対策・エンコーディング対応**という地味だが重要な問題が積み重なっている。

この実装で得た教訓を一行で言うと：**「入力を信じるな、全部バリデーションしてから触れ」**。

ユーザーデータは必ず想定外の形で来る。そのときにクラッシュするかエラーを返せるかが、SaaSの品質を分ける。

---

*この実装はFastAPI + SQLAlchemy + pandas + AWS Lambda/RDS 構成のバックエンドとして使用。*
