# GitHub Actions コーディングルール

このドキュメントはGitHub Actionsワークフローを実装する際のコーディングルールを定義します。
Claude Codeはこのルールに従ってGitHub Actionsワークフローの実装をアシストしてください。

---

## 1. ファイル・ディレクトリ構成

- ワークフローファイルは `.github/workflows/` 配下に配置する
- ファイル名はケバブケースで、目的が明確になる名前にする（例: `ci-build.yml`, `deploy-staging.yml`）
- 再利用可能なワークフローは `.github/workflows/reusable/` 配下にまとめる
- カスタムアクションは `.github/actions/<action-name>/action.yml` に配置する

---

## 2. ワークフロー・ジョブの命名

- `name` フィールドは必ず記述する
- ワークフロー名・ジョブ名・ステップ名は**日本語**で記述し、目的を明示する
- ジョブIDはスネークケースを使用する（例: `build_and_test`）
- ステップの `name` も省略せず、何をしているか分かる名前をつける

```yaml
# Good
name: CI パイプライン

jobs:
  build_and_test:
    name: ビルドとテスト
    steps:
      - name: Node.js のセットアップ
        uses: actions/setup-node@v4
      - name: 依存関係のインストール
        run: npm ci
```

---

## 3. トリガー（on）の設定

- トリガーは必要最小限に絞り、不要なワークフロー実行を避ける
- `push` トリガーには `branches` フィルタを必ず指定する
- `pull_request` には `paths` フィルタを活用し、無関係なファイル変更では実行しない
- `workflow_dispatch` を追加して手動実行できるようにする

```yaml
on:
  push:
    branches: [main, develop]
    paths:
      - 'src/**'
      - 'package.json'
  pull_request:
    branches: [main]
  workflow_dispatch:
```

---

## 4. 環境変数・シークレット

- ハードコードは禁止。機密情報は必ず `secrets` を使用する
- 非機密の共通値は `env` または `vars` で定義する
- シークレット名はアッパースネークケース（例: `AWS_ACCESS_KEY_ID`）
- シークレットを使用するステップにはコメントで用途を明記する

```yaml
env:
  NODE_VERSION: '20'

steps:
  - name: Deploy to AWS
    env:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }} # デプロイ用IAMキー
```

---

## 5. アクションのバージョン固定

- `uses` で指定するアクションは必ずバージョンを固定する
- サードパーティのアクションはコミットハッシュで固定することを推奨
- Dependabotでアクションのバージョンを自動更新する設定を入れる

```yaml
# Bad
- uses: actions/checkout@main

# Good
- uses: actions/checkout@v4

# Best（セキュリティ重視）
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
```

---

## 6. 権限（permissions）の最小化

- ワークフローおよびジョブレベルで `permissions` を明示的に設定する
- デフォルト権限は `read-all` または `none` とし、必要なものだけ付与する

```yaml
permissions:
  contents: read
  pull-requests: write
```

---

## 7. タイムアウトの設定

- `timeout-minutes` をジョブに設定し、ハングアップを防ぐ
- 想定実行時間の 1.5〜2 倍程度を目安にする

```yaml
jobs:
  build:
    timeout-minutes: 30
```

---

## 8. キャッシュの活用

- 依存関係のインストールには必ずキャッシュを設定する
- キャッシュキーにはロックファイルのハッシュを含める

```yaml
- name: Cache dependencies
  uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-
```

---

## 9. 条件分岐（if）の記述

- `if` 条件は明示的に記述し、暗黙的な挙動に依存しない
- デプロイ系ジョブには必ずブランチ条件を設定する
- ステップの失敗を許容する場合は `continue-on-error: true` を明示する

```yaml
- name: Deploy to production
  if: github.ref == 'refs/heads/main' && github.event_name == 'push'
```

---

## 10. 再利用性・モジュール化

- 共通処理は Composite Action または Reusable Workflow に切り出す
- 同一ワークフロー内での繰り返し処理は `matrix` 戦略を使う

---

## 11. セキュリティ対策

- `pull_request_target` の使用は原則禁止（権限昇格リスク）
- 外部からの入力を `run` に直接埋め込まない（スクリプトインジェクション対策）

```yaml
# Bad（インジェクションリスク）
- run: echo "${{ github.event.issue.title }}"

# Good（環境変数経由）
- run: echo "$ISSUE_TITLE"
  env:
    ISSUE_TITLE: ${{ github.event.issue.title }}
```

---

## 12. run ステップのスクリプト記述ルール

GitHub ActionsのジョブでインラインスクリプトをRunする場合は以下のルールに従う。

### 12-1. シェルの明示

- `shell` を必ず明示する。省略した場合のデフォルト動作はランナーOSに依存するため、意図しない挙動を防ぐ

```yaml
- name: Run script
  shell: bash
  run: echo "Hello"
```

### 12-2. bash スクリプトの安全オプション

- bash を使用する場合、スクリプトの先頭に必ず `set -euo pipefail` を記述する
  - `-e`: コマンドが失敗したら即終了
  - `-u`: 未定義変数の参照をエラーにする
  - `-o pipefail`: パイプの途中でコマンドが失敗した場合もエラーにする

```yaml
- name: Run script
  shell: bash
  run: |
    set -euo pipefail
    echo "安全なスクリプト"
```

### 12-3. 複数行スクリプトはブロックスカラーで記述

- 複数コマンドを記述する場合は `|`（リテラルブロックスカラー）を使用し、1行に複数コマンドをつなげない

```yaml
# Bad
- run: command1 && command2 && command3

# Good
- run: |
    command1
    command2
    command3
```

### 12-4. 長いスクリプトは外部ファイルに切り出す

- 概ね20行以上のスクリプトは `.github/scripts/` 配下に外部ファイルとして切り出す
- 外部スクリプトのファイル名はケバブケース（例: `deploy.sh`, `run-tests.sh`）
- スクリプトファイルには実行権限を付与し、リポジトリにコミットする

```yaml
# Bad（長いインラインスクリプト）
- run: |
    set -euo pipefail
    # 20行以上のスクリプト...

# Good（外部ファイルに切り出し）
- name: Run deploy script
  shell: bash
  run: .github/scripts/deploy.sh
```

### 12-5. 変数の参照はダブルクォートで囲む

- シェル変数の参照は単語分割・グロブ展開を防ぐため、必ずダブルクォートで囲む
- 例外として、意図的に単語分割が必要な場合はコメントで理由を記述する

```yaml
- run: |
    set -euo pipefail
    FILE_PATH="$HOME/output.txt"
    echo "Path: ${FILE_PATH}"
```

### 12-6. コマンドの出力をGitHub Actionsの出力変数に渡す

- ステップ間で値を受け渡す場合は `$GITHUB_OUTPUT` を使用する（`set-output` コマンドは非推奨）
- `$GITHUB_ENV` を使用した環境変数の受け渡しも同様に記述する

```yaml
- name: Set output value
  id: my_step
  shell: bash
  run: |
    set -euo pipefail
    VERSION=$(cat VERSION)
    echo "version=${VERSION}" >> "$GITHUB_OUTPUT"

- name: Use output value
  shell: bash
  run: echo "Version is ${{ steps.my_step.outputs.version }}"
```

### 12-7. エラーハンドリングとクリーンアップ

- リソースを確保するスクリプトは `trap` を使って確実にクリーンアップする
- 一時ファイル・ディレクトリは必ず削除処理を入れる

```yaml
- name: Run with cleanup
  shell: bash
  run: |
    set -euo pipefail
    TMPDIR=$(mktemp -d)
    trap 'rm -rf "${TMPDIR}"' EXIT

    # 処理
    cp ./artifact "${TMPDIR}/artifact"
    process "${TMPDIR}/artifact"
```

### 12-8. 機密情報の取り扱い

- スクリプト内で機密情報を `echo` しない
- デバッグ出力に機密情報が含まれないよう注意する
- secrets をスクリプトに渡す場合は必ず環境変数経由にする

```yaml
# Bad
- run: curl -H "Authorization: ${{ secrets.API_TOKEN }}" https://example.com

# Good
- run: curl -H "Authorization: ${API_TOKEN}" https://example.com
  env:
    API_TOKEN: ${{ secrets.API_TOKEN }}
```

### 12-9. スクリプトへのコメント記述

- 処理の意図が分かりにくい箇所にはインラインコメントを記述する
- スクリプトの先頭にそのステップの目的をコメントで記述する
- TODO/FIXME コメントにはチケット番号や担当者を記載する

```yaml
- name: Build Docker image
  shell: bash
  run: |
    set -euo pipefail
    # Docker イメージをビルドしてタグ付けする
    IMAGE_TAG="${GITHUB_SHA::8}" # コミットハッシュの先頭8文字をタグに使用
    docker build -t "myapp:${IMAGE_TAG}" .
    docker tag "myapp:${IMAGE_TAG}" "myapp:latest"
```

### 12-10. 冪等性（べきとうせい）の確保

- スクリプトは再実行しても同じ結果になるよう設計する
- ファイルの作成・移動・削除など副作用のある処理は事前に状態を確認する

```yaml
- name: Create output directory
  shell: bash
  run: |
    set -euo pipefail
    # 既に存在していても安全に実行できるよう -p オプションを使用
    mkdir -p ./dist/output
```

---

## 13. ログ・デバッグ

- 機密情報を `echo` などで出力しない
- デバッグ用ステップは本番マージ前に削除する
- デバッグログの制御には `runner.debug` フラグを活用する

```yaml
- name: Debug info
  if: runner.debug == '1'
  run: |
    echo "Debug mode enabled"
    env | sort
```
