---
title: "クラウド最後の防衛線：Confidential Containersがメモリ内データを守る仕組み"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "自動化"]
published: true
---

---
title: "クラウド最後の防衛線：Confidential Containersがメモリ内データを守る仕組み"
date: "2026-03-24"
category: "infra"
tags: ["Kubernetes", "セキュリティ", "TEE", "Confidential Computing", "クラウドネイティブ", "Intel TDX", "AMD SEV"]
lang: "ja"
author: "TechsFree"
draft: false
summary: "静的データの暗号化、転送中データの暗号化——では実行中のメモリデータは？Confidential Containers（CoCo）はハードウェアTEE技術を用いて、クラウドプロバイダの管理者でさえ覗けない信頼実行環境を構築します。アーキテクチャ、リモートアテステーション、Trustee鍵管理、そしてKubernetesへの実装まで徹底解説。"
readingTime: 14
---

ディスクを暗号化し、TLSを有効にし、SecretsをKMSに保存した。それでも、クラウドプロバイダの管理者や侵害されたHypervisor、あるいは悪意のある内部者がVMのメモリを直接読めたとしたら？

**これは仮想の脅威ではない。** パブリッククラウドでは、十分な権限を持つ人物にとって、あなたのワークロードのメモリは「真に非公開」ではなかった。従来のセキュリティモデルは基盤インフラを信頼するという前提に立っていたが、AIの時代における機密データ処理において、その前提はますます危険になっている。

Confidential Containers（CoCo）プロジェクトは、その前提を覆す。

---

## はじめに：データ保護の3形態と最後のピース

セキュリティ業界には「データの3形態」という古典的なフレームワークがある：

| 状態 | 保護手段 | 成熟度 |
|------|---------|--------|
| Data at Rest（静的データ） | ディスク暗号化（LUKS/KMS） | ✅ 成熟 |
| Data in Transit（転送中データ） | TLS/mTLS | ✅ 成熟 |
| **Data in Use（実行中データ）** | **ハードウェアTEE** | 🔧 2026年、実用化へ |

最初の2ピースは解決済みだ。しかし、メモリ内で処理中のデータ——実行時データ——は長らく空白地帯だった。CoCoはそのピースを埋める。

---

## 一、ハードウェアTEE：Hypervisorも信頼しない

Confidential Computingのコアは**TEE（Trusted Execution Environment：信頼実行環境）**。これはソフトウェアレベルの暗号化ではなく、**CPUハードウェアレベル**のメモリ保護だ。

### 3つの主要TEE

**Intel TDX（Trust Domain Extensions）**：第4世代Xeon以降でサポート。VMをTrust Domainに分割し、MKTME（マルチキー全メモリ暗号化）でTDのメモリを暗号化。Hypervisorが完全に侵害されても、TD内のデータは読み取れない。

**AMD SEV-SNP（Secure Encrypted Virtualization - Secure Nested Paging）**：AMD EPYC Milan以降でサポート。各VMが独立したメモリ暗号化キーを使用し、SPT（安全ページテーブル）がメモリ再マッピング攻撃を防ぐ。

**IBM Secure Execution（s390x）**：VM全体が暗号化イメージに含まれ、Hypervisorの管理者でもアクセス不可。

> 本質的な理解：これは「より優れたソフトウェア暗号化」ではなく、**CPU設計レベルの信頼の再構築**だ。

---

## 二、CoCo アーキテクチャ：信頼境界はどこに引くか

CNCFのConfidential Containersプロジェクトは、TEE能力とKubernetesをシームレスに統合する。アーキテクチャの核心は**Podを単位としてTEE保護境界を構築**すること（Pod-Centric）。

```
┌─────────────────────────────────────────────────────┐
│           Kubernetesノード（不信頼ホスト）              │
│                                                     │
│  kubelet → containerd → kata-shim → hypervisor      │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │        PodVM / TEE Enclave（信頼済み）         │   │
│  │                                             │   │
│  │  kata-agent → Pod                           │   │
│  │  image-rs   → イメージ取得/復号               │   │
│  │  CDH        → Secret取得                    │   │
│  │  AA         → TEEアテステーション生成         │   │
│  └─────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
                        │ KBS Protocol（TLS + アテステーション）
                        ▼
┌─────────────────────────────────────────────────────┐
│             Trustee（独立した信頼環境）                  │
│  KBS（鍵ブローカー）→ AS（アテステーション検証）→ RVPS    │
└─────────────────────────────────────────────────────┘
```

この設計の革命的な前提：**Kubernetesコントロールプレーン（Kubelet/containerd/APIサーバー）は信頼しない**。

自分でデプロイしたKubeletでさえ、CoCoの脅威モデルでは信頼されない。TEEハードウェア内部のみが「信頼済み」だ。これはより急進的なゼロトラストモデルだ。

---

## 三、リモートアテステーション：「私は誰か」を証明する方法

これがCoCoの最も精巧な部分——**リモートアテステーション（Remote Attestation）**。

暗号化されたSecret（DBパスワード、APIキー）を「真にTDX TEE内で動作している、期待されたコードを使用したワークロード」にのみ渡したい。どう検証するか？TEEハードウェア自身に証明させる。

### KBS Protocolのアテステーションフロー

```
1. ワークロード → CDH：Secret要求
2. CDH → AA：アテステーション開始
3. AA → TEEハードウェア：Evidence取得（TDX Quote / SNP Report）
4. AA → KBS：/kbs/v0/auth（チャレンジnonce取得）
5. AA → KBS：/kbs/v0/attest（Evidence + EK公開鍵 + nonce署名）
6. KBS → AS：Evidence検証
7. AS → RVPS：参照値との比較
8. AS → KBS：EAR証明トークン出力
9. KBS：OPAポリシー評価（「証明OK、だがSecretへのアクセス権は？」）
10. KBS → CDH：EK公開鍵で暗号化されたSecret返却
11. CDH復号 → ワークロード
```

---

## 四、Kubernetesでの実践：最初のConfidential Podを動かす

### CoCo実行環境インストール

```bash
helm install coco oci://ghcr.io/confidential-containers/charts/confidential-containers \
  --namespace coco-system \
  --create-namespace \
  --version 0.18.0

kubectl get runtimeclass
```

主要なRuntimeClass：

| RuntimeClass | 用途 |
|------|------|
| `kata-qemu-tdx` | Intel TDX（ベアメタル） |
| `kata-qemu-snp` | AMD SEV-SNP（ベアメタル） |
| `kata-qemu-nvidia-gpu-tdx` | TDX + GPU（AI学習） |
| `kata-remote` | Peer Pods（クラウドCVM） |

### 最初のConfidential Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: confidential-app
spec:
  runtimeClassName: kata-qemu-tdx
  containers:
  - name: app
    image: myregistry.io/myapp:v1.0@sha256:<digest>  # digestで固定推奨
    resources:
      requests:
        memory: "512Mi"
        cpu: "500m"
```

---

## 五、ユースケースと成熟度評価

**CoCo最適シナリオ：**
- 医療データ処理（HIPAA/GDPR準拠、マルチテナント）
- 金融リスクモデル（知財保護＋データプライバシー）
- AI連合学習ノード（各社データを協調者に漏らさない）
- 暗号資産/鍵管理（クラウドプロバイダ内部脅威対策）

**現在の制限（2026年）：**
- ハードウェア要件：第4世代Intel XeonまたはAMD EPYC Milan以上
- 性能オーバーヘッド：起動に30〜60秒追加、CPU5〜15%増
- デバッグ困難：`kubectl exec`禁止はセキュリティ特性であると同時に運用課題

---

## まとめ：信頼を再設計する時代が来た

CoCoは「既存システムに保護レイヤーを追加する」ものではなく、**脅威モデルレベルから「信頼できるコンピューティング」を再定義した**設計だ。

AI時代に機密データがGPUクラスターで処理される量が増える中、Confidential Computingはニッチなセキュリティ要件からメインストリームのインフラ標準へと進化していく。2026年現在、CoCoは内部テストから本番実用の重要な転換点にある。機密データ処理を扱うビジネスなら、今すぐ技術評価を始める価値がある。

> **注目の数字**：Intel TDXはAWS/Azure/GCPで全面サポート済み。CoCo v0.18はK8s 1.29+対応。GPU TEEサポート（TDX + NVIDIA H100）は2025年末にBeta入り。

---

*参考：CNCF Confidential Containers公式ドキュメント、RATS RFC 9334、Trustee GitHub、Kata Containersドキュメント*

