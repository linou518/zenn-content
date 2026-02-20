---
title: "Slack連携と災害復旧体制の構築"
emoji: "📝"
type: "tech"
topics: ["OpenClaw", "AI"]
published: true
---


システムが安定稼働し始めた後、次の課題は「もっと機能を増やす」ではなく「すべてが崩壊したらどうする？」でした。一見無関係だが本質的に同じ2つのこと：Slack連携（通信チャネルの拡張）と災害復旧体制の構築（システム復元可能性の確保）を記録します。

## Slack Socket Mode接続

OpenClawはずっとTelegramで通信していましたが、業務シナリオをカバーするためにSlackを追加。多くのチームコラボレーションはSlack上で行われ、agentがTelegramでしか応答できなければ大量の業務コンテキストを見逃します。

Slackの統合方式には従来のWebhookではなく**Socket Mode**を選択：
- **Webhookは公開URLが必要**：内部ネットワークのサーバーでは追加の複雑さとコスト
- **Socket Modeは外向き接続**：内部からSlackサーバーに接続、公開IP不要、ポート開放不要、ファイアウォールフレンドリー

## 災害復旧体制

### 三層防護

**第1層：Healthchecks.io 監視**
重要サービスにチェックポイントを登録。定期的にping、タイムアウトでアラート発報。

```bash
*/5 * * * * curl -fsS --retry 3 https://hc-ping.com/<check-uuid> > /dev/null
```

**第2層：GitHub Private Repo**
すべての設定ファイル、agentのSOUL/MEMORYファイル、重要スクリプトをGitHub private repoにプッシュ。ローカルマシンが全壊しても、GitHubから復元可能。

**第3層：GPG暗号化**
APIトークン、パスワード、SSH鍵などの機密情報はGPGで暗号化してからプッシュ。

```bash
tar czf secrets.tar.gz tokens/ keys/
gpg --symmetric --cipher-algo AES256 -o secrets.tar.gz.gpg secrets.tar.gz
shred -u secrets.tar.gz
```

### DISASTER-RECOVERY.md

復旧フロードキュメントを作成、4段階の復旧プロセスを定義：

| フェーズ | 時間 | 内容 |
|----------|------|------|
| 1. 評価と通信 | 0-30分 | 障害範囲確認、agent状態確認 |
| 2. コア復旧 | 30-60分 | gateway復旧、GitHub から設定取得、secrets復号 |
| 3. サービス復旧 | 60-90分 | 全agent起動、Dashboard復旧、接続性検証 |
| 4. 検証と振り返り | 90-120分 | bot応答テスト、データ整合性確認 |

**総目標：2時間以内に全面復旧。**

## 所感

災害復旧は「やっても感謝されず、やらなければ事故時に責任を取る」仕事です。「うちのシステムは安定している、災害復旧は不要」と考える人は多い——ハードディスクがカチッと鳴るまでは。

**システムの信頼性は最強のコンポーネントではなく、最弱のリンクで決まります。** OpenClawが自動フェイルオーバーでき、agentがインテリジェントに応答できても、設定ファイルのバックアップがなければ、一つの壊れたディスクですべてがゼロに戻ります。

最善の準備をし、最悪の事態に備える。これが運用の第一信条でしょう。
