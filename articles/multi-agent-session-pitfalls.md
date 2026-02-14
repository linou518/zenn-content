---
title: "マルチAIエージェントのセッション管理で踏んだ落とし穴と解決策"
emoji: "🔧"
type: "tech"
topics: ["ai", "multiagent", "nodejs", "debug", "automation"]
published: true
---

## はじめに

複数のAIエージェントを1つのゲートウェイで運用する「マルチエージェント構成」は、業務自動化の強力なアプローチです。しかし、セッション管理には意外な落とし穴があります。

本記事では、実運用中に遭遇した**セッションパス検証のバグ**と**認証プロファイルのフォーマット問題**について共有します。

## 問題1: セッションパス検証エラー

### 症状

エージェントにメッセージを送信すると、以下のエラーが発生し全エージェントが応答不能に：

```
handler failed: Error: Session file path must be within sessions directory
```

### 調査

ソースコードにデバッグログを追加し、変数値を可視化：

```
sessionsDir = /path/to/agents/main/sessions
candidate = /path/to/agents/agentX/sessions/xxx.jsonl
```

セッションファイルは正しいエージェントディレクトリに作成されるが、**読み取り検証時にデフォルトの main エージェントにフォールバック**していた。

### 解決策

パス検証関数にワークアラウンドを適用：

```javascript
if (path.isAbsolute(trimmed)) {
  const m = trimmed.match(/^(.*\/agents\/[^\/]+\/sessions)\//);
  if (m) effectiveBase = m[1];
}
```

## 問題2: 認証プロファイルのフォーマット不備

問題1を修復後に表面化。各エージェントの auth-profiles.json に `version`、`type`、`lastGood` フィールドが不足していた。

**正しいフォーマット:**
```json
{
  "version": 1,
  "profiles": {
    "anthropic:default": {
      "type": "token",
      "provider": "anthropic",
      "token": "sk-..."
    }
  },
  "lastGood": {
    "anthropic": "anthropic:default"
  }
}
```

## 教訓

1. **問題は重なる** — 一つ修正したら次の層が見える
2. **変数値の可視化が最強** — console.error で実値を出力
3. **リモートパッチは慎重に** — SSH越しのワンライナーは引用符問題を起こす。ファイル転送→実行が安全
