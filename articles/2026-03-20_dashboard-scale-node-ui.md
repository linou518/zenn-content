---
title: "ノードが10台増えたとき、ダッシュボードのUIはどう変わるべきか"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "自動化"]
published: true
---

---
title: "ノードが10台増えたとき、ダッシュボードのUIはどう変わるべきか"
emoji: "📊"
type: "tech"
topics: ["Dashboard", "UI設計", "SPA", "ホームラボ", "フロントエンド"]
published: true
category: "frontend"
---

今日、ホームクラスターに10台のGMKミニPCが追加された。

これまで12台だったノードが一気に22台に増え、aws-expert、gcp-expert、snowflake-expertなど8つの専門家agentが新たに稼働し始めた。インフラ担当としては嬉しいニュースだが、UI担当としては一つの問題が生まれた。

**ダッシュボードの「ノード管理」画面が、想定より早くパンクする。**

---

## 問題：12台設計のUIに22台を詰め込む

TechsFreeのダッシュボードは、現在Flask + Vanilla JSで動くSPAだ。ノード管理画面はグリッドレイアウトで各ノードをカード表示している。

12台のときは問題なかった。画面に収まり、ひと目でステータスが確認できた。

しかし22台になると：
- カードが縦に長くなりすぎてスクロールが必要
- 「どのノードがどのagentを担当しているか」が一目でわからない
- フィルタなしでは探したいノードを見つけにくい

これは単純な「カードを増やす」問題ではない。**情報設計の問題**だ。

---

## 解決の方向性：グループ化と密度の調整

ノードが増えたとき、UIの対処法は大きく3つある。

### 1. グループ化（Role/Purpose別）

今回追加されたノードは明確な役割ごとに分類できる。

```
専門家グループ A: aws-expert, gcp-expert, snowflake-expert
専門家グループ B: databricks-expert, k8s-expert, iac-expert
専門家グループ C: streaming-expert, llm-expert
コアインフラ: joe, jack, infra, web
ワーカー: work-a, work-b, apps, family, personal, pi4
待機中: agent04〜agent10
```

カードを役割グループで折りたたむことで、「今日見たい情報」だけを開ける設計にできる。VMware vSphereやKubernetesのダッシュボードがNamespace/Clusterで区切るのと同じ発想だ。

### 2. 表示密度の切り替え

大規模インフラUIの定番パターン：**コンパクトモード**の提供。

- **詳細カード**: 従来通り。IPアドレス、稼働時間、agent一覧を表示
- **コンパクト行**: ホスト名・ステータスドット・IP・用途だけを1行に収める
- **タイルマップ**: ステータス色のみの小さなタイルを並べる（Nagiosスタイル）

表示切り替えはローカルストレージに保存し、次回訪問時も維持する。

### 3. フィルタ・検索の追加

22台のうち、日常的に確認するのは特定のノードだけだ。

```javascript
// シンプルな実装例
function filterNodes(query) {
  const q = query.toLowerCase();
  document.querySelectorAll('.node-card').forEach(card => {
    const name = card.dataset.hostname.toLowerCase();
    const role = card.dataset.role.toLowerCase();
    card.style.display = (name.includes(q) || role.includes(q)) ? '' : 'none';
  });
}
```

インクリメンタルサーチで十分。ElasticsearchもFuseも要らない。

---

## 実装時の注意点：APIのページング

フロントエンド側の設計が決まったとしても、バックエンドにも落とし穴がある。

22台のノードを毎回全件取得するAPIは、ノードが100台になっても同じ設計で動くか？

TechsFreeのダッシュボードは `/api/ocm/nodes` から全ノード情報を一括取得している。現在の22台では問題ないが、ステータスチェックがタイムアウトしやすいノード（オフラインのagentなど）が増えると、ページ表示が遅くなる。

対策として検討しているのは：
- **オフラインノードを非同期で後から取得**（まずオンラインだけ先に表示）
- **ステータスのキャッシュ時間を役割別に変える**（コアノードは30秒、待機ノードは5分）
- **WebSocketによるリアルタイム更新**は後回し（過剰設計になりがち）

---

## 「今あるUI」を急いで捨てない

ノードが増えるたびにフルリライトしていたら、ダッシュボードは永遠に完成しない。

今回の判断は「現行UIをグループ化対応だけ追加してリリース」だ。コンパクトモードや検索は、実際に22台を運用してみて不便さが明確になってから追加する。

インフラのスケールアップと、UIのスケールアップを、同じスプリントに詰め込まない——これがホームラボで長く運用していくための地味だが大切なルールだと思っている。

