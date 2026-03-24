---
title: "Pythonの`datetime`二重インポート罠——`type object 'datetime.datetime' has no attribute 'datetime'`の正体"
emoji: "🐍"
type: "tech"
topics: ["python", "flask", "datetime", "debug", "backend"]
published: true
---

FlaskのAPIサーバーにコードを追加していたとき、突然このエラーに遭遇した。

```
AttributeError: type object 'datetime.datetime' has no attribute 'datetime'
```

実際のコードはこうだった。

```python
datetime.datetime.now()
```

一見なんの問題もないように見える。だがPythonは「そんな属性はない」と言って落ちた。

## 何が起きていたか

問題の根っこはファイルの先頭にあった。

```python
import datetime
from datetime import datetime
```

この2行が同じファイルに並んでいた。

`import datetime` は `datetime` という**モジュール**を `datetime` という名前にバインドする。  
`from datetime import datetime` は `datetime` **クラス**を `datetime` という名前にバインドする。

後から書いた `from datetime import datetime` が先の `import datetime` を**上書き**する。その結果、名前 `datetime` はもはやモジュールではなくクラスを指している。

そこで `datetime.datetime.now()` を呼ぼうとすると、Pythonは「`datetime`クラスの中の`datetime`属性」を探すことになる。当然そんなものはない。だからエラーになる。

## なぜ気づかないのか

コードが長くなると、インポート行を見落とす。他のファイルからコードをコピーしてきたとき、インポート文ごと持ってくることがある。既存のインポートと衝突しているかどうかは、エラーが出るまで気づかない。

今回のケースでは、もともとあったコードが `import datetime` を使っており、後から追加したコードが `from datetime import datetime` を使っていた。どちらも「動くはず」なコードだったが、同じ名前空間に共存した瞬間に壊れた。

## 直し方

解決策は単純で、どちらか一方に統一することだ。

**パターンA: モジュールとして使う**

```python
import datetime

# 使うとき
now = datetime.datetime.now()
delta = datetime.timedelta(days=7)
```

**パターンB: クラスを直接インポートする**

```python
from datetime import datetime, timedelta

# 使うとき
now = datetime.now()
delta = timedelta(days=7)
```

どちらでもいいが、混在させてはいけない。

## 実際の修正

問題のあったコードでは `from datetime import datetime` を採用することにした。ファイル全体で `datetime.datetime.now()` と書いていた箇所を `datetime.now()` に書き換えた。

```python
# 修正前
created_at = datetime.datetime.now().isoformat()
timestamp = datetime.datetime.now().strftime('%Y%m%d-%H%M%S')

# 修正後
created_at = datetime.now().isoformat()
timestamp = datetime.now().strftime('%Y%m%d-%H%M%S')
```

影響箇所は3箇所あった。全部修正してFlaskを再起動したところ、エラーは消えた。

## 教訓

Pythonの `import` は後勝ちで名前空間を上書きする。同じ名前のバインディングが複数あると、最後に書かれた方が有効になる。これはPythonの仕様通りの動作だが、長いファイルでは気づきにくい。

linterやIDEを使っていれば「この名前は上書きされています」と警告してくれることが多い。静的解析ツールをCI/CDに組み込んでおくと、こういう問題をマージ前に検出できる。

それと、コードを複数のファイルからコピー&ペーストするときは**インポート文も必ず確認する**こと。これが今回の実際の根本原因だった。

---

## まとめ

| 現象 | 原因 |
|------|------|
| `datetime.datetime` has no attribute `datetime` | `from datetime import datetime` が `import datetime` を上書きしている |
| 解決策 | どちらか一方に統一する |
| 予防策 | linter / 静的解析でインポート重複を検出する |

地味なバグだが、踏んだときのデバッグは案外時間を食う。インポート行を疑うのは後回しになりがちだからだ。同じ罠に落ちた人の参考になれば。
