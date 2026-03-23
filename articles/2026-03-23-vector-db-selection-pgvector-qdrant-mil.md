---
title: "RAGシステムの土台を選ぶ：pgvector vs Qdrant vs Milvus 完全比較2026"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "自動化"]
published: true
---

---
title: "RAGシステムの土台を選ぶ：pgvector vs Qdrant vs Milvus 完全比較2026"
date: "2026-03-23"
category: "ai"
tags: ["ベクトルDB", "pgvector", "Qdrant", "Milvus", "RAG", "AI", "LLM"]
lang: "ja"
author: "TechsFree"
draft: false
summary: "pgvector・Qdrant・Milvus——三大ベクトルDBはそれぞれ異なる哲学を持つ。選択を誤ると、後からの移行コストは非常に高い。原理から実践まで、あなたのシナリオに最適な選択を解説する。"
readingTime: 10
---

AIアプリが爆発的に普及する時代、RAG（Retrieval-Augmented Generation）を構築するすべてのチームが同じ問いに直面する：**ベクトルデータベースはどれを選ぶか？**

pgvector、Qdrant、Milvusは現在最も主流な三択だ。それぞれ「軽量統合」「高性能専用」「分散大規模」という三つの哲学を代表している。選択を誤り、ビジネスが成長してから移行するのは非常にコストが高い。

本記事では底層原理から実践比較まで、一度で正しい選択ができるよう解説する。

---

## なぜベクトルDB選定がこれほど重要か

ベクトルデータベースの核心タスクは一つだけ：**高次元空間で最も「類似した」ベクトルを素早く見つける**こと。

ユーザーが質問をすると、LLMは10万件のドキュメントから最も関連性の高い5段落を見つけなければならない。これは完全一致ではなく、近似最近傍探索（ANN）だ。選んだベクトルDBがこの検索の速さ・精度・スケールを決める。

選定が重要な理由は三つある：
1. **データ結合が深い**：ベクトル化されたデータ・インデックス構造・メタデータがすべて格納されており、移行はすべてのデータを再処理することを意味する
2. **クエリ性能の差が大きい**：同じデータ量でも、システムによってレイテンシが10倍異なることがある
3. **運用コストの差が顕著**：「PostgreSQL拡張をインストールするだけ」から「分散クラスタを維持管理する」まで、複雑さがまったく異なる

---

## 三大流派：それぞれが解決する問題

```
┌─────────────────────────────────────────────────────┐
│           ベクトルDB三大流派                          │
├──────────────┬────────────────┬──────────────────────┤
│   pgvector   │    Qdrant      │      Milvus          │
│   軽量統合   │  高性能専用    │   分散大規模          │
├──────────────┼────────────────┼──────────────────────┤
│ PostgreSQL   │ Rust実装       │ クラウドネイティブ    │
│ 拡張プラグイン│ 独立サービス  │ 独立クラスタ          │
│ 中小規模向け │ 中大規模向け   │ 超大規模向け          │
└──────────────┴────────────────┴──────────────────────┘
```

### pgvector：PostgreSQLのベクトル化拡張

pgvectorのコア思想は：**すでにPostgreSQLがある、ベクトルはデータの一種に過ぎない**。

```sql
CREATE EXTENSION vector;
CREATE TABLE documents (
  id bigserial PRIMARY KEY,
  content text,
  embedding vector(1536)
);
CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops);

-- 類似度検索
SELECT content, 1 - (embedding <=> $1) AS similarity
FROM documents
ORDER BY embedding <=> $1
LIMIT 5;
```

**メリット**：既存PostgreSQLに統合・SQL全機能利用・ハイブリッド検索が自然
**デメリット**：百万件超でパフォーマンス低下・リアルタイムマルチテナント弱い

### Qdrant：ベクトル検索専用Rustエンジン

```python
client.search(
    collection_name="documents",
    query_vector=[0.1, 0.2, ...],
    query_filter={"must": [{"key": "source", "match": {"value": "blog"}}]},
    limit=5
)
```

**メリット**：Rust実装で高性能・フィルタリング最適化・多ベクトル対応
**デメリット**：独立サービスのデプロイ必要・分散は有償

### Milvus：工業級分散ベクトルDB

**メリット**：真の水平スケール・10億件規模対応・GPU加速対応
**デメリット**：アーキテクチャ複雑・学習コスト高い

---

## コア指標比較

| 項目 | pgvector | Qdrant | Milvus |
|------|----------|--------|--------|
| **適合データ量** | <500万件 | 100万〜1億件 | 1億件以上 |
| **P99レイテンシ** | 10-100ms | 1-10ms | 5-50ms |
| **運用複雑度** | ★☆☆☆☆ | ★★☆☆☆ | ★★★★☆ |
| **フィルタリング** | 最強（SQL） | 強 | 中 |
| **水平スケール** | 限定的 | 中（有償） | ネイティブ |
| **ACID** | 完全対応 | 結果整合性 | 結果整合性 |

---

## 選定決定木

```
予想ベクトル数は？
│
├─ 100万件未満
│   └─ PostgreSQL使用中？ → YES: pgvector / NO: Qdrant
│
├─ 100万〜5000万件
│   └─ 複雑フィルタリング必要？ → YES: pgvector調整版 / NO: Qdrant
│
└─ 5000万件以上 → Milvus一択
```

---

## 本番で学んだ罠

**pgvector**：HNSWインデックスはパラメータ変更不可（再作成が必要）
**Qdrant**：Collectionのベクトル次元は変更不可・payload_indexは手動作成
**Milvus**：削除後はcompactが必要・本番でLiteモード禁止

---

## 結論：最良の選択はない、最適な選択があるだけ

- **pgvector**：PostgreSQL既存チームのデフォルト選択、百万件以下で十分
- **Qdrant**：ほとんどの独立AIアプリにおける推奨選択
- **Milvus**：大規模分散シナリオの工業級ソリューション

選定原則：まず動くものを作り、それから最適化する——将来の兆件データのために今日Milvusクラスタを構築するのは避けよう。

---

*参考：pgvector 0.7.0ドキュメント | Qdrant公式Benchmark | Milvus 2.4ドキュメント*

