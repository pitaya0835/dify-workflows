# CLAUDE.md

このファイルは、このリポジトリで Dify ワークフロー DSL（`workflows/*.yml`）を編集・レビューする際のガイドです。
Claude Code や他の開発者が YAML を直接編集・確認するときの参考にしてください。

## このリポジトリの位置づけ

- Dify 本体（アプリケーション）はここには含まれません。実行・動作確認は別途ローカルに立てた Dify 上で行います（`README.md` 参照）。
- ここで管理するのは Dify の「DSL エクスポート/インポート」機能で扱われる YAML ファイルのみです。
- 基本の流れ: **Dify GUI で編集・実行確認 → エクスポート → このリポジトリにコミット**。YAML を直接手で書き換えることもできますが、必ず Dify に再インポートして動作確認してからコミットしてください（手書きの YAML はノードIDの不整合など壊れやすいミスが起きがちです）。

## DSL の基本構造

Dify のワークフロー DSL は次のようなトップレベル構造を持ちます。

```yaml
app:
  description: ''
  icon: 🤖
  icon_background: '#FFEAD5'
  mode: workflow        # workflow（バッチ的な単発実行アプリ）or chat / advanced-chat 等
  name: アプリ名
  use_icon_as_answer_icon: false
kind: app               # 固定値
version: 0.1.5          # DSLフォーマットのバージョン。Difyのバージョンによって値が異なることがある
workflow:
  conversation_variables: []   # チャット系アプリで使う会話変数（workflowモードでは通常空）
  environment_variables: []    # ワークフロー全体で使う環境変数
  features: { ... }            # ファイルアップロード可否、開始メッセージ等のUI機能設定
  graph:
    nodes: [ ... ]              # ノード（処理ブロック）の配列
    edges: [ ... ]              # ノード同士をつなぐ配線の配列
```

編集対象になるのはほぼ常に `workflow.graph.nodes` と `workflow.graph.edges` です。

### nodes

各ノードは概ね次の形をしています。

```yaml
- id: llm                 # グラフ内で一意なノードID（edgeのsource/targetから参照される）
  type: custom             # Dify UI 上の描画種別。ノード種別は data.type 側で指定する
  position: { x: 400, y: 280 }
  width: 244
  height: 120
  data:
    type: llm               # ← 実際のノード種別（下記「よく使うnode type」参照）
    title: LLM 回答生成      # UI上に表示される名前
    desc: ''
    # ここから先は type ごとに異なるフィールド（後述）
```

- `id` は英数字・アンダースコア程度のシンプルな文字列で構いません（Dify が自動採番するとタイムスタンプ風の数値文字列になりますが、手編集する場合は分かりやすい名前で問題ありません）。
- `position` / `width` / `height` はキャンバス上の見た目の情報です。無くてもインポートは通ることが多いですが、ノードが重なって見づらくなるため、手で追加するノードにはおおよその座標を与えておくことを推奨します。

### edges

```yaml
- id: start-llm
  source: start          # 接続元ノードのid
  sourceHandle: source    # 接続元のハンドル名（下記参照）
  target: llm             # 接続先ノードのid
  targetHandle: target    # 接続先のハンドル名（通常は固定で "target"）
  type: custom
  data:
    sourceType: start      # 接続元ノードの data.type と一致させる
    targetType: llm         # 接続先ノードの data.type と一致させる
    isInIteration: false
```

- 通常のノード（start, llm, code, knowledge-retrieval, answer など）から出る配線の `sourceHandle` は `source` 固定です。
- `if-else` ノードだけは分岐があるため、`sourceHandle` に `true`（条件に一致した場合）/ `false`（一致しなかった場合＝else）を使います。複数条件（elif相当）を使う場合は `cases` の `case_id` の値がそのまま `sourceHandle` になります。
- `targetHandle` は基本的に常に `target` です。

## よく使う node type（`data.type`）

| type | 役割 | 主なフィールド |
|---|---|---|
| `start` | ワークフローの入口。ユーザー入力変数を定義する | `variables: [{variable, label, type, required, max_length}]` |
| `llm` | LLMを呼び出す | `model: {provider, name, mode, completion_params}`, `prompt_template: [{role, text}]`, `context`, `vision` |
| `if-else` | 条件分岐 | `cases: [{case_id, logical_operator, conditions: [{variable_selector, comparison_operator, value}]}]` |
| `knowledge-retrieval` | ナレッジベース（Dataset）からの検索 | `query_variable_selector`, `dataset_ids`, `retrieval_mode` |
| `code` | Python/JavaScriptコードを実行 | `code_language`, `code`, `variables`（入力）, `outputs`（出力の型定義） |
| `end` | ワークフローの出口（`workflow` モード用）。出力変数を定義する | `outputs: [{variable, value_selector: [nodeId, fieldName]}]` |
| `answer` | チャット系アプリ（chat / advanced-chat）で回答を返す | `answer: "{{#nodeId.field#}}"` のようなテンプレート文字列 |

補足:
- `workflow`（単発実行）モードのアプリは `end` ノードで終端します。`chat` / `advanced-chat` モードのアプリは `answer` ノードを使います。このリポジトリのサンプルは `mode: workflow` + `end` ノードで統一しています。
- `if-else` で分岐した先をどう合流させるか: 各分岐の末尾にそれぞれ `end`（または `answer`）ノードを置くのが最も単純で分かりやすい方法です（`sample-branching-workflow.yml` はこの形）。1つの `end` に複数分岐を合流させたい場合は、間に「変数集約（Variable Aggregator）」ノードを挟んで出力変数を1本化する必要があります（分岐は排他的に実行されるため、`end` の `value_selector` が特定ノードを直接指していると、そのノードが実行されなかった分岐では値が空になってしまうため）。

## edge の source/target 整合性

編集時に壊れやすいポイントなので、変更のたびに以下を確認してください。

1. すべての edge の `source` / `target` が、`nodes` に実在する `id` と一致していること。
2. `data.sourceType` / `data.targetType` が、対応するノードの `data.type` と一致していること。
3. `if-else` ノードから出る edge の `sourceHandle` が、`true` / `false`（または該当する `case_id`）になっていること。それ以外のノードからの edge は `sourceHandle: source` であること。
4. ノードを削除したら、そのノードを `source` または `target` にしている edge も必ず削除すること（浮いた edge が残っていると Dify 側でインポートエラーになります）。
5. ノードを追加したら、そのノードへの入り口（前段からの edge）と、必要なら出口（後段への edge）の両方を追加すること。

## `{{#ノードID.変数名#}}` 形式の変数参照

- LLM のプロンプトや Answer ノードのテンプレート文字列の中では、他ノードの出力を `{{#nodeId.variableName#}}` という形式で埋め込みます。
  - 例: `{{#start.query#}}` → `start` ノード（Start）の `query` という入力変数を参照。
  - 例: `{{#llm.text#}}` → `llm` ノード（LLM）が生成したテキスト出力を参照。
- YAML の構造化フィールド（`value_selector` など）では、文字列テンプレートではなく配列 `[nodeId, variableName]` で同じ参照を表現します。
  - 例: `end` ノードの `outputs[].value_selector: [llm, text]` は、上記の `{{#llm.text#}}` と同じ参照先を指します。
- 参照先のノードIDが存在しない、または該当ノードにその変数名が無い場合、Dify 上で警告・エラーになります。ノードIDをリネームしたときは、参照している全箇所（他ノードのプロンプト、`value_selector`、`if-else` の `variable_selector` など）を漏れなく更新してください。
- LLM ノード自身が持つ代表的な出力変数は `text`（生成結果の文字列）です。`code` ノードの出力変数名は `outputs` で自分で定義した名前になります。

## 編集時の注意点

- **座標・サイズの崩れ**: `position` / `width` / `height` を手で調整しなくてもインポート自体は通りますが、Dify の画面上でノードが重なって見づらくなることがあります。ノードを追加する際はおおよそでよいので他ノードとずらした座標を与えてください。
- **バージョン差異**: `version:`（DSLフォーマットバージョン）や `features:` の中身は、Dify のバージョンによって項目が増減することがあります。手元の Dify からエクスポートした YAML と大きく構造が異なる場合は、エクスポートされた方の構造を正として書き換えてください。
- **モデル名・プロバイダ**: サンプルの `provider: openai` / `name: gpt-4o-mini` は例示です。手元の環境で有効化されているモデルプロバイダに合わせて書き換える必要があります（インポート後にモデル未設定エラーが出た場合は、LLM ノードを開いてモデルを選び直し、再エクスポートしてください）。
- **手書き編集は最小限に**: YAML を直接編集するのは、プロンプト文言の微調整や条件式の値変更など、影響範囲が明確なものに留めるのが安全です。ノード構成そのものを大きく変える場合は、Dify の GUI 上で編集してからエクスポートし直すことを推奨します。
- **インポート前の整合性チェック**: コミット前に、`source`/`target` が実在する `id` を指しているか、`{{#...#}}` 参照が存在する変数を指しているかを目視確認してください。

## 試行錯誤のサイクル（要約）

1. `workflows/*.yml` を Dify にインポートする（または既存アプリを開く）。
2. Dify の GUI 上でノード・エッジ・プロンプトを編集する。
3. テスト実行で、想定した入力に対して期待した出力が得られるか確認する（分岐がある場合は各分岐を個別に確認）。
4. 問題なければ DSL をエクスポートし、`workflows/` 配下の該当ファイルを上書きする。
5. 差分（`git diff`）を確認し、変更内容が分かるコミットメッセージでコミット・プッシュする。

詳しい手順は `README.md` を参照してください。
