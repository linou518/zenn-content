---
title: "Mattermostプラグインを自作して、AIエージェントのモデルをUIからワンクリック切替にした話"
emoji: "🔌"
type: "tech"
topics: ["mattermost", "plugin", "ai", "multiagent", "openclaw"]
published: true
---

# Mattermostプラグインを自作して、AIエージェントのモデルをUIからワンクリック切替にした話

## TL;DR

20以上のAIエージェントを運用していると、「このagentだけGPT-4oに切り替えたい」「全員Claudeに戻して」みたいな操作が日常的に発生する。毎回設定ファイルを書き換えてgatewayを再起動するのは面倒すぎる。そこでMattermostのカスタムプラグインを作り、チャンネルヘッダーからワンクリックでモデルを切り替えられるようにした。

## 背景：なぜMattermostプラグインなのか

うちのmulti-agentシステムでは、全agentの通信基盤としてMattermostを使っている。ユーザーが各agentとDMでやりとりし、agent同士もDMで連携する。つまりMattermostは「agentの管理画面」でもある。

モデル切り替えの要件はこうだった：

1. MattermostのUIから、操作対象のagentを選んでモデルを変更できる
2. 変更は即座にagentに反映される（再起動不要）
3. 現在どのモデルを使っているか一目でわかる

「じゃあMattermostプラグインとして作ろう」は自然な発想だった。

## アーキテクチャ

```
[Mattermost UI]
    ↓ チャンネルヘッダーボタン押下
[Plugin Frontend (React)]
    ↓ API呼び出し
[Plugin Backend (Go)]
    ↓ KVStore に保存
    ↓ agentが毎session開始時にKVを能動的に取得
[Agent (OpenClaw)]
    ↓ session_status(model=xxx) で切替
```

### KVStore：Mattermost内蔵のKey-Value

Mattermostプラグインには `KVStore` という永続化ストレージが付いている。SQLiteやPostgreSQLに別途テーブルを切る必要がない。`plugin.API.KVSet(key, value)` / `KVGet(key)` だけで読み書きできる。

モデル設定はここに保存する。キーは `model_preference`、値はJSON：

```json
{
  "model": "anthropic/claude-opus-4-6",
  "updated_at": "2026-03-27T13:22:00+09:00",
  "updated_by": "admin"
}
```

### フロントエンド：チャンネルヘッダーに切替ボタン

Mattermostプラグインの `channel_header_button` 拡張ポイントを使って、チャンネルヘッダーにアイコンボタンを配置。押すとモーダルが開き、利用可能なモデル一覧から選択できる。

```javascript
registry.registerChannelHeaderButtonAction(
    <ModelSwitcherIcon />,
    (channel) => openModelSwitcherModal(channel),
    'Switch AI Model'
);
```

### バックエンド：GoでREST API

```go
func (p *Plugin) handleModelGet(w http.ResponseWriter, r *http.Request) {
    data, _ := p.API.KVGet("model_preference")
    w.Header().Set("Content-Type", "application/json")
    w.Write(data)
}

func (p *Plugin) handleModelSet(w http.ResponseWriter, r *http.Request) {
    var req ModelRequest
    json.NewDecoder(r.Body).Decode(&req)
    data, _ := json.Marshal(req)
    p.API.KVSet("model_preference", data)
    w.WriteHeader(http.StatusOK)
}
```

## 一番ハマったところ：通知の壁

最初の設計では、モデル変更時にagentへリアルタイム通知する予定だった。KVに書き込んだ後、HTTPで内部サービス経由でagentに「モデル変わったよ」と伝える。

しかしこれが動かない。原因はDockerネットワーク。Mattermostはコンテナ内で動いており、通知先のサービスは別のホストで動いている。コンテナ内からホストネットワークへのHTTP通信がファイアウォールに阻まれた。

### 解決策：プル型に切り替え

push型（プラグイン → agent）を諦めて、pull型（agent → プラグイン）に切り替えた。

各agentが毎回session開始時に、プラグインAPIから現在のモデル設定を取得する：

```bash
curl -s "http://mattermost-server:8065/plugins/<plugin-id>/api/v1/model" \
  -H "Authorization: Bearer <token>"
```

返ってきたモデルが現在のsessionと異なれば、`session_status(model=xxx)` で切り替え。これを各agentの設定に「session開頭の必須手順」として書いておく。

結果として：
- 通知基盤への依存がゼロになった
- モデル切替は「次のsession開始時」に反映される（即時ではないが、十分実用的）
- 実装がシンプルになった

## もう一つの機能：RHS File Browser

同じプラグインに「ファイルブラウザ」機能も入れた。Mattermost右サイドパネル（RHS）に共有ディレクトリの中身を表示し、ファイルの閲覧や管理ができる。

ハマりポイントは `toggleRHSPlugin` の呼び出し方。これはRedux actionなので、直接関数呼び出しではなく `store.dispatch()` 経由で呼ぶ必要がある：

```javascript
// ❌ 動かない
toggleRHSPlugin();

// ✅ 正しい
store.dispatch(toggleRHSPlugin());
```

Mattermostプラグインのフロントエンドドキュメントにはこの辺りの記載が薄く、ソースコードを読んで解決した。

## 運用の感想

3日ほど使ってみて、「UIからモデルを切り替えられる」のは思った以上に便利だった。特にClaude OpusとSonnetを使い分けたいとき——重い推論タスクにはOpus、日常会話にはSonnet——ワンクリックで済む。

KVStore + pull型アーキテクチャは、リアルタイム性では劣るが、安定性と単純さでは圧勝。multi-agentシステムでは「壊れにくい」が最優先だと再認識した。

## まとめ

- Mattermostプラグインは「agentの管理UI」として十分に使える
- KVStoreは設定値の永続化に最適（DB不要）
- push型通知が難しいなら、pull型に割り切る方が運用が楽
- `toggleRHSPlugin` は Redux action、`store.dispatch()` 必須

Mattermostをagent基盤に使っているなら、プラグイン自作は検討の価値あり。公式ドキュメントは最低限だが、GitHubの `mattermost-plugin-starter-template` から始めれば1日で動くものが作れる。
