# Terraformコーディング規約

このドキュメントは、engineering-playbookプロジェクトにおけるTerraformコードの記述規約を定義します。

## 目次

- [jsonencode使用ルール](#jsonencode使用ルール)
- [aws_iam_policy_documentを使う理由](#aws_iam_policy_documentを使う理由)

---

## jsonencode使用ルール

**基本原則**: `jsonencode`は原則禁止。使用する場合は**必ずコメントで理由を明記**すること。

### jsonencode禁止のケース

以下の用途では、`jsonencode`の代わりに`aws_iam_policy_document`データソースを使用してください。

| 用途 | 代替手段 |
|------|----------|
| IAMポリシー（assume_role_policy, policy） | `aws_iam_policy_document` データソース |
| S3バケットポリシー | `aws_iam_policy_document` データソース |
| SNSトピックポリシー | `aws_iam_policy_document` データソース |
| SQSキューポリシー | `aws_iam_policy_document` データソース |
| KMSキーポリシー | `aws_iam_policy_document` データソース |
| CloudWatch Logsリソースポリシー | `aws_iam_policy_document` データソース |
| Secrets Managerリソースポリシー | `aws_iam_policy_document` データソース |
| Lambda関数ポリシー | `aws_iam_policy_document` データソース |
| ECRリポジトリポリシー | `aws_iam_policy_document` データソース |

### jsonencode許容のケース（コメント必須）

以下のケースでは`jsonencode`の使用が許容されますが、**必ず理由コメントを付けてください**。

| 用途 | 理由 |
|------|------|
| EventBridge `event_pattern` | 専用データソースが存在しない |
| Step Functions `definition` | 専用データソースが存在しない |
| API Gateway `body`（OpenAPI定義） | 専用データソースが存在しない |
| ECSタスク定義 `container_definitions` | 専用データソースが存在しない |
| Lambda環境変数（JSON文字列値） | 環境変数の値としてJSON文字列を設定する場合（例: 複雑な設定オブジェクトをJSON文字列として渡す） |

### コメントテンプレート（コピペ用）

**注意**: 既存のドキュメントでは「jsonencodeの使用が妥当です（CLAUDE.md参照）」という表現を使用している箇所があります。新規作成時は以下の簡潔なテンプレートを使用してください。既存ドキュメントの表現は、更新時に順次統一していきます。

```hcl
# ----------------------------------------
# jsonencode使用時の必須コメント（以下からコピペ）
# ----------------------------------------

# EventBridge event_pattern用
# 注意: event_patternは専用データソースが存在しないため、jsonencodeを使用

# Step Functions definition用
# 注意: Step Functions定義は専用データソースが存在しないため、jsonencodeを使用

# ECSタスク定義 container_definitions用
# 注意: container_definitionsは専用データソースが存在しないため、jsonencodeを使用

# API Gateway body用
# 注意: OpenAPI定義は専用データソースが存在しないため、jsonencodeを使用

# その他のケース用（理由を具体的に記載）
# 注意: <具体的な理由>のため、jsonencodeを使用
```

### セルフチェックリスト

Terraformコードを書いたら、以下を確認してください：

- [ ] `jsonencode`を使用している箇所をすべて特定したか
- [ ] 各`jsonencode`に対して、禁止ケースでないか確認したか
- [ ] 許容ケースの場合、理由コメントを付けたか
- [ ] 許容ケースでjsonencodeを使用した箇所すべてにコメントを付けたか
- [ ] IAMポリシーに`jsonencode`を使っていないか（最重要）

### 良い例・悪い例

#### 悪い例（IAMポリシーでjsonencodeを使用）

```hcl
resource "aws_iam_role" "example" {
  name = "example-role"

  # NG: IAMポリシーにjsonencodeを使用している
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "lambda.amazonaws.com"
      }
    }]
  })
}
```

#### 良い例（aws_iam_policy_documentを使用）

```hcl
data "aws_iam_policy_document" "example_assume_role" {
  statement {
    effect = "Allow"

    principals {
      type        = "Service"
      identifiers = ["lambda.amazonaws.com"]
    }

    actions = ["sts:AssumeRole"]
  }
}

resource "aws_iam_role" "example" {
  name               = "example-role"
  assume_role_policy = data.aws_iam_policy_document.example_assume_role.json
}
```

#### 良い例（EventBridgeでjsonencodeを使用 + コメント付き）

```hcl
resource "aws_cloudwatch_event_rule" "example" {
  name        = "example-rule"
  description = "Example EventBridge rule"

  # 注意: event_patternは専用データソースが存在しないため、jsonencodeを使用
  event_pattern = jsonencode({
    source      = ["aws.s3"]
    detail-type = ["Object Created"]
  })
}
```

---

## aws_iam_policy_documentを使う理由

`jsonencode`ではなく`aws_iam_policy_document`データソースを使用する理由：

1. **Terraformネイティブな記述で可読性が高い**
   - HCLの構文に沿った自然な記述
   - ポリシーの構造をより明確に表現できる

2. **IDEサポートが充実**
   - HCLの構文チェックが効く
   - IDEの補完機能が使える
   - `terraform fmt`で自動整形される

3. **エラー検出が容易**
   - JSONの構造エラーをTerraformの構文チェック段階で検出
   - タイプミスや構造の誤りを早期に発見

4. **柔軟な組み合わせが可能**
   - 複数のステートメントを組み合わせる際のオーバーライド機能が使える
   - `override_policy_documents`: 既存ポリシーを上書き
   - `source_policy_documents`: 複数のポリシーをマージ

5. **保守性の向上**
   - 変更時の差分が明確
   - コードレビューがしやすい

### 使用例：複数ポリシーの組み合わせ

```hcl
# ベースポリシー
data "aws_iam_policy_document" "base" {
  statement {
    effect = "Allow"
    actions = ["s3:GetObject"]
    resources = ["arn:aws:s3:::example-bucket/*"]
  }
}

# 追加ポリシー
data "aws_iam_policy_document" "additional" {
  statement {
    effect = "Allow"
    actions = ["s3:PutObject"]
    resources = ["arn:aws:s3:::example-bucket/*"]
  }
}

# 組み合わせ
data "aws_iam_policy_document" "combined" {
  source_policy_documents = [
    data.aws_iam_policy_document.base.json,
    data.aws_iam_policy_document.additional.json,
  ]
}

resource "aws_iam_policy" "example" {
  name   = "example-policy"
  policy = data.aws_iam_policy_document.combined.json
}
```

---

## 関連ドキュメント

- [CLAUDE.md](../../CLAUDE.md) - プロジェクト固有のガイドライン
- [Terraform公式ドキュメント: aws_iam_policy_document](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/iam_policy_document)
