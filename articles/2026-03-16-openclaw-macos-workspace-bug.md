---
title: "OpenClaw macOS で workspace 設定が無視されるバグと workaround"
emoji: "🐛"
type: "tech"
topics: ["OpenClaw", "macOS", "multiagent", "インフラ運用", "バグ修正"]
published: true
---

OpenClaw で Mac Mini M1 を新規ノードとして追加した際、`agents.list[].workspace` の設定がまったく効かないという問題にハマった。同じ罠を踏む人がいるかもしれないので、調査過程と workaround を記録しておく。

## 症状

Mac Mini（macOS Ventura）に OpenClaw 2026.3.13 をインストールし、agent を追加。config で workspace を明示的に指定：

```yaml
agents:
  list:
    - id: my-agent
      workspace: /Users/openclaw/.openclaw/agents/my-agent/workspace
```

gateway を再起動するたびに、**設定を無視して** `~/.openclaw/workspace/` が自動生成され、セッションの `cwd` もそこを指す。workspace ディレクトリには出荷時のデフォルトファイル（AGENTS.md, SOUL.md, TOOLS.md 等）が毎回展開される。

つまり、agent 用に用意したカスタム workspace が一切使われない。

## 検証：Linux では正常

同日、同じバージョン 2026.3.13 を Linux（Ubuntu 24.04）の別ノードにもセットアップしていた。こちらは `agents.list[].workspace` を絶対パスで指定すれば**正常に動作**する。デフォルトの `~/.openclaw/workspace/` は作成すらされない。

つまり **macOS 固有の問題**。

## 試したこと（全部ダメ）

1. `agents.list[].workspace` に絶対パス → ❌
2. `agents.defaults.workspace` にフォールバック指定 → ❌
3. `agentDir` フィールドを追加 → ❌（そもそもこのフィールドは存在しない）
4. `openclaw agents add my-agent --workspace /path` で CLI から作成 → ❌
5. `~/.openclaw/workspace/` を毎回削除して再起動 → ❌（毎回復活する）

ソースコードの `resolveAgentWorkspaceDir()` を読むと、ロジック自体は正しく workspace 設定を参照している。しかし macOS の code path のどこかで、この関数の結果が上書きされている模様。

## workaround：symlink + chflags

デフォルトパスを強制的に正しい場所に向ける力技：

```bash
# デフォルト workspace を削除
rm -rf ~/.openclaw/workspace

# 本来使いたい workspace へ symlink
ln -s /Users/openclaw/.openclaw/agents/my-agent/workspace ~/.openclaw/workspace

# macOS の不可変フラグで symlink 自体を保護（上書き防止）
sudo chflags -h uchg ~/.openclaw/workspace
```

`chflags -h uchg` は macOS 固有のコマンドで、symlink そのものに immutable フラグを付ける。これにより gateway が再起動しても symlink を削除・上書きできなくなる。

### 結果

gateway 再起動後、セッションの `cwd` が正しい workspace を指すことを確認。カスタム SOUL.md や TOOLS.md も正しく読み込まれた。✅

## 制限事項

この workaround は **シングル agent ノード限定**。`~/.openclaw/workspace/` という一つのパスを一つの agent workspace に向けるだけなので、macOS 上で複数 agent を動かす場合は使えない。

根本修正は OpenClaw 側で対応が必要。GitHub Issue を作成済みなので、今後のバージョンで修正されることを期待。

## 教訓

- **workspace パスは必ず絶対パスで指定する**。`~` はどの OS でも展開されない（これは仕様）。
- **macOS と Linux で挙動が異なることがある**。同じバージョンでもプラットフォーム固有のバグは存在する。
- **新規ノード追加時は、セッション JSONL の `cwd` を必ず確認**。config が正しくても実際の挙動が違うことがある。
- `chflags` は macOS 運用の強い味方。パッケージマネージャやデーモンによるファイル上書きを防ぎたい場面で使える。

## 環境

- OpenClaw: 2026.3.13
- macOS: Ventura（Mac Mini M1）
- 正常動作確認: Ubuntu 24.04

<!-- 本文已脱敏処理: agent名等を一般化 -->
