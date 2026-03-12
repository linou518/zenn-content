---
title: "フロントが叩くAPIがバックエンドに存在しない――405 Method Not Allowedの原因と直し方"
emoji: "🔧"
type: "tech"
topics: ["Flask", "Python", "SPA", "デバッグ", "Web開発"]
published: true
---

自作ダッシュボードで「プロジェクト名を変更すると 405 Method Not Allowed」という報告を受けた。原因はシンプルだったが、SPAを長期運用していると必ずぶつかる"あるある"なので記録しておく。

## 症状

プロジェクト一覧ページで名前をインライン編集して保存すると、ブラウザのコンソールに `POST /api/simple-tasks/rename 405 (METHOD NOT ALLOWED)` が出る。Flask側のログにも同様の405が記録されていた。

## 原因：フロントとバックエンドの "ルート不一致"

Flask側には `/api/update-project-name` というルートが **別の文脈で** 存在していたが、フロントのJavaScriptは `/api/simple-tasks/rename` を叩いていた。

つまり：

- フロント（SPA）側で新しいAPIパスを使うコードを書いた
- バックエンド（Flask）側に対応するルートを追加し忘れた

405が返るのは、Flaskがそのパスを知らないため。404ではなく405になるケースは、同じプレフィックスの別ルートが存在するときにFlaskのルーティングが「パスは合ってるがメソッドが違う」と解釈する場合がある。

## 修正

`server.py` の simple-tasks ルート群の近くに、対応するエンドポイントを追加した。

```python
@app.route('/api/simple-tasks/rename', methods=['POST'])
def simple_tasks_rename():
    data = request.get_json()
    project_id = data.get('project')
    new_name = data.get('name')
    
    if not project_id or not new_name:
        return jsonify({'error': 'project and name required'}), 400
    
    # JSONファイルから該当プロジェクトを探して名前を更新
    tasks_data = load_simple_tasks()
    for proj in tasks_data.get('projects', []):
        if proj['id'] == project_id:
            proj['name'] = new_name
            break
    else:
        return jsonify({'error': 'project not found'}), 404
    
    save_simple_tasks(tasks_data)
    return jsonify({'ok': True})
```

## ハマりポイント：プロセスが残っていた

修正を書いてFlaskを再起動したが、依然として405が返る。

原因は **古いFlaskプロセスがポートを掴んだまま残っていた** こと。systemdの `restart` では古いプロセスが正常にkillされず、新しいプロセスが起動に失敗していた。

```bash
# 犯人を見つけて殺す
fuser -k <PORT>/tcp

# 改めて起動
systemctl --user start task-dashboard.service
```

これでようやく修正が反映された。

## 教訓：SPA + 自前バックエンドの "APIルート管理"

1000行を超えるSPAと2000行を超えるFlask backend を一人（+AI）で回していると、フロントで新機能を足したときにバックエンド側のルート追加を忘れる、というミスは起こる。

### 防ぎ方

1. **APIルートの一覧表を持つ**。`README` やプロジェクトのドキュメントにエンドポイント一覧を書いておく。新しいルートを足したら必ず更新する
2. **フロント側のfetch/axiosコールをgrepする**。定期的に `grep -r "fetch\|axios\|/api/" index.html` してバックエンドのルートと突き合わせる
3. **起動時にルート一覧を出力する**。Flaskなら `app.url_map` を使って起動ログにルート一覧を出せる
4. **プロセスの残骸に注意**。修正したのに反映されないときは、まずポートを掴んでいるプロセスを確認する（`fuser` / `lsof -i`）

```python
# Flask起動時にルート一覧を出力する例
with app.test_request_context():
    for rule in app.url_map.iter_rules():
        print(f"  {rule.methods} {rule.rule}")
```

## まとめ

405 Method Not Allowed は「ルートが存在しない」か「メソッドが違う」のどちらか。SPAで自前APIを育てていると、フロントとバックエンドの間にルートの齟齬が生まれやすい。APIのルート一覧を管理し、変更時に両方を同時に更新する習慣が大事。

そして、修正を入れたのに反映されないときは――まず `fuser` でポートを調べよう。十中八九、古いプロセスが居座っている。

<!-- 本文已脱敏処理: IP/密码/Token等敏感信息已替換 -->
