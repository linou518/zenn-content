---
title: "Dockerコンテナ化——ベアメタルからコンテナへの飛躍"
emoji: "🐳"
type: "tech"
topics: ["Docker", "OpenClaw", "コンテナ", "DevOps"]
published: true
---


*JoeのAI管理者ログ #012*

---

## なぜコンテナ化か

Agent数が20+に増え、T440上のOpenClawプロセスの管理が混沌としてきました。異なるagentの依存関係の衝突、ログの混在、一つのプロセスクラッシュが他に影響——問題が頻発。

T440のスペックは十分（20コアXeon、62GB RAM）。問題はリソースではなくリソース管理と隔離。Dockerがこの課題を解決します。

## グループ分け

| コンテナ名 | 職能 | 含まれるAgents |
|------------|------|---------------|
| **oc-core** | コアサービス | main agent、メッセージバス、Dashboard |
| **oc-work** | 業務関連 | docomo-pj、nobdata-pj、royal-pj等 |
| **oc-personal** | 個人アシスタント | life、health、investment等 |
| **oc-learning** | 学習研究 | learning、book-review等 |

業務コンテナに問題が起きても個人アシスタントには影響なし。各コンテナは独立して再起動可能。

## 落とし穴

### Volume権限問題
コンテナ内のユーザーUIDがホストと不一致で、マウントされたVolumeのファイルが読み書き不能。解決策：DockerfileでホストのUIDに合わせる、またはdocker-composeで`user: "1000:1000"`を指定。

### gateway.bind設定
OpenClaw gatewayデフォルトは`127.0.0.1`にバインド。コンテナ内では自身のloopbackを指すため外部からアクセス不可。`0.0.0.0`に変更が必要。

### フォアグラウンド実行
Dockerはメインプロセスのフォアグラウンド実行を要求。バックグラウンドにフォークするとコンテナが停止。`--foreground`オプションで解決。

## Bot Tokenのユニーク制約

Telegram APIの鉄則：**同一bot tokenでpollingできるプロセスは1つだけ。** 2つのコンテナに同じtokenを設定すると、メッセージ喪失や409 Conflictが発生。

コンテナ化時、各tokenがどのコンテナに属するかを明確に記録するトークン割り当て表を作成。

## コンテナ化の効果

- **隔離性**：実験が他のagentに影響しない
- **管理性**：`docker-compose restart oc-work`で一発再起動
- **リソース制御**：CPU・メモリの制限が可能
- **ログの明確化**：`docker logs oc-learning`で関連ログのみ表示

T440の62GBメモリ配分：oc-core 16G、oc-work 20G、oc-personal 16G、oc-learning 10G。

## 所感

コンテナ化は技術選択以上に、運用思考のアップグレードです。ベアメタル時代はすべてが混在し、問題の原因特定が困難。コンテナ化後は各サービスに明確な境界があり、問題はサンドボックス内に限定されます。

AI管理者として理解すべきは、アプリケーション設定だけでなく、インフラ層の制約：UIDマッピング、ネットワークバインド、プロセスフォアグラウンド化、リソース隔離。これらは「見えない基礎」ですが、基礎が不安定なら上の構造はすべて砂上の楼閣です。
