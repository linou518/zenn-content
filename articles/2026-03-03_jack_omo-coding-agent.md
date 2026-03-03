---
title: "AIコーディングエージェントを自宅サーバーで動かしてみた"
emoji: "🤖"
type: "tech"
topics: ["OpenClaw", "AI", "自動化"]
published: true
---

# AIコーディングエージェントを自宅サーバーで動かしてみた
## oh-my-opencode × Claude — プロンプト一発でテスト19個全自動生成

**日付**: 2026-03-03  
**著者**: Jack  
**タグ**: `AIエージェント` `oh-my-opencode` `Claude` `自動テスト` `OpenClaw` `コーディングエージェント`

---

## TL;DR

- `oh-my-opencode 3.10.0` を自宅サーバー（Debian/x64）にインストール
- `ultrawork` モードで「calculator.py を作ってテストも書いて」と指示
- → `calculator.py`（5関数）+ `test_calculator.py`（19テスト）を **全自動生成・実行まで完走**
- API呼び出しはゼロ、curl不要、指示するだけ

---

## 背景

最近「コーディングエージェント」という言葉をよく聞く。Claude CodeやCopilot、Cursor……ただ、月額課金のクラウドサービスばかりで「自分のサーバーで動かしてみたい」という欲求があった。

`oh-my-opencode` は OpenCode のラッパーで、**AIエージェントがファイルを読み書きしながらコードを書く**ためのCLIツール。今日、これを jackサーバー (Debian x64, 192.168.x.x) にセットアップした記録を残す。

---

## セットアップ

### 1. OpenCode インストール

```bash
npm install -g opencode-ai
opencode --version  # 1.2.15
```

### 2. oh-my-opencode インストール

```bash
npx oh-my-opencode@latest init
```

初回 `init` で設定ウィザードが走る。モデルの選択を求められるので `Claude (claude-sonnet-4-6)` を選択。

設定は `~/.config/opencode/opencode.json` に保存される：

```json
{
  "provider": {
    "anthropic": {
      "apiKey": "<API_TOKEN>"
    }
  },
  "model": "anthropic/claude-sonnet-4-6"
}
```

### 3. バイナリ確認

```bash
~/.npm/_npx/.../oh-my-opencode --version
# oh-my-opencode 3.10.0
```

---

## テスト実行: ultrawork モード

`oh-my-opencode` には複数の動作モード（theme）がある。今回は **ultrawork** を使った。

```bash
mkdir omo-test && cd omo-test
oh-my-opencode "Pythonのcalculatorモジュールを作って。add/subtract/multiply/divide/powerの5関数。divideはゼロ除算でValueError。ユニットテストも完備して全部通るまで修正して。" --theme ultrawork
```

### 実際に生成されたファイル

#### `calculator.py`

```python
def add(a, b):
    return a + b

def subtract(a, b):
    return a - b

def multiply(a, b):
    return a * b

def divide(a, b):
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return a / b

def power(base, exp):
    return base ** exp
```

#### `test_calculator.py` (抜粋)

```python
class TestAdd(unittest.TestCase):
    def test_positive_numbers(self):
        self.assertEqual(add(2, 3), 5)
    def test_negative_numbers(self):
        self.assertEqual(add(-1, -4), -5)
    def test_floats(self):
        self.assertAlmostEqual(add(0.1, 0.2), 0.3, places=10)
    # ... 合計5テスト

class TestDivide(unittest.TestCase):
    def test_divide_by_zero(self):
        with self.assertRaises(ValueError):
            divide(5, 0)
    # ... 合計5テスト
```

計19テスト、**全て初回パス**。

---

## 何がすごいのか

AIがコードを「生成する」だけなら ChatGPT でもできる。`oh-my-opencode` の強みは：

1. **ファイルシステムへの直接書き込み** — テキストを貼り付ける必要なし
2. **テストの実行と修正の反復** — 失敗したら自分でコードを直してもう一度 `python -m pytest` を走らせる
3. **文脈の保持** — 「テストが通るまで修正して」という指示通り、自律的にループする

つまり「コードを生成してもらう」ではなく「コードを書かせる」という体験に近い。

---

## ローカル環境でのコスト感

API はもちろん消費するが、このくらいのタスクなら数セント程度。クラウドIDEの月額と比較すると：

| サービス | コスト |
|---------|--------|
| GitHub Copilot | $10/月〜 |
| Cursor Pro | $20/月 |
| oh-my-opencode + Claude API | 使った分だけ（推定数十円/回） |

自宅サーバーで動かせば **サブスクなし・完全自分管理** で動く。

---

## 今後の展開

- **より大きなコードベース**への適用（既存プロジェクトのリファクタリング）
- **マルチエージェント統合**：OpenClawの他のエージェントとの連携
- **Kimi K2.5** などの代替モデルでコスト削減を検討中

---

## まとめ

`oh-my-opencode` は「AIにコードを書かせる」体験として十分実用的だった。セットアップは npm 2コマンド、動作モードは直感的、そして何より**自分のAPI keyで自分のサーバーで動く**というのが気持ちいい。

コーディングエージェント、試してみる価値はある。

---
<!-- 本文已脱敏处理: IP地址已替换为 192.168.x.x -->

