---
title: "Kubeflow Trainer v2：一つのTrainJob APIがすべてのAI訓練フレームワークを統治する"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "自動化"]
published: true
---

---
title: "Kubeflow Trainer v2：一つのTrainJob APIがすべてのAI訓練フレームワークを統治する"
date: "2026-03-24"
category: "ai"
tags: ["Kubernetes", "AI", "MLOps", "分散学習", "PyTorch", "LLM", "Kubeflow", "クラウドネイティブ"]
lang: "ja"
author: "TechsFree"
draft: false
summary: "かつてKubernetes上でのAI訓練には、PyTorch用にPyTorchJob、TF用にTFJob、MPIにはMPIJob——6種類のAPI、6種類の運用知識が必要だった。Kubeflow Trainer v2は、統一されたTrainJob APIでこの混乱に終止符を打った。TrainJobアーキテクチャ、ClusterTrainingRuntime設計、TorchTune内蔵LLMファインチューニング、v1からの移行パスを徹底解説。"
readingTime: 13
---

AIエンジニアが最も嫌うことは何か？ハイパーパラメータチューニングではない。GPUを待つことでもない。——**Kubernetes上で分散学習Jobを設定すること**だ。

PyTorchJob、TFJob、MPIJob、XGBoostJob、PaddleJob、JAXJob。6種類のCRD、6種類のYAML形式、6種類の運用知識体系。フレームワークを変えるたびに、新しいAPIを学び直す必要がある。さらに不合理なことに、各APIの背後でGang SchedulingやFailure Restartなど、K8sエコシステムにすでにソリューションが存在する機能を独自実装していた。

2025年7月、Kubeflow Trainer v2.0が正式リリースされ、この混乱に終止符が打たれた。

---

## はじめに：v1の問題はどこにあったか

Training Operator v1（Kubeflowの前世代）は、各フレームワークのOperatorを強引に組み合わせたものだった。根本的な問題は以下の通り：

| 問題 | 具体的な症状 |
|------|------------|
| フレームワーク爆発 | 6種類の独立したCRD、6種類の異なるAPI |
| 学習コスト高 | AIエンジニアがPod/コンテナ仕様を理解する必要 |
| 拡張困難 | 新フレームワーク追加にOperatorコアの変更が必要 |
| 車輪の再発明 | Gang SchedulingなどK8sエコシステム既存機能を独自実装 |
| クローズドソース不可 | v1はコミュニティのフレームワーク前提、プライベートフレームワーク未対応 |

v2の目標は明確：**AIエンジニアからK8sの複雑さを抽象化し、単一のTrainJob APIですべてのフレームワークをカバーする**。

---

## 一、アーキテクチャの核心：役割分離

v2は明確なロール境界を設計した：

```
┌─────────────────────────────────────────────────────────────┐
│  プラットフォーム管理者（Platform Administrator）               │
│  → ClusterTrainingRuntime / TrainingRuntimeを管理           │
│  → 学習インフラテンプレートを定義（イメージ/リソース/分散設定） │
└──────────────────────────┬──────────────────────────────────┘
                           │ runtimeRef（参照）
┌──────────────────────────▼──────────────────────────────────┐
│  AI エンジニア（AI Practitioner）                              │
│  → TrainJob作成（Runtimeを参照、K8sの詳細不要）               │
│  → またはPython SDK：TrainerClient.train()                    │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、3つのCRD、1つの体系

### TrainJob：AIエンジニアのインターフェース

完全なLLMファインチューニングJobの例：

```yaml
apiVersion: trainer.kubeflow.org/v1alpha1
kind: TrainJob
metadata:
  name: llm-finetune-job
  namespace: team-ml
spec:
  runtimeRef:
    kind: ClusterTrainingRuntime
    name: torch-distributed

  trainer:
    numNodes: 4
    image: pytorch/pytorch:2.7.1-cuda12.8-cudnn9-runtime
    command: ["python", "/workspace/train.py"]
    resourcesPerNode:
      requests:
        nvidia.com/gpu: "4"
      limits:
        nvidia.com/gpu: "4"

  # HuggingFaceからデータセットとモデルを自動ダウンロード
  initializer:
    dataset:
      storageUri: "hf://tatsu-lab/alpaca"
    model:
      storageUri: "hf://meta-llama/Llama-3.2-7B-Instruct"
```

AIエンジニアが書くのは**学習ロジックに関連するパラメータのみ**。Podsの作成方法、torchrunの起動方法、rendezvousの設定——すべてRuntimeに任せる。

### 内蔵ClusterTrainingRuntime一覧

| Runtime | フレームワーク | 説明 |
|---------|-------------|------|
| `torch-distributed` | PyTorch DDP/FSDP | 標準分散学習 |
| `deepspeed-distributed` | DeepSpeed | ZeROシリーズ大規模モデル学習 |
| `torchtune-llama3.2-7b` | TorchTune | 内蔵LLMファインチューニング（コード不要！）|
| `torch-distributed-with-cache` | PyTorch + Cache | v2.1新機能データキャッシュ |
| `mlx-distributed` | MLX | Apple Siliconクラスター |

---

## 三、Python SDK：ローコード学習

```python
from kubeflow.trainer import TrainerClient

client = TrainerClient()

def train_func():
    import torch
    import torch.distributed as dist
    
    dist.init_process_group(backend='nccl')
    model = MyModel()
    
    for epoch in range(10):
        # 学習ループ
        pass

# 1行でK8sクラスターへ投入
job = client.train(
    func=train_func,
    num_nodes=4,
    resources_per_node={"gpu": "4", "memory": "32Gi"},
    runtime_ref="torch-distributed",
)

# リアルタイムで学習ログを確認
client.get_job_logs(name=job.name)
```

SDKが自動処理するもの：コードのConfigMapへのシリアライズ → Podへの注入 → 分散環境変数の設定 → torchrunの起動。AIエンジニアはK8sの存在を意識しない。

---

## 四、内蔵LLMファインチューニング：BuiltinTrainer + TorchTune

v2.1のキラー機能——学習コードを一行も書かずにLLMをファインチューニング：

```yaml
apiVersion: trainer.kubeflow.org/v1alpha1
kind: TrainJob
metadata:
  name: llama-sft
spec:
  runtimeRef:
    kind: ClusterTrainingRuntime
    name: torchtune-llama3.2-7b  # 内蔵LLMファインチューニングRuntime

  initializer:
    dataset:
      storageUri: "hf://tatsu-lab/alpaca"
    model:
      storageUri: "hf://meta-llama/Llama-3.2-7B-Instruct"

  trainer:
    numNodes: 2
    resourcesPerNode:
      requests:
        nvidia.com/gpu: "4"
```

TorchTune BuiltinTrainerが自動処理：QLoRA/LoRAファインチューニング、混合精度学習、FSDPシャーディング、チェックポイント保存。

---

## 五、v1からの移行：3ステップで完了

```yaml
# v1（PyTorchJob）— 以前の書き方
apiVersion: kubeflow.org/v1
kind: PyTorchJob
spec:
  pytorchReplicaSpecs:
    Master:
      replicas: 1
      template:
        spec:
          containers:
          - name: pytorch
            image: pytorch/pytorch:2.4.0
            resources:
              limits:
                nvidia.com/gpu: 4
    Worker:
      replicas: 3
      ...

# v2（TrainJob）— 同等の記述、よりシンプル
apiVersion: trainer.kubeflow.org/v1alpha1
kind: TrainJob
spec:
  runtimeRef:
    kind: ClusterTrainingRuntime
    name: torch-distributed
  trainer:
    numNodes: 4        # Master + 3 Workers = 4ノード
    resourcesPerNode:
      limits:
        nvidia.com/gpu: "4"
```

---

## まとめ：AIインフラが成熟する時代

Kubeflow Trainer v2は、AIインフラコミュニティが「各自で車輪を再発明する」段階から「標準レイヤーを共同構築する」段階に移行したことを示すシグナルだ。

核心的な価値はAPI統一だけではない。**関心の分離**——AIエンジニアはアルゴリズムに集中し、インフラエンジニアはプラットフォームを管理し、K8sエコシステム（JobSet、Kueue）が重複機能を担う。

2026年、LLMファインチューニング需要の爆発的増加とともに、Kubeflow Trainer v2はKubernetes上のAI学習のデファクトスタンダードになりつつある。チームがまだPyTorchJob YAMLを手書きしているなら、移行を検討する時期だ。

> **注目の数字**：v2.1はK8s 1.29+、Kueue 0.9+、PyTorch 2.7をサポート。Python SDKはPyTorch公式エコシステム認定済み（2025/07）。TAS（Topology-Aware Scheduling）でNVLinkバンド幅を最大活用。

---

*参考：Kubeflow Trainer v2公式ドキュメント、JobSetプロジェクト、Kueueドキュメント、PyTorch分散学習ガイド*

