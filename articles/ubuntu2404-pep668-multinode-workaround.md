---
title: "Ubuntu 24.04 の PEP 668 に月末自動化がやられた話と、SSH経由マルチノード回避策"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "自動化"]
published: true
---

---
title: "Ubuntu 24.04 の PEP 668 に月末自動化がやられた話と、SSH経由マルチノード回避策"
emoji: "🐍"
type: "tech"
topics: ["ubuntu", "python", "automation", "infra", "troubleshooting"]
published: true
---

# Ubuntu 24.04 の PEP 668 に月末自動化がやられた話と、SSH経由マルチノード回避策

## 背景

複数のノードで動かしているインフラがある。
月末になると勤怠打刻・請求書送付・各種フォーム提出などを自動化する `monthly-endofmonth.sh` が走る設計だ。

この自動化スクリプト、3月末の実行でいきなり死んだ。

---

## 何が起きたか

エラーの表面はこうだった：

```
error: externally-managed-environment

× This environment is externally managed
╰─> To install Python packages system-wide, either use a
    distribution package (apt install python3-xxx) or create
    a virtual environment (python3 -m venv PATH)...
```

Ubuntu 24.04 で `pip install playwright` を実行したら PEP 668 に弾かれた。

**PEP 668** とは、2022年に採択されたPythonパッケージ管理の仕様変更で、Ubuntu 24.04（Noble）ではシステムPythonへの直接 `pip install` を禁止している。apt管理のPythonとpipが競合してシステムを壊す事故を防ぐための措置だ。

以前は `pip install --break-system-packages` で強制突破できたが、Ubuntu 24.04 では完全にブロックされるケースも多い。

---

## 本当の根本原因：Phase 0 と本番実行のノードが違った

PEP 668 自体は既知の話だ。問題は別にあった。

Phase 0（事前チェック）は **infra-server**（Python環境整備済みノード）で実行していた。infra-server には playwright・openpyxl・google-auth など必要なPythonライブラリがすでに整備されている。

だが本番の月末処理は **Ubuntu 24.04 の別ノード**で実行する設計だった。

**結果：Phase 0 は全部グリーン → 本番でいきなり爆発。**

こういう環境差はマルチノード構成でよく起きる。「俺の環境では動く」のインフラ版だ。

---

## 取った解決策

**Python処理をSSH経由で infra-server に投げる**という方式にした。

`monthly-endofmonth.sh` の該当箇所を以下のように変更：

```bash
# Before: Ubuntu 24.04 ノードでローカル実行（PEP 668に弾かれる）
python3 /mnt/shared/scripts/make_timesheet.py

# After: SSH経由で infra-server の Python環境を使う
ssh user@192.168.x.x "python3 /mnt/shared/scripts/make_timesheet.py"
```

infra-server は家庭内LANで固定IPを持ち、スクリプト本体は NFS (`/mnt/shared/`) で共有されているため、どちらのノードから実行しても同じファイルを参照できる。

スクリプト修正後のサマリー：

| タスク | 結果 |
|--------|------|
| Timesheet xlsx生成 | ⚠️ SameFileError（同日2回実行が原因、ファイル自体はOK） |
| HajimariWORKS打刻 | ✅ 21日分全保存成功 |
| Pasture NOB DATA | ✅ 全日入力完了 |
| Pasture オプテージ | ✅ 全日入力完了 |

---

## 教訓

### 1. Phase 0 チェックは本番実行ノードで行う

「どこかで動いた」は「どこでも動く」じゃない。依存関係・Python環境・ファイルパスはノードごとに異なる。Phase 0 チェックは実際に本番処理を動かすノード上で実行すべきだった。

### 2. Ubuntu 24.04 のシステムPython環境を整備するより移送する

PEP 668 に立ち向かうより、整備済みの環境に仕事を持っていくほうが安全で速い。

`venv` を使う方法もあるが、マルチノード・NFSマウント環境では venv パスの管理が面倒になる。SSH越しに整備済みノードへ投げるほうがシンプルだった。

### 3. 月末処理は余裕を持ってテストする

月末1日前に手動テストしても遅い。来月は Phase 0 を月中（25日あたり）に `--dry-run` 付きで一度回す仕組みを入れる予定だ。

---

## おまけ：SameFileError のバグ

今回のデバッグ中に副産物として発見したバグ。`make_timesheet.py` で同じ日に2回実行すると：

```
shutil.SameFileError: '/mnt/shared/.../timesheet.xlsx' and '/mnt/shared/.../timesheet.xlsx' are the same file
```

`shutil.copy()` の宛先に同じパスを渡していた。既存ファイルの上書き前に存在チェックが必要だった。来月の再実行時に修正する。

---

## まとめ

- Ubuntu 24.04 は PEP 668 でシステム Python への pip install を拒否する
- マルチノード構成では「どこで動くか」を意識した設計が必要
- 整備済み環境を SSH 経由で使う方式はシンプルかつ有効
- 月末自動化は余裕を持った事前検証が必須

<!-- 本文已脱敏处理: 内网IP/ホスト名等の内部情報は置換済み -->

