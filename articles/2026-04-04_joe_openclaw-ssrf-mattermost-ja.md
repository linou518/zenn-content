---
title: "OpenClaw 2026.4.2 全集群アップグレードで踏んだ SSRF 地雷と直し方"
emoji: "💣"
type: "tech"
topics: ["openclaw", "mattermost", "ssrf", "infrastructure", "devops"]
published: true
---

2026-04-04 は、うちの OpenClaw 集群を Anthropic 系から `openai-codex/gpt-5.4` へ切り替える日になった。きっかけは Anthropic 側の方針変更で、第三方 harness から subscription quota を使えなくなったこと。3月の利用量をざっくり見積もると、全 Opus 運用で月 4,700 ドル級、Sonnet でも 945 ドル級。さすがに笑えないので、既存の ChatGPT Plus で回せる Codex 系へ寄せる判断をした。

ただ、モデル切替だけでは終わらなかった。`gpt-5.4` を安定して使うには OpenClaw を 2026.3.3 以降へ上げる必要があり、実際には検証ノードで 2026.4.2 を先に確認してから、最終的に 13 ノードへ横展開した。

ここで一番ハマったのが **SSRF 保護**だ。

## 症状：アップグレード後に bot が急に黙る

2026.4.2 では Mattermost への内部接続がデフォルトで弾かれる。症状は分かりやすくて、アップグレード後に bot が急に黙る。ログには以下が出る：

```
SsrFBlockedError: Blocked hostname or private/internal/special-use IP address
```

最初は「Mattermost 側の死活か？」と思うが、原因はネットワーク到達性ではなく **OpenClaw 側の設定**だった。

## 罠が二段ある

### 罠①：型が違うと通らない

`allowPrivateNetwork` は **boolean の `true` でないと通らない**。

- ❌ 文字列 `"true"`
- ❌ Python 風の `True`
- ✅ boolean `true`

あるノードではまさにそれを踏んで、複数 agent が全部再接続できなくなった。

### 罠②：設定場所が違うと無効

設定場所が **Mattermost 全体のトップレベルではなく、各 account の内部** でないと効かない。

あるノードでは `allowPrivateNetwork: True` が Mattermost の上位に置かれていて無効化されていた。

つまり「**値の型**」と「**置き場所**」の両方が正しくないと機能しない。

## 正しい設定例

```yaml
# ❌ ダメな例（トップレベルに置いている）
mattermost:
  allowPrivateNetwork: true   # ← ここに置いても無効
  accounts:
    - token: <BOT_TOKEN>
      url: http://192.168.x.x:8065

# ✅ 正しい例（各 account の内部に置く）
mattermost:
  accounts:
    - token: <BOT_TOKEN>
      url: http://192.168.x.x:8065
      allowPrivateNetwork: true   # ← ここが正解
```

## 修正手順

各ノードで以下を実行：

```bash
# 設定を更新（--strict-json で型を保証）
openclaw config set mattermost.accounts[0].allowPrivateNetwork true --strict-json

# gateway 再起動
openclaw gateway restart

# 動作確認（新規会話で bot が応答するか）
```

:::message
`config.patch` 相当の感覚で触るより、`openclaw config set <path> <value> --strict-json` を前提にしたほうが型ミスを防げる。
:::

## 今回の展開規模

**13 ノード**を 2026.4.2 + `openai-codex/gpt-5.4` へ更新し、全ノードで SSRF 設定も修正した。

アップグレード自体は通っても、Mattermost bot が無反応なら「バージョンは上がったが設定で死んでいる」可能性をまず疑ったほうが早い。

## 副産物：モデル切替は新規セッションから

モデル切替は **既存セッションには効かず、新規セッションから反映**される。

運用で切替をやるなら、以下を checklist に入れるべき：

1. バージョン確認（`openclaw --version`）
2. SSRF 設定修正（各 account 配下に boolean `true`）
3. gateway 再起動（`openclaw gateway restart`）
4. **新規会話**での応答確認

## まとめ

今回の教訓はこれに尽きる。

> アップグレード時に怖いのは派手なクラッシュより、**"起動しているのに bot がしゃべらない" 系の半壊**だ。

SSRF 保護はセキュリティ上必要な機能だが、内網の Mattermost を使っている環境では必ず引っかかる。設定の「型」と「場所」の両方を確認すること。

<!-- 本文已脱敏处理: IP地址/Token等敏感信息已替换 -->
