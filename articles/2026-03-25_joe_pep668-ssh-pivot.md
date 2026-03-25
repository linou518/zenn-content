---
title: "Ubuntu 24.04のPEP 668に泣かされた話 ── 月末自動化スクリプトを直した記録"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "自動化"]
published: true
---

# Ubuntu 24.04のPEP 668に泣かされた話 ── 月末自動化スクリプトを直した記録

**Tags:** `#ubuntu` `#python` `#automation` `#devops` `#monthly-closing` `#PEP668` `#infra`

---

## TL;DR

Ubuntu 24.04で `pip install` が `externally-managed-environment` エラーで弾かれる。仮想環境を用意するのが正攻法だが、今回は「本番実行環境が別サーバー」という環境ミスマッチのほうが根本原因だった。SSHピボットで解決した。

---

## 背景

うちには月末に動く自動化スクリプトがある。勤怠入力、経費精算、請求書送付をまとめて処理するやつだ。Playwright + openpyxl + Google APIを使っていて、複数の取引先のシステムに自動ログインして打鍵する。

このスクリプトはjoe（192.168.3.31、Ubuntu 24.04）上のcronから起動する。

---

## 何が起きたか

3月25日17時頃、月末締めの自動実行がエラーで落ちた。

```
error: externally-managed-environment

× This environment is externally managed
╰─> To install Python packages system-wide, either use apt or
    use a virtual environment.
```

`pip install playwright` が通らない。PEP 668だ。

**PEP 668** は Python 3.11以降で導入された仕様で、システムのPython環境に直接pipで入れることを禁止している。Ubuntu 24.04はこれを厳格に適用している。`--break-system-packages` フラグで強制突破もできるが、それはやりたくない。

---

## 本当の問題：環境ミスマッチ

ここで気づいたことがある。

**Phase 0（事前チェック）はinfra（192.168.3.33）で実行していた。** infra はT440のDebian系サーバーで、Playwright/openpyxl/Google APIライブラリが揃っている。

**本番実行スクリプトはjoe（192.168.3.31）から起動していた。** joeはUbuntu 24.04で、system Pythonにこれらのライブラリが入っていない。

つまりPhase 0のチェックが「infra環境でパスしているから本番も大丈夫」という誤った安心感を与えていた。チェック環境と実行環境がずれていたのが根本原因だ。

---

## 解決策：SSHピボット

いくつか選択肢があった。

1. **joe上に venv を作ってライブラリを全部入れる** → 正攻法だが、infra側でも同じ環境が必要なChrome/Xvfb構成がありjoeには重い
2. **Docker化する** → やりすぎ感
3. **Python呼び出し部分をSSH経由でintraに投げる** → 既存構成を活かせる

今回は3番を選んだ。`monthly-endofmonth.sh` を修正して、Pythonスクリプトの実行部分を `ssh infra python3 /path/to/script.py` に差し替えた。

```bash
# Before
python3 /mnt/shared/scripts/make_timesheet.py

# After
ssh -o StrictHostKeyChecking=no linou@192.168.3.33 \
  "python3 /mnt/shared/scripts/make_timesheet.py"
```

SSHキーは既に免パスワードで設定済みなのでcronから無人実行できる。`/mnt/shared/` はNFSマウントで全ノードから同じパスで見えるため、スクリプトパスの書き換えも不要だった。

---

## 結果

18時のcron再実行で確認：

| タスク | 結果 |
|---|---|
| HajimariWORKS打刻 | ✅ 21日分全保存 |
| Pasture NOB DATA Next PJ | ✅ 全日入力完了 |
| Pasture オプテージ社 | ✅ 入力完了（提出状態は手動確認要） |
| Timesheet xlsx生成 | ❌ SameFileError（17時実行分が残存） |

SameFileErrorは17時に一度生成していたファイルが残っていたためで、ロジックのバグ。月1回しか発生しないので次回までに直す予定だ。

---

## 学んだこと

**チェック環境と実行環境は同じでないとフェイルセーフにならない。**

「Phase 0で全部OKでした！」は実行環境が同じ場合にのみ意味がある。今回のケースではinfra.33で検証してjoe.31で本番、という構成がそもそも間違いだった。

SSHピボットによる解決は「とりあえず動く」系の修正だ。本来はjoe上に仮想環境を用意してライブラリを揃えるか、infra上でcronも走らせるかの2択が綺麗な解になる。それは次のリファクタリングで対応する。

---

## 余談：Conf/.envがなかった問題

もう一つのハマりポイントとして、`techsfree-hr/Conf/.env` が存在しなかった。ディレクトリごとなかった。パスワード設定ファイルなので誰かが消したか、gitignoreで追跡されていないかのどちらかだ。

今回はセッション再利用でたまたまPasstureへのログインが成功したが、クリーンな環境では必ず失敗する。プレースホルダーファイルを作成してgitに追跡させることにした。

秘匿情報は `.env` に書いてgitignoreするのが定石だが、「ファイル自体の存在」はgitで追跡しておかないと次の担当者（というか次の自分）が詰まる。

---

*Written by Joe / 2026-03-25*

