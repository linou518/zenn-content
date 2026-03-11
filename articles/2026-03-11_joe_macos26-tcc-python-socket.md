---
title: "macOS 26 (Tahoe) でPythonのsocket接続がブロックされた話と LaunchDaemon で解決した記録"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "自動化"]
published: true
---

# macOS 26 (Tahoe) でPythonのsocket接続がブロックされた話と LaunchDaemon で解決した記録

## TL;DR

macOS 26 (Tahoe) 環境で、Homebrewでインストールした Python から WebSocket 接続を試みると `errno 65 (No route to host)` が発生した。`curl` は通るのに Python だけ失敗する謎現象。原因は **TCC (Transparency, Consent, and Control)** の制限で、回避策は LaunchAgent → LaunchDaemon への移行だった。

---

## 経緯

OpenClaw（AI アシスタントフレームワーク）を中古 Mac mini (M1, macOS 26.0.1 Tahoe) に自動インストールする販売用セットアップスクリプトを作っていた。LINE Bridge として WebSocket リレーサーバーに接続する Python スクリプトを LaunchAgent として登録したのだが、Gateway が起動しない。

ログを見ると：

```
socket.connect_ex(('192.168.3.34', 3500)) → errno 65: No route to host
```

192.168.3.34 は同一 LAN 内の別ホスト。ping は通る。curl は通る。なのに Python だけ失敗する。

---

## デバッグ過程

### まず疑ったこと（全部ハズレ）

**1. IPアドレス/ポートの問題かと思った**

```bash
ping 192.168.3.34        # OK
curl http://192.168.3.34:3500  # OK
nc -zv 192.168.3.34 3500       # OK
```

全部通る。Python の問題に絞られた。

**2. 外部 IP（ヘアピンNAT）の問題かと思った**

当初 `wss://linebot.techsfree.com`（外部ドメイン）に接続しようとしていた。Mac mini からルーターの外部 IP への接続がルーターによってブロックされるケースはある（ヘアピンNAT非対応）。これは実際に問題だったので内部 IP に変更したが、それでもまだ失敗した。

**3. ファイアウォールかと思った**

```bash
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate
# Firewall is disabled.
```

無効だった。

### 犯人は TCC だった

`sudo python3 script.py` で実行すると接続成功した。

LaunchAgent（ユーザーレベル）で実行すると失敗。root で実行すると成功。

macOS の **TCC（Transparency, Consent, and Control）** は、サードパーティアプリのシステムリソースへのアクセスを管理する仕組みだ。macOS 26 Tahoe では、Homebrew でインストールした Python がネットワーク接続時に TCC の審査対象となり、LaunchAgent（ユーザー空間）での実行時には一部の接続がブロックされる。

検証：

```bash
# LaunchAgent（ユーザーレベル）経由 → FAIL
launchctl load ~/Library/LaunchAgents/com.openclaw.linebridge.plist
# → errno 65: No route to host

# 直接 sudo 実行 → OK
sudo /opt/homebrew/bin/python3 line_bridge_v2.py
# → Connected to relay server ✓

# subprocess.run(['curl', ...]) でも OK
# → curl はシステムバイナリのため TCC バイパス
```

---

## 解決策：LaunchDaemon に昇格

LaunchAgent（`~/Library/LaunchAgents/`）はユーザー空間で動く。LaunchDaemon（`/Library/LaunchDaemons/`）はシステムレベル（root）で動き、TCC をバイパスする。

**変更前（LaunchAgent）:**
```xml
<!-- ~/Library/LaunchAgents/com.openclaw.linebridge.plist -->
<key>Label</key>
<string>com.openclaw.linebridge</string>
```

**変更後（LaunchDaemon）:**
```xml
<!-- /Library/LaunchDaemons/com.openclaw.linebridge.plist -->
<key>Label</key>
<string>com.openclaw.linebridge</string>
<key>UserName</key>
<string>openclaw</string>  <!-- root 権限だが一般ユーザーとして実行 -->
```

`UserName` キーを指定することで、root 権限のデーモンとして起動しつつ、ファイルの読み書きは `openclaw` ユーザーとして行える。

```bash
# 登録コマンド
sudo launchctl bootstrap system /Library/LaunchDaemons/com.openclaw.linebridge.plist
sudo launchctl enable system/com.openclaw.linebridge
```

結果：`Connected to relay server ✓` — 一発で解決した。

---

## macOS 26 で気をつけること（まとめ）

| 実行方法 | TCC制限 | 備考 |
|---------|---------|------|
| システムバイナリ（curl等） | なし | 常に通る |
| `sudo python3` | なし | root はバイパス |
| LaunchAgent + Homebrew Python | **あり** | errno 65 が出る |
| LaunchDaemon + UserName指定 | なし | 推奨パターン |
| subprocess.run(['curl', ...]) | なし | Python から curl を呼べば通る（応急処置） |

macOS 26 はセキュリティが強化されており、これまで問題なく動いていたスクリプトが動かなくなるケースが増えている。特に **常駐デーモン + サードパーティランタイム + ネットワーク接続** の組み合わせは要注意だ。

---

## 余談：Mac mini 販売用セットアップ自動化

この問題は「中古 Mac mini に OpenClaw をインストールして AI アシスタントとして販売する」という商品開発の過程で踏んだ。

GMK の Ubuntu 版では Clonezilla でイメージコピーができたが、Mac は Apple Silicon + SIP があるためイメージ複製が使えない。代わりにインストール自動化スクリプトを整備した：

- `factory-setup-mac.sh` — 完全自動インストール（約15分）
- `per-unit-setup-mac.sh` — 各台ごとの Token 個別化（5秒）
- `factory-qc-mac.sh` — 出荷前 QC（全22項目チェック）
- `factory-clean-mac.sh` — 出荷前クリーンアップ

今回の TCC 問題は `factory-setup-mac.sh` の LINE Bridge セクションに修正を反映済み。次の台からは自動で LaunchDaemon として登録される。

---

## タグ案

`#macOS` `#macOS26` `#Tahoe` `#Python` `#TCC` `#LaunchDaemon` `#LaunchAgent` `#ネットワーク` `#トラブルシューティング` `#OpenClaw` `#AI` `#自動化`

