---
title: "高可用フェイルオーバーの構築：90秒で自動接管"
emoji: "📝"
type: "tech"
topics: ["OpenClaw", "AI"]
published: true
---


OpenClawを独立PCに移した後、すぐに一つの問題が浮上しました：PC-Aがダウンしたらどうなる？すべてのbotが接続不能になり、すべてのagentが停止します。この記事では、PC-Bを使ってシンプルだが効果的なフェイルオーバーメカニズムを構築した過程を記録します。

## 設計思想

高可用の核心はシンプルです：バックアップノードが、プライマリノード障害時に自動的に引き継ぐ。

- **PC-A（192.168.x.x）**：プライマリノード、OpenClaw gatewayを通常運用
- **PC-B（192.168.x.x）**：バックアップノード、監視スクリプトを実行、通常は待機、障害時に接管

目標：プライマリ障害後90秒以内に自動接管、プライマリ復旧後に自動解放。

## failover-monitor.sh

PC-B上で実行するbashスクリプトが核心です：

```bash
# 擬似コード
while true; do
    if PC-Aのポート18789に到達可能; then
        fail_count=0
        if ローカルでgatewayが実行中; then
            ローカルgatewayを停止  # プライマリ復旧、解放
        fi
    else
        fail_count++
        if fail_count >= 3; then
            ローカルgatewayを起動  # 連続3回失敗、接管
        fi
    fi
    sleep 30
done
```

30秒ごとにチェックし、連続3回失敗で接管をトリガー——最低90秒の確認ウィンドウ。ネットワークの揺らぎによる誤切替を防ぐための設計です。

## 落とし穴：pgrep -f のマッチング罠

当初`pgrep -f "openclaw gateway"`でローカルgatewayの実行確認をしていました。

大間違いでした。OpenClawのagentはshellコマンドを実行できます。agentが"openclaw"や"gateway"を含むコマンドを実行すると、`pgrep -f`がその一時的なプロセスにマッチし、gatewayが実行中だと誤認します。

**解決策**：`ss -tlnp | grep :18789`でポートリッスンを確認する方式に変更。ポートチェックはプロセス名マッチよりはるかに信頼性が高い。

```bash
check_local_running() {
    ss -tlnp | grep -q ":18789 " && return 0 || return 1
}

check_primary() {
    timeout 5 bash -c "echo > /dev/tcp/192.168.x.x/18789" 2>/dev/null
}
```

## 落とし穴：自分自身を停止できない

スクリプトはsystemdサービスとして`Restart=always`で実行。プライマリ復旧時にgatewayを停止する必要がありますが、gatewayとmonitorの依存関係を整理しないと、gateway停止がmonitorに連鎖影響する可能性があります。

**解決策**：monitorスクリプトとgatewayサービスを完全に独立させる。systemdではなく、直接`openclaw gateway start/stop`で管理。

## テスト結果

| シナリオ | 所要時間 |
|----------|----------|
| プライマリ障害 → バックアップ接管 | ～65秒 |
| プライマリ復旧 → バックアップ解放 | ～30秒 |

## 不完全だが十分

制限事項はあります：切替中のセッションコンテキストは失われ、設定同期は手動、ポートのみのチェック。

しかし個人プロジェクトには十分です。エンジニアリングの技術は、完璧と実用の間のバランスポイントを見つけることにあります。

この小さなmonitorスクリプトで初めて実感しました：**信頼性は単一点の完璧さではなく、システムレベルの冗長性によって実現される**。
