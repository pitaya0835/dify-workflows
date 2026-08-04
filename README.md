# dify-workflows

[Dify](https://github.com/langgenius/dify) のワークフロー DSL（YAML）をバージョン管理するためのリポジトリです。
実際のワークフローの中身は Dify の画面上（GUI）で組み立て、動作確認できたものを YAML としてエクスポートし、このリポジトリにコミットして管理します。

## リポジトリ構成

```
.
├── README.md          # このファイル
├── CLAUDE.md          # Dify workflow DSL のガイド（Claude Code 向け）
├── .gitignore
└── workflows/
    ├── sample-qa-workflow.yml         # Start → LLM → End の最小構成サンプル
    └── sample-branching-workflow.yml  # Start → IF/ELSE → LLM(×2) → End の分岐サンプル
```

## 1. Dify 本体をローカルで起動する

ワークフローの動作確認には、Dify 本体（Web UI + API + Worker + DB 等）が必要です。
公式リポジトリの docker-compose を使ってローカルに一式立ち上げます。

```bash
# 1. 公式リポジトリを別ディレクトリに clone する
#    （このリポジトリとは別物。ワークフローYAMLの管理専用がこちらのリポジトリ）
git clone https://github.com/langgenius/dify.git
cd dify/docker

# 2. .env を用意する
cp .env.example .env
# 必要に応じて .env を編集する（ポート番号、モデルプロバイダの設定など）

# 3. コンテナ群を起動する
docker compose up -d

# 4. 状態確認
docker compose ps
```

起動後、ブラウザで `http://localhost/install` にアクセスし、初回セットアップ（管理者アカウント作成）を行います。
セットアップが完了すると `http://localhost` からログインできるようになります。

停止する場合:

```bash
docker compose down
```

データを含めて完全に削除する場合（ボリュームも消えるので注意）:

```bash
docker compose down -v
```

### モデルプロバイダの設定

LLM ノードを動かすには、少なくとも1つのモデルプロバイダ（OpenAI, Azure OpenAI, Anthropic など）を Dify の「設定 > モデルプロバイダ」から有効化し、APIキーを登録しておく必要があります。
このリポジトリのサンプル YAML は `provider: openai` / `name: gpt-4o-mini` を例として使っていますが、手元の環境で使えるプロバイダ・モデル名に読み替えてください（インポート後、LLM ノードを開いてモデルを選び直せば反映されます）。

## 2. 試行錯誤サイクル（インポート → 実行確認 → エクスポート → コミット）

このリポジトリのワークフローは、次のサイクルで育てていきます。

1. **インポート**
   Dify の画面右上「スタジオ」→ 対象アプリを開く（無ければ「最初から作成」→「インポート」）→
   `workflows/*.yml` の中身を DSL インポート画面に貼り付ける、または YAML ファイルをアップロードする。

2. **編集・実行確認**
   Dify のワークフローエディタ上でノードやプロンプトを調整し、画面右上の「実行」（テスト実行）で動作確認する。
   - 入力変数（Start ノードで定義したもの）を与えて実行し、期待した出力（End / Answer ノードの内容）になるか確認する。
   - IF/ELSE などの分岐がある場合は、両方の分岐を通るテストケースをそれぞれ試す。

3. **エクスポート**
   動作確認が取れたら、エディタ右上のメニューから「DSLをエクスポート」を選び、YAML をダウンロードする。

4. **コミット**
   ダウンロードした YAML で `workflows/` 配下の対象ファイルを上書きし、差分を確認してコミットする。

   ```bash
   git add workflows/sample-qa-workflow.yml
   git commit -m "Update sample-qa-workflow: プロンプト調整"
   git push
   ```

この 1〜4 を繰り返すことで、「Dify 上での試行錯誤」と「Git 上での変更履歴管理」を両立させます。
DSL の構造や編集時の注意点については [CLAUDE.md](./CLAUDE.md) を参照してください。

## 参考

- Dify 本体: https://github.com/langgenius/dify
- Dify 公式ドキュメント: https://docs.dify.ai/
