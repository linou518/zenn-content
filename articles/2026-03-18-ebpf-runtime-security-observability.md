---
title: "eBPF：クラウドネイティブセキュリティと可観測性のカーネル革命"
emoji: "🐝"
type: "tech"
topics: ["eBPF", "Kubernetes", "セキュリティ", "可観測性"]
published: true
category: "infra"
lang: "ja"
---

# eBPF：クラウドネイティブセキュリティと可観測性のカーネル革命

> 参考: [Cloud Native Now](https://cloudnativenow.com/editorial-calendar/best-of-2025/ebpf-the-silent-power-behind-cloud-natives-next-phase-2/), [Medium DevOps Year Review](https://medium.com/@inboryn/2025-devops-year-in-review-the-5-biggest-infrastructure-shifts-and-what-they-mean-for-2026-ffb7a6735139), [Tetragon](https://tetragon.io/)

---

## 核心発見

- **eBPFがメインストリーム化**：2025年、AWS EKSがデフォルトでCilium（eBPFベースCNI）を採用し、eBPFの完全主流化を象徴
- **大幅なパフォーマンス向上**：CiliumのeBPFデータパスは従来のiptablesネットワークより30-40%のスループット向上を実現
- **ゼロインストゥルメンテーション可観測性の実現**：アプリケーションコードの変更やsidecar注入不要で、全てのsyscall、ネットワークパケット、ファイルアクセスを追跡可能
- **カーネルレベルセキュリティの質的変化**：Tetragon/Falcoがカーネル層で脅威を検知し、ユーザー空間のソリューションより高速な応答を実現

---

## 詳細内容

### eBPFとは何か（一言で）

eBPF（拡張版Berkeley Packet Filter）は、Linuxカーネル内のサンドボックス機構で、**カーネル空間で安全にカスタムプログラムを実行**できる技術です。カーネルソースの変更や再起動不要で、システムの最下層でふるまいを傍受、観測、修正することが可能です。

### 2025年：eBPF普及の転換点

2025年はeBPFが「実験的技術」から「業界標準」へ転換した年でした：

| イベント | 意義 |
|------|------|
| AWS EKSでCilium CNIがデフォルト | 最大手クラウドプロバイダーがeBPFネットワークを標準選択肢に |
| CiliumがCNCF正式卒業 | コミュニティ承認、企業の安心採用 |
| Linux kernel 6.x系でeBPF機能安定化 | 技術基盤の成熟 |
| Falco / Tetragon大規模商用実装 | セキュリティ分野での実装加速 |

### クラウドネイティブにおけるeBPFの4大応用分野

#### 1. ネットワーク（Cilium）
従来のKubernetesネットワークはiptablesに依存し、Pod数の爆発的増加に伴ってルールチェーンが増大、パフォーマンスが深刻に低下していました。Ciliumはカーネル内でeBPFを使用してネットワークパケットを直接処理し、iptablesを迂回：
- スループット30-40%向上
- ネットワークポリシー実行遅延削減
- L7レベル（HTTP/gRPC）のポリシー対応

#### 2. 可観測性（Pixie / Hubble）
従来の可観測性では、コードに計測点を追加したり、Jaeger/OpenTelemetryエージェントを注入する必要がありました。eBPFは「ゼロインストゥルメンテーション」を実現：
- 全てのHTTPリクエスト、データベースクエリ、DNS解決を自動収集
- 開発者はコードを一行も変更する必要なし
- Pixieは30秒で展開し、全サービストポロジーを可視化

#### 3. セキュリティランタイム（Tetragon / Falco）
これは2026年最も注目すべき方向性です。eBPFセキュリティツールは**カーネル層**でポリシーを実施：

**Tetragonの機能：**
- 全プロセスのファイルアクセス、ネットワーク接続、権限昇格を監視
- 異常検知時にカーネル層でプロセスを即座に停止（ユーザー空間での対応を待たない）
- Kubernetes対応：どのPod、Namespace、ServiceAccountが何を行っているか把握
- GitOps対応のTracingPolicy CRDでポリシー定義

**従来のHIDSとの違い：**
従来のホスト侵入検知システム（Falco旧版など）はユーザー空間でaudit logを読み取るため、攻撃者は検知前に操作を完了できました。eBPFはカーネル層で直接フックし、**攻撃行為の発生時にリアルタイムで阻止**します。

#### 4. Service Mesh（Istio Ambient Mesh）
従来のIstioは各PodにEnvoy sidecarを注入し、リソースの無駄が深刻（Pod1つあたり約100MB追加消費）でした。Ambient MeshはeBPFでノード層のトラフィックを接管：
- sidecar注入不要
- mTLS、トラフィックポリシーが引き続き有効
- リソース消費大幅削減

### 実用事例データ

- **主要クラウドプロバイダー**：eBPFによるネットワーク可観測性再構築でサーバー負荷を3分の1に削減
- **大手Eコマース**：eBPFを使用したクラウドネイティブ通信網異常検知システム
- **大手金融機関**：Kata Containers + eBPFによる細粒度プラットフォームセキュリティ実装
- **大手クラウド事業者**：eBPF適応型L7負載分散によるインフラコスト19%削減

### リスクと課題（冷静な評価）

技術トレンドに惑わされず、eBPFの実際の課題も認識が必要：

1. **カーネルバージョン依存**：多くの高度なeBPF機能にはLinux kernel 5.8+が必要。古いシステム（CentOS 7/RHEL 7）は完全に非対応。Windowsサポートは初期段階
2. **デバッグ難易度**：eBPFプログラムの問題発生時、通常アプリケーションよりデバッグが困難で、専門知識が必要
3. **エコシステムの断片化**：Cilium、Calico、Falco、Tetragonがそれぞれ独自のeBPF実装を持ち、標準化推進中だが機能重複や非互換性が存在
4. **高権限要求**：eBPFプログラム実行には`CAP_BPF`または`CAP_SYS_ADMIN`が必要で、厳格なセキュリティ環境では追加承認が必要

---

## まとめ

### 即座に実施可能（低コスト）

1. **既存KubernetesクラスターのCNI評価**：まだflannel/calico+iptablesを使用している場合、Ciliumへの移行可能性を評価。新規クラスターは直接Cilium採用、既存クラスターは移行計画を策定
2. **Tetragonセキュリティ監視導入**：本番サーバーでTetragonを展開し、どのプロセスが機密ファイルにアクセスしているか、異常なネットワーク接続を確立しているかをリアルタイム把握
3. **Linuxカーネルバージョン確認**：`uname -r`で全ノードが5.8以上であることを確認し、完全なeBPF機能利用を保証

### 中期計画

4. **可観測性アップグレード**：現在の監視システムが手動計測に依存している場合、Pixie（ゼロインストゥルメンテーションAPM）導入を検討し、観測コストを大幅削減
5. **Tetragon TracingPolicy学習**：CRDを使用したセキュリティポリシー定義の習得は、2026年のSREコアスキルの一つ

### 認識アップデート

- **eBPFは直接書くコードではなく、使用するツールの基盤技術**：技術用語に惑わされず、Cilium使用はeBPF活用を意味
- **「カーネル級セキュリティ」の実用価値**：異常動作するプロセスがあった場合、Tetragonはカーネル層で発見・阻止可能
- **プラットフォーム工学のトレンド**：DORA 2025データによると、成熟したプラットフォーム工学チームを持つ組織は、そうでない組織の3.5倍のデプロイ頻度を実現

---

## 関連トピック

- **Cilium Network Policy詳細**：L7対応ネットワークポリシー、HTTPメソッドレベルのアクセス制御
- **Tetragon TracingPolicy実践**：CRDでセキュリティポリシー作成、コンテナエスケープ・権限昇格検知
- **Platform Engineering & Backstage**：内部開発者プラットフォーム、インフラ複雑性削減
- **LLMインフラ経済学**：AIワークロードのコスト最適化、2026年の新たな挑戦
- **OpenTofu vs Terraform**：HashiCorpライセンス変更後のIaCツール選択

<!-- 本文已脱敏処理 -->