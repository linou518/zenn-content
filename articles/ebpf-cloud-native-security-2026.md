---
title: "eBPF：Linuxカーネルで静かに革命を起こすセキュリティ黒魔術"
emoji: "⚡"
type: "tech"
topics: ["eBPF", "Kubernetes", "\u30bb\u30ad\u30e5\u30ea\u30c6\u30a3", "\u53ef\u89b3\u6e2c\u6027"]
published: true
---

## はじめに：静かなカーネル革命

コンテナが侵害され、攻撃者がカーネル内を横断移動し始めたとき、あなたのセキュリティシステムはそれをどうやって検知するのだろうか？

従来の答えは「監査ログが書き出されてから、SIEMがアラートを上げて……」というものだった。すでに手遅れである。

2026年の答えは、**eBPFが攻撃行動の瞬間に、カーネル層で直接ブロックする**だ。

これは未来の話ではない。2025年、AWSはEKS（マネージドKubernetesサービス）のデフォルトCNIをCiliumに変更すると発表した。CiliumはeBPF上に構築されている。この決断は一つの事実を宣言している：**eBPFは実験室を出て、本番インフラの核心に入った**。

---

## eBPFとは何か？一言で説明する

eBPF（拡張バークレーパケットフィルター）は、Linuxカーネル内のサンドボックス機構だ。カーネルのソースコードを変更せず、システムを再起動せずに、**カーネル空間でカスタムプログラムを安全に実行**できる。

政府機関に合法的に送り込まれたエージェントが、すべての往来文書をリアルタイムで監視しながら、誰にも気づかれず、どのプログラムの動作も妨げないイメージだ。eBPFはそういうカーネルの中の「合法スパイ」である。

その魔法は、システムの最底層であらゆる挙動を傍受・観測・修正できながら、カーネルをクラッシュさせないことにある。

---

## 2025年：eBPFの転換点

| 出来事 | 意義 |
|--------|------|
| AWS EKS がデフォルト Cilium CNI に | 最大のクラウドベンダーが後押し、eBPF ネットワークがデフォルトに |
| Cilium が CNCF を正式卒業 | 企業導入の信頼性が確立 |
| Linux kernel 6.x で eBPF 機能が安定 | 本番利用に十分な成熟度 |
| Falco / Tetragon が大規模商用化 | セキュリティ用途が加速 |

象徴的な数字：Alibaba Cloud は eBPF を活用したアダプティブ L7 ロードバランシングで、**インフラコストを 19% 削減**した。

---

## クラウドネイティブにおける eBPF の4大活用シーン

### ①ネットワーク高速化（Cilium）

Kubernetes の従来ネットワークは iptables に依存している。Pod が 500 個を超えると、iptables のルールチェーンは爆発的に増加し、パフォーマンスが急落する。

Cilium は eBPF でカーネル内部のパケット処理を行い、iptables を完全に迂回する：

- **スループット 30〜40% 向上**
- ネットワークポリシーの実行レイテンシ低減
- L7 対応（HTTP メソッドレベルのアクセス制御が可能）

### ②ゼロ計装の可観測性（Pixie / Hubble）

従来の可観測性はコード埋め込みや sidecar 注入が必要だった。eBPF はこれを根本から変える。

**コード変更なし・sidecar なしで監視できる対象：**
- すべての HTTP リクエスト・レスポンス
- DB クエリ（MySQL / PostgreSQL / Redis）
- DNS 解決
- ファイルアクセスパターン

Pixie はデプロイから 30 秒以内にサービス全体のトポロジーを可視化できる。

### ③カーネルレベルのセキュリティランタイム（Tetragon / Falco）

これが最大の質的変化だ。

**従来の問題**：ユーザー空間の監査ログを読む従来の HIDS は、攻撃者がログ書き込み前に操作を完了できてしまう。まるで防犯カメラが「録画」しかしていないようなものだ。

**Tetragon のアプローチ**：すべてのシステムコールをカーネル層でフック。Nginx ワーカーが `/etc/passwd` にアクセスしようとした瞬間、Tetragon は**カーネル層でそのプロセスを即座に終了**する。ユーザー空間のどんなセキュリティツールより速い。

しかも Kubernetes 対応：どの Namespace のどの Pod のどの ServiceAccount が操作しているかを把握し、TracingPolicy CRD で GitOps フレンドリーに管理できる。

### ④サイドカーなし Service Mesh（Istio Ambient Mesh）

従来の Istio は Pod ごとに Envoy sidecar を注入し、Pod あたり〜100MB の余分なメモリを消費していた。Ambient Mesh は eBPF でノード層のトラフィックを一元管理し、**sidecar なしで mTLS・ポリシー・可観測性を維持**する。

---

## 実際の導入データ

- **Canopus**：eBPF でネットワーク可観測性を再構築、サーバー占有率 **3分の1** に
- **Rakuten Mobile**：クラウドネイティブ通信網の異常検知に eBPF を採用
- **蚂蚁集団（Ant Group）**：Kata Containers + eBPF で細粒度プラットフォームセキュリティを実現
- **Alibaba Cloud**：eBPF アダプティブ L7 LB でインフラコスト **19% 削減**

---

## 冷静な評価：eBPF の実際の課題

**カーネルバージョン依存**：多くの高度な eBPF 機能には Linux kernel 5.8+ が必要。CentOS 7 / RHEL 7 系の古いシステムは非対応。Windows サポートはまだ初期段階。

**デバッグの難しさ**：eBPF プログラムのトラブルシューティングは通常のアプリより難易度が高く、専門知識（bpftool、bpftrace 等）が必要。

**エコシステムの断片化**：Cilium、Calico、Falco、Tetragon それぞれが独自実装を持ち、機能が重複・非互換な部分も。

**権限要件**：`CAP_BPF` または `CAP_SYS_ADMIN` が必要で、厳格なセキュリティ環境では追加承認が必要。

---

## 今すぐできること

**ステップ1**：カーネルバージョン確認。

```bash
uname -r
# 5.8 以上であることを確認
```

**ステップ2**：Kubernetes を使っているなら現在の CNI を評価。新規クラスターは直接 Cilium を選択、既存クラスターは移行パスを計画する。

**ステップ3**：テスト環境に Tetragon をデプロイ。

```bash
helm repo add cilium https://helm.cilium.io
helm install tetragon cilium/tetragon -n kube-system
```

最初の TracingPolicy として「どのプロセスが `/etc/passwd` にアクセスしているか」を監視してみよう。eBPF の能力を直感的に理解できる。

---

## 結論：もう「様子見」は終わり

eBPF はあなたが直接書く技術ではない——Cilium、Pixie、Tetragon を使えば、eBPF を活用していることになる。

それが意味するのは思考の転換だ：**セキュリティと可観測性はアドオンではなく、インフラそのものの能力になった**。

AWS がデフォルトにした以上、「まだ様子を見る」という選択肢はもう存在しない。

> **関連トピック：** Cilium Network Policy 詳解 / Tetragon TracingPolicy 実践 / Platform Engineering 2026

---

*参考：Cloud Native Now、DevOps Year in Review、Tetragon 公式ドキュメント*