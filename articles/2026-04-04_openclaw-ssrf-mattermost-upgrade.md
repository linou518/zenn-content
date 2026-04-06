---
title: "OpenClaw 2026.4.2 全集群アップグレードで踏んだ SSRF 地雷と直し方"
emoji: "💣"
type: "tech"
topics: ["openclaw", "mattermost", "ssrf", "infrastructure", "multiagent"]
published: true
---

2026-04-04 は、うちの OpenClaw 集群を Anthropic 系から `openai-codex/gpt-5.4` へ切り替える日になった。きっかけは Anthropic 側の方針変更で、第三方 harness から subscription quota を使えなくなったこと。3月の利用量をざっくり見積もると、全 Opus 運用で月 4,700 ドル級、Sonnet でも 945 ドル級。さすがに笑えないので、既存の ChatGPT Plus で回せる Codex 系へ寄せる判断をした。

## アップグレードの経緯

ただ、モデル切替だけでは終わらなかった。`gpt-5.4` を安定して使うには OpenClaw を 2026.3.3 以降へ上げる必要があり、実際には family ノードで 2026.4.2 を先に検証してから、最終的に 13 ノードへ横展開した。

対象ノードは以下の 13 台：

- joe / jack / infra / work-a / work-b / apps / family / personal / pi4
- agent01 / agent02 / agent03 / macmini

全ノードを 2026.4.2 + `openai-codex/gpt-5.4` へ更新した。

## ハマったのは SSRF 保護

ここで一番ハマったのが **SSRF 保護**だ。2026.4.2 では Mattermost への内部接続がデフォルトで弾かれる。

### 症状

症状は分かりやすくて、アップグレード後に bot が急に黙る。ログには以下のエラーが出る：

```
SsrFBlockedError: Blocked hostname or private/internal/special-use IP address
```

最初は「Mattermost 側の死活か？」と思うが、原因はネットワーク到達性ではなく **OpenClaw 側の設定**だった。

## 罠が二段ある

### 罠1: boolean の型が厳密

`allowPrivateNetwork` が **boolean の `true` でないと通らない**こと。

- ❌ 文字列 `"true"` → NG
- ❌ Python 風の `True` → NG
- ✅ boolean の `true` → OK

personal ノードではまさにそれを踏んで、health / life / learning が全部再接続できなくなった。

### 罠2: 設定の置き場所が違う

設定場所が **Mattermost 全体のトップレベルではなく、各 account の内部**であること。

family ノードでは `allowPrivateNetwork: True` が Mattermost の上位に置かれていて無効化されていた。

つまり「**値の型**」と「**置き場所**」の両方が正しくないと効かない。

## 修正方法

最終的な修正方針はシンプルで、各ノードで Mattermost の**各 account 配下**に boolean `true` を入れて gateway を再起動すること。

CLI も今回の版で少し変わっていて、以前の `config.patch` 相当の感覚で触るより、以下のコマンドを前提に考えたほうが事故が少ない：

```bash
openclaw config set <path> <value> --strict-json
```

修正後は必ず gateway を再起動する：

```bash
openclaw gateway restart
```

## 副産物として気づいたこと

モデル切替は**既存セッションには効かず、新規セッションから反映**されることも再確認できた。

運用で切替をやるなら、以下を一つの checklist に入れるべき：

1. バージョン確認
2. SSRF 設定修正（各 account 配下に boolean `true`）
3. gateway 再起動
4. **新規会話**での応答確認

## まとめ

アップグレード時に怖いのは派手なクラッシュより、**「起動しているのに bot がしゃべらない」系の半壊**だ。

今回の教訓：

- SSRF エラーは「ネットワーク障害」ではなく「設定ミス」を先に疑う
- `allowPrivateNetwork` は型と場所の両方を確認する
- モデル切替は新規セッションから有効になる

こういう半壊系が一番めんどくさい。同じ罠を踏む人が減れば幸い。

<!-- 本文已脱敏处理: IP/密码/Token等敏感信息已替换 -->
