---
title: "Dashboardノード構成刷新：4台から7台へ、SSHキー配布とステータスバッジ実装記"
emoji: "🖥️"
type: "tech"
topics: ["dashboard", "ssh", "python", "flask", "css"]
published: true
---

自宅ラボのDashboardを大幅にリファクタリングした。きっかけは単純で、管理するノードが増えたのに、古い4台構成のまま放置していたことへの罪悪感。

---

## 旧構成の問題点

旧Dashboard（v1）は以下の4ノードを管理していた：

- `T440`（joe）
- `GMK1`〜`GMK3`

実際には7台のマシンが動いており、infra（192.168.x.x）やweb（192.168.x.x）を除いた6台＋joe（.x）が運用中。Dashboardの情報が現実と乖離していた。

---

## 新構成への移行

### ノード定義の更新

```python
NODES = [
    {"id": "joe",      "ip": "192.168.x.x", "hostname": "joe",      "user": "openclaw"},
    {"id": "jack",     "ip": "192.168.x.x", "hostname": "jack",     "user": "openclaw"},
    {"id": "work-a",   "ip": "192.168.x.x", "hostname": "work-a",   "user": "openclaw"},
    {"id": "work-b",   "ip": "192.168.x.x", "hostname": "work-b",   "user": "openclaw"},
    {"id": "hobby",    "ip": "192.168.x.x", "hostname": "hobby",    "user": "openclaw"},
    {"id": "family",   "ip": "192.168.x.x", "hostname": "family",   "user": "openclaw"},
    {"id": "personal", "ip": "192.168.x.x", "hostname": "personal", "user": "openclaw"},
]
```

インフラ担当の infra（.x）と web（.x）はDashboardの管理対象外とした。運用上、これらはサービス基盤として別管理するほうが責任分離として正しい。

### SSHキー配布問題

移行で一番詰まったのはここ。`T440`（joe）から各ノードへSSHが通らず、サブスクリプション情報が取得できなかった。

```bash
# sshpassで一括配布
for host in jack work-a work-b hobby family personal; do
    sshpass -p "$PASS" ssh-copy-id -o StrictHostKeyChecking=no openclaw@$host
done
```

`ssh-copy-id`を`sshpass`経由で流すのは若干お行儀が悪いが、ラボ環境なので割り切り。本番ではProvisioning toolを使う。

---

## フロントエンド改善

### 丸ドット → ステータスバッジ

以前は小さな丸（●）でオンライン/オフラインを示していたが、視認性が悪かった。バッジ形式に変更した：

```css
.status-badge {
    display: inline-flex;
    align-items: center;
    gap: 4px;
    padding: 2px 8px;
    border-radius: 12px;
    font-size: 11px;
    font-weight: 600;
}

.status-badge.online  { background: rgba(74,222,128,.15); color: #4ade80; }
.status-badge.offline { background: rgba(248,113,113,.15); color: #f87171; }
.status-badge.warning { background: rgba(251,191,36,.15);  color: #fbbf24; }
```

「运行中」「不可达」「停止」の3状態に整理。「警告」状態はレスポンスが遅い（ping 500ms超）ケースに使う。

### Agentタグ表示

各ノードカードにそのノードで動いているOpenClaw agentの一覧を表示するようにした。

```json
// node_subscriptions.json（一部）
{
    "work-b": {
        "agents": ["techsfree-web", "techsfree-hr", "techsfree-fr"]
    }
}
```

バックエンドで`get_node_subscription()`関数を追加し、ノードAPIレスポンスにマージする形で対応。

```python
def get_node_subscription(node_id: str) -> dict:
    try:
        with open(SUBSCRIPTIONS_FILE) as f:
            data = json.load(f)
        return data.get(node_id, {})
    except Exception:
        return {}
```

---

## 削除したもの

**Bot数のstat**を削除した。「このノードに何個のBotが動いている」という情報を表示していたが、Agentタグと完全に重複しており、数字だけでは意味がなかった。

削除の判断基準：「この情報がなくなったとき、誰かが困るか？」——答えがNoなら消す。

---

## 4列グリッドへの変更

7ノードを表示するため、グリッドを4列構成にした。

```css
.node-grid {
    display: grid;
    grid-template-columns: repeat(4, 1fr) !important;
    gap: 16px;
}

@media (max-width: 1200px) {
    .node-grid { grid-template-columns: repeat(2, 1fr) !important; }
}
@media (max-width: 900px) {
    .node-grid { grid-template-columns: 1fr !important; }
}
```

`!important`を使っているのは、既存のCSS変数との競合を避けるため。本来は設計を整理すべきだが、今日のスコープ外。

---

## 学んだこと

1. **現実とダッシュボードのズレは放置するほど治しにくくなる。** 少し増えたら即更新する習慣が重要。
2. **SSH接続は「通るか確認」してから機能を実装する。** SSH前提の機能を作り込んでからSSHが通らないと気づくのは時間の無駄。
3. **UIの「丸ドット」は視認性が低い。** 色盲の方への配慮も含め、バッジ+テキスト表記のほうが汎用性が高い。

次のステップはノードのアラート機能。応答がなくなったら通知が飛ぶようにしたい。

---

**タグ案:** `Dashboard`, `自宅ラボ`, `フロントエンド`, `SSH`, `Python`, `Flask`, `CSS Grid`, `Node管理`
