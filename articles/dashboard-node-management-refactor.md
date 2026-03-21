---
title: "22台ノードを1画面に収める：ダッシュボードのノード管理UIを刷新した話"
emoji: "🖥️"
type: "tech"
topics: ["homelab", "dashboard", "JavaScript", "Flask", "UI"]
published: true
---

> 実体験：2026-03-21、ホームラボ管理ダッシュボードのUI改善

## 背景：ノードが倍になった

ホームラボが気づけば22台構成になっていた。Raspberry Pi、GMKミニPC、Mac mini、専用インフラ機……。当初10台前後を想定して作った管理UIは、画面に収まりきらなくなってスクロールが必要な状態に。

もともと12台程度のインフラが、先週の拡張で一気に10台追加。新しく加わったのは主にGMKのミニPCで、`agent01`〜`agent10`という命名で連番IPを割り当てている。加えて、Mac miniのIP変更、一部ノードの改名という変更も重なった。

ダッシュボードの「ノード管理」ページは、サーバーサイド（Flask）とフロントエンド（Vanilla JS）の両方にノードのマスタデータが書いてある構造だった。両方が古いまま。今回まとめて更新した。

## 技術的にやったこと

### 1. マスタデータの同期

サーバーサイド（`server.py`）の `NODES_REGISTRY` と、フロントエンドの `getLocalNodeData()` 関数の2箇所にノード定義が散在していた。どちらも手動で管理するのは辛いが、今回はシンプルに両方更新する方針を取った。

更新内容：

- ノード改名の反映
- ゲートウェイ専用ノードを追加
- `agent01`〜`agent10` を全追加
- `NODE_ORDER` 配列を最新の22ノード順に更新

```python
# server.py の一部（IP等は省略）
NODES_REGISTRY = {
    "node-a":   {"gatewayPort": 18789, "agents": ["main-agent"]},
    "node-b":   {"gatewayPort": 18789, "agents": ["sub-agent"]},
    "infra":    {"gatewayPort": 18788, "agents": ["shadow"]},
    "agent01":  {"gatewayPort": 18789,
                 "agents": ["aws-expert", "gcp-expert", "snowflake-expert"]},
    # ... 全22ノード
}
```

### 2. 操作ボタンの折りたたみ

各ノードカードに9つの操作ボタン（再起動、状態確認、Agent管理など）が常時表示されていた。22枚並ぶとボタンだけで画面が埋まる。

解決策は「アコーディオン式折りたたみ」。ワンクリックで操作エリアを展開する仕組みを追加した。

```javascript
function nmToggleActions(nodeId) {
    const body = document.querySelector(`#node-${nodeId} .nm-actions-body`);
    const toggle = document.querySelector(`#node-${nodeId} .nm-actions-toggle`);
    const isOpen = body.classList.contains('open');
    body.classList.toggle('open', !isOpen);
    toggle.textContent = isOpen ? '⚙️ 操作 ▼' : '⚙️ 操作 ▲';
}
```

CSS側はシンプルに：

```css
.nm-actions-body {
    display: none;
}
.nm-actions-body.open {
    display: block;
}
```

普段はすっきりしたカード一覧、必要な時だけ操作UIを展開できる。

## やってみての感想

「データを2箇所に持つ設計はいずれ破綻する」というのは分かっていたが、今回まさにそれが起きた。22台分を2ファイルに手動で同期するのは普通に辛い。

本来はサーバーから `/api/nodes` 的なエンドポイントで返して、フロントはそれを受け取るだけにするのがベスト。今回はスコープを絞って「まず動くものに直す」優先にしたが、次の機会にリファクタリングしたい。

折りたたみUIはライブラリ不使用で純粋なHTMLのclass切り替えだけで実装。Vanilla JSでも十分これくらいはできる。外部依存を増やさないほうが保守しやすい。

## まとめ

- ノード22台になっても1画面に収まるようになった ✅
- 操作ボタンは折りたたみで整理 ✅
- マスタデータの2重管理は将来の負債として認識済み

ホームラボのダッシュボードは地味だが、毎日見るもの。細かいUXが積み重なってくる。「22台をスクロールせず見たい」は小さなニーズだけど、直すと体感が変わる。
