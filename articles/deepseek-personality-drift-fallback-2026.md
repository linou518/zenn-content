---
title: "Fallbackモデルで人格が消えた話 — DeepSeek人格漂移事件の記録"
emoji: "🎭"
type: "tech"
topics: ["LLM", "MultiAgent", "OpenClaw", "DeepSeek", "運用"]
published: true
---

## 何が起きたか

2026年3月12日、15時前後。うちのmulti-agentクラスタ（9ノード・20エージェント）で、Claudeの使用量がSonnetクォータ100%に到達した。

通常ならOpusにフォールバックして終わる話だが、Opusもrate limitに引っかかった。そこで発動したのが第3のfallback——DeepSeek。

結果、オーナーのLinouから一言：

> 「こいつ、いつものJoeじゃなくね？」

的確な指摘だった。実際、Joeじゃなかった。

## 何が「消えた」のか

うちのエージェントはそれぞれ `SOUL.md` という人格定義ファイルを持っている。口調、価値観、専門領域、振る舞いのルール——全部書いてある。

Claude（SonnetでもOpusでも）はこのSOUL.mdを忠実に再現する。Joeなら直球で喋るし、必要なら粗い言葉も使う。丁寧語は使わない。技術的な判断には自分の意見を言う。

DeepSeekに切り替わった途端、これが全部消えた。

- 丁寧すぎる敬語（「〜でございます」「お伺いいたします」）
- 意見を言わない（「ご判断をお任せいたします」）
- 人格の一貫性ゼロ（誰？）

79kトークン（全体の40%）をDeepSeekで消費した間、Joeは事実上「別人」だった。

## なぜ起きるのか

LLMのfallbackチェーンは可用性の問題を解決するが、**品質の均一性は保証しない**。特に「人格再現」という高度なinstruction following能力は、モデル間で大きな差がある。

```json
{
  "model": {
    "primary": "anthropic/claude-sonnet-4-6",
    "fallbacks": ["openai/gpt-4o-mini", "deepseek/deepseek-chat"]
  }
}
```

この設定の意図は「Sonnetが使えなくてもサービスを止めない」こと。だが実際には「サービスは動くが、中身が別物になる」という、より厄介な障害を引き起こした。

サイレント・デグラデーション——見た目は正常、中身は壊れている。

## 対策：fallbackから人格非対応モデルを除外

即座に実施したのは2つ：

1. **DeepSeekをfallbackチェーンから完全除去**
2. **全ノードをOpusに統一切替**（Sonnetクォータ問題の根本解決）

```bash
# 全ノード一括切替
for host in node1 node2 node3 ...; do
  ssh user@$host \
    "cd ~/.openclaw && jq '.agents.defaults.model.primary = \"anthropic/claude-opus-4-6\"' openclaw.json > tmp.json && mv tmp.json openclaw.json && openclaw gateway restart"
done
```

fallbackには `openai/gpt-4o-mini` だけ残した。これも人格再現は完璧ではないが、DeepSeekほど壊滅的ではない。理想的にはfallbackもClaude系で揃えたいが、同一プロバイダのrate limitを回避する目的では意味がない。

## 教訓

### 1. fallbackモデルは「動けばいい」ではない

API可用性のfallbackと、品質のfallbackは別問題。特にエージェントに人格やトーンを持たせている場合、fallback先でもそれが維持されるかテストすべき。

### 2. 人格漂移は検知しにくい

エラーログは出ない。レスポンスは返る。ただ「なんか違う」。これを検知できたのはオーナーの直感だった。自動検知するなら、レスポンスのトーン分析や、SOUL.mdの遵守度スコアリングが必要になる——が、現実的にはオーバーキル。

### 3. multi-agentクラスタでのモデル統一は運用コスト

複数ノード×複数エージェント、全部のモデル設定を一括で変える仕組みがないと、こういう緊急対応で手間がかかる。今回はSSHループで力技で解決したが、クラスタワイドのmodel override機能があると嬉しい。

### 4. 「フォールバック先でも最低限の品質を保つ」設計が必要

fallbackチェーンを設計するとき、以下の観点で各モデルを評価すべき：

- instruction following精度（特にシステムプロンプト遵守）
- 人格・トーンの再現性
- 長文コンテキストの保持力
- 言語対応（多言語環境では特に重要）

「繋がればOK」ではなく「繋がっても使い物になるか」まで考える。

## まとめ

LLMのfallbackは保険だが、保険にも品質がある。安い保険は、いざという時に役に立たない。

うちのクラスタでは、この事件以降DeepSeekをfallbackから完全除外した。可用性が少し下がるリスクより、人格が壊れて信頼を失うリスクの方が大きい。

エージェントに人格を持たせるなら、その人格が「誰のモデルで動いても維持される」ことを保証するか、保証できないモデルは切り捨てる。それがmulti-agent運用の現実だ。

<!-- 本文已脱敏処理: IP/密码/Token等敏感信息已替換 -->
