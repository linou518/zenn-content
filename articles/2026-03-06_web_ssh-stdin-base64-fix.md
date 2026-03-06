---
title: "SSH経由でファイルを書き込むとき、base64の改行で詰まった話"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "自動化"]
published: true
---

# SSH経由でファイルを書き込むとき、base64の改行で詰まった話

Flask + SSH で設定ファイルを書き込む機能を実装していたら、特定ノードだけエラーになるバグに遭遇した。原因は `base64` コマンドのデフォルト動作にあった。実装の顛末と、より堅牢な代替手法をまとめる。

---

## 何を作っていたか

家庭内LANに複数のLinuxノードを置いて、Flaskベースの管理ダッシュボードから各ノードの設定ファイル（`auth-profiles.json`）を書き換える機能を実装していた。ローカルノードは直接ファイルを書けるが、リモートノードはSSH越しに書き込む必要がある。

当初の実装はこうだった。

```python
import base64, subprocess

content = json.dumps(auth_data, indent=2)
encoded = base64.b64encode(content.encode()).decode()

ssh_cmd = f"echo '{encoded}' | base64 -d > /path/to/auth-profiles.json"
subprocess.run(["ssh", f"user@{host}", ssh_cmd], ...)
```

一見きれいに見えるが、これが落とし穴だった。

---

## 症状：特定ノードだけ失敗する

テストしていると、`infra`（ローカル）は成功するのに、`joe`、`work-a` などのリモートノードでは書き込みが失敗する。エラーメッセージは `base64: invalid input` や、ファイルが空になる、JSON が壊れるなど。

ノードごとに環境が違うのか？ SSHの設定か？ と追いかけていくと、実は原因はPython側にあった。

---

## 原因：base64のデフォルト改行挙動

Pythonの `base64.b64encode()` は改行なしのバイト列を返すが、シェルの `echo '...'` に長い文字列を渡すと、**シェルによっては途中で解釈が変わる**。さらに致命的なのが、`base64 -d` コマンド側の挙動だ。

`base64` コマンド（GNU coreutils）はエンコード時に**デフォルトで76文字ごとに改行**を挿入する。Pythonが改行なしで生成したBase64文字列をシェルの `echo` に渡すと、SSHのシェルが文字列をどう扱うかによって結果が変わる。

実際、`auth-profiles.json` のサイズがノードによって異なり、JSONが長いノードほど失敗していた。文字列が長くなるほど、シェルでの引用符エスケープや行継続の問題が表面化していた。

---

## 修正：標準入力経由でファイルを渡す

引用符問題もBase64改行問題も、根本的に回避する方法がある。**SSHの標準入力にファイル内容を流し込む**手法だ。

```python
import json, subprocess

def write_auth_profiles(host: str, path: str, auth_data: dict) -> bool:
    content = json.dumps(auth_data, indent=2).encode()

    if host == "localhost":
        # ローカルは直接書き込み
        with open(path, "w") as f:
            json.dump(auth_data, f, indent=2)
        return True

    # リモートはSSHのstdinで流す
    result = subprocess.run(
        ["ssh", f"user@{host}", f"cat > {path}"],
        input=content,
        capture_output=True,
    )
    return result.returncode == 0
```

ポイントは `input=content`。Pythonの `subprocess.run` はバイト列を標準入力に渡せる。SSH側は `cat > /path/to/file` で標準入力をそのままファイルに書き出す。

**引用符なし、Base64なし、改行問題なし。**

---

## ビフォー／アフター比較

| 項目 | 旧実装（base64 echo） | 新実装（stdin cat） |
|------|----------------------|-------------------|
| コマンド複雑さ | 高（エスケープ地獄） | 低（cat一発） |
| 文字数制限 | あり（シェル依存） | なし |
| 特殊文字耐性 | 弱い | 強い |
| デバッグしやすさ | 低い | 高い |

---

## 教訓

SSH経由でファイルを書き込むとき、`echo '...' | base64 -d` パターンはシンプルに見えて罠が多い。特に：

1. **JSONが長い**（数百バイト以上）と文字列操作がシェルの限界に当たりやすい
2. **ノード環境差異**（bashのバージョン、シェルの設定）で挙動が変わる
3. **デバッグが難しい**：SSHコマンドが長い文字列を含むとログも読みにくい

`subprocess.run(input=...)` + `cat > file` はシンプルで堅牢。base64を使う必要がない状況では、最初からこちらを選ぶべきだった。

---

## おまけ：同じパターンが他にも15箇所あった

修正後に `server.py` を `grep` したら、同じ `echo '...' | base64 -d` パターンがbot作成・設定書き込みなど15箇所に散在していた。今回は修正箇所を1関数に集約することで、将来の改修コストも下げた。技術的負債は見つけたら即対処、が原則。

---

**タグ案**: `#Python` `#SSH` `#Flask` `#DevOps` `#Linux` `#バグ修正` `#実装ノート`

