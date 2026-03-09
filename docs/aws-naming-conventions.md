# AWS命名規則

AWS リソースの命名に関する標準規則です。一貫した命名により、リソース管理の効率化と可読性の向上を図ります。

## 目次

- [概要](#概要)
- [基本原則](#基本原則)
- [locals定義](#locals定義)
- [リソース別命名規則](#リソース別命名規則)
- [命名時の注意事項](#命名時の注意事項)
- [例外処理](#例外処理)
- [関連ドキュメント](#関連ドキュメント)

---

## 概要

### 背景

- AWSリソースの命名規則が明確に決まっておらず、これまで実装してきたパターンを参考に感覚で名称を決定している
- AIによるコーディングやレビューの割合が増えているため、命名規則が明確にないことでコーディングにブレが生じる

### 目的

- 各リソースごとに基本となる命名規則を定義し、それに沿って名称決定を行う
- 文字数制限などの例外時には回避策としてのパターンも設定しておく
- PRレビューでの参照基準として活用する

### 対象読者

- Terraformコードを記述する開発者
- ドキュメント作成者（コード例を記載する際）
- コードレビュワー
- AIによる自動コーディング・レビューシステム

---

## 基本原則

1. **一貫性**: 同一コンポーネント内では統一した命名パターンを使用
2. **可読性**: リソース名から用途と環境が明確に判別できる
3. **拡張性**: 新しいサービスや環境に対応できる柔軟な構造
4. **AWS規約準拠**: AWSサービス固有の制約に対応

### 命名規則の構成要素

- **環境識別子** (`environment` / `env`): 本番・ステージング等の環境区分
- **サービス名** (`servicename`): アプリケーション・サービスの名称
- **機能名** (`function`): コンポーネント・機能の名称
- **顧客識別子** (`customer`): 顧客固有リソースの場合
- **用途接尾辞**: リソース種別を示すサフィックス

### 文字結合ルール

- **基本**: ハイフン (`-`) を使用
- **例外**: Glue（Athena）のテーブル名・カラム名はスネークケース (`_`) を使用
- **Terraform リソース名**: snake_case（既存規約に従う）

---

## locals定義

命名に用いるローカル変数は Terraform上の `locals.tf` で以下を定義すること（既存規約・policy と一致）。

| 変数名 | 説明 | 例 |
|--------|------|-----|
| `environment` | 環境名（フル） | `production`, `staging`, `development` |
| `env` | 環境名（短縮） | `prod`, `stg`, `dev` |
| `servicename` | サービス名 | `canly-admin`, `canlyhp-cms`, `benefits` |
| `function` | コンポーネント/機能名 | `api`, `front`, `batch`, `common` |
| `customer` | 顧客識別子（共通の場合は `common`） | `common`, `livinghouse`, `valor` |

**使用例**:
```hcl
locals {
  environment = "production"
  env         = "prod"
  servicename = "canly-admin"
  function    = "api"
  customer    = "common"
}
```

---

## リソース別命名規則

### Application Load Balancer（ALB）

**リソース**: `aws_lb`

| パターン | 形式 | 使用例 |
|----------|------|--------|
| **推奨（標準）** | `{environment}-{servicename}-{function}` | `production-canly-admin-api` |
| Internal ALB | `{environment}-{servicename}-{function}-int` | `production-canly-admin-api-int` |
| 代替（短縮） | `{env}-{servicename}-{function}` | `prod-canly-admin-api` |

### Target Group

**リソース**: `aws_lb_target_group`

| パターン | 形式 | 例 | 備考 |
|----------|------|-----|------|
| **推奨** | `substr(local.ecs_service_name, 0, 32)` | - | ECSサービス名と一致させつつ32文字制限に対応 |
| 代替 | `{env}-{servicename}-{function}` | `prod-canly-admin-api` | 32文字以内で表現 |
| 内部用 | `{environment}-{servicename}-{function}-int` | `prod-canly-admin-api-int` | 内部ALB用 |
| クライアント環境 | `{customer}` | `hotland` | 顧客単位で1TGの構成 |

### ECS

**リソース**: `aws_ecs_cluster`, `aws_ecs_service`, `aws_ecs_task_definition`

| リソース | 形式 | 例 |
|----------|------|-----|
| **クラスタ名** | `{environment}-{servicename}-{function}` | `production-canly-admin-api` |
| **サービス名** | クラスタと同一 | `production-canly-admin-api` |
| **タスクfamily** | クラスタと同一 | `production-canly-admin-api` |
| **複数機能サービス** | `{environment}-{servicename}-{function}-{sub_function}` | `production-canly-admin-worker-bulk-create-posts` |

### Security Group

**リソース**: `aws_security_group`

基本形: `{environment}-{servicename}-{function}-{AWSサービス名}`

| 用途 | 形式 | 例 |
|------|------|-----|
| **ALB用** | `{environment}-{servicename}-{function}-lb` | `production-canly-admin-api-lb` |
| **ECS用** | `{environment}-{servicename}-{function}-ecs` | `production-canly-admin-api-ecs` |
| **RDS用** | `{environment}-{servicename}-{function}-rds` | `production-canly-admin-rds` |
| **DocDB用** | `{environment}-{servicename}-{function}-docdb` | `production-canly-homepage-docdb` |
| **Lambda用** | `{environment}-{servicename}-{function}-lambda` | `production-benefits-data-lambda` |
| **VPC Endpoint用** | `{environment}-{servicename}-{function}-vpc-endpoint` | `production-canly-admin-vpc-endpoint` |
| **ElastiCache用** | `{environment}-{servicename}-{function}-valkey` | `production-canly-admin-valkey` |

### S3 Bucket

**リソース**: `aws_s3_bucket`

| 用途 | 形式 | 備考 |
|------|------|------|
| **汎用** | `{environment}-{servicename}-{function}` | 共通バケット |
| **ALBログ** | `{environment}-{servicename}-{function}-alb-log` | - |
| **CloudFrontログ** | `{environment}-{servicename}-{function}-cloudfront-log` | - |
| **WAFログ** | `aws-waf-logs-{environment}-{servicename}-{function}` | プレフィックス `aws-waf-logs-` 必須 |
| **Firehoseログ** | `{environment}-{servicename}-firehose-log` | - |
| **用途付き** | `{environment}-{servicename}-{function}-{用途}` | 例: `-images-external`, `-maintenance` |

### RDS / Aurora

**リソース**: `aws_rds_cluster`, `aws_rds_cluster_instance`

| リソース | 形式 | 例 |
|----------|------|-----|
| **クラスタ** | `{environment}-{servicename}-aurora-mysql-cluster` | `production-canly-admin-aurora-mysql-cluster` |
| **インスタンス** | `{environment}-{servicename}-{engine_name}-{count.index + 1}` | `production-canly-admin-aurora-mysql-1` |
| **DocDBクラスタ** | `{environment}-{servicename}-{function}-docdb` | `production-canly-homepage-docdb` |
| **DocDBインスタンス** | `{environment}-{servicename}-{function}-docdb-{count.index + 1}` | `production-canly-homepage-docdb-1` |

### IAM Role

**リソース**: `aws_iam_role`

| 用途 | 形式 | 例 |
|------|------|-----|
| **ECS Task** | `{environment}-{servicename}-{function}-ecs-task` | `production-canly-admin-api-ecs-task` |
| **ECS Task Execution** | `{environment}-{servicename}-{function}-ecs-task-execution` | `production-canly-admin-api-ecs-task-execution` |
| **Lambda** | `{environment}-{servicename}-{function}-lambda` | `production-benefits-data-lambda` |
| **GitHub Actions** | `{environment}-{servicename}-{function}-github-actions` | `production-canly-admin-github-actions` |
| **Step Functions** | `{environment}-{servicename}-{function}-sfn` | `production-canly-admin-batch-sfn` |
| **Datadog連携** | `{environment}-{servicename}-{function}-integration-role` | `production-canly-admin-integration-role` |
| **クライアント環境** | `{environment}-{servicename}-{customer}-{用途}` | `production-canlyhp-cms-hotland-ecs-task-execution` |

### Lambda

**リソース**: `aws_lambda_function`

| パターン | 形式 | 例 |
|----------|------|-----|
| **通常関数** | `{environment}-{servicename}-{function}-{処理名}` | `prod-benefits-data-api-s3-bucket-copy-handler` |

### WAF

**リソース**: `aws_wafv2_web_acl`

| パターン | 形式 | 例 |
|----------|------|-----|
| **CloudFront用** | `{environment}-{servicename}-cloudfront-waf` | `production-canly-admin-cloudfront-waf` |
| **ALB用** | `{environment}-{servicename}-alb-waf` | `production-canly-admin-alb-waf` |

### CloudWatch Logs

**リソース**: `aws_cloudwatch_log_group`

| 用途 | 形式 | 備考 |
|------|------|------|
| **Lambda** | `/aws/lambda/{environment}-{servicename}-{function}` | - |
| **RDS** | `/aws/rds/cluster/{cluster_identifier}/audit` | クラスタ識別子に連動 |
| **Amplify** | `/aws/amplify/${aws_amplify_app.this.id}` | AWSの慣例に従う |
| **Batch** | `/aws/batch/{environment}-{servicename}-{function}` | - |

### AWS Batch

| リソース | 形式 | 例 |
|----------|------|-----|
| **Compute Environment** | `{environment}-{servicename}-{function}` | `production-canly-admin-batch` |
| **Job Queue** | `{environment}-{servicename}-{function}-{priority}` | `production-canly-admin-batch-high-priority` |
| **Job Definition** | `{environment}-{servicename}-{function}-job` | `production-canly-admin-batch-job` |
| **Fair Share Policy** | `{environment}-{servicename}-{function}-fair-share-policy` | `production-canly-admin-batch-fair-share-policy` |

### ElastiCache（Valkey）

**注意**: 新規作成の場合はValkey利用を想定

| リソース | 形式 | 例 |
|----------|------|-----|
| **Replication Group** | `{environment}-{servicename}-valkey` | `production-canly-admin-valkey` |
| **サブネットグループ** | `{environment}-{servicename}-valkey-subnet-group` | `production-canly-admin-valkey-subnet-group` |
| **パラメータグループ** | `{environment}-{servicename}-valkey{engine_version}` | `production-canly-admin-valkey8` |

### その他サービス

| リソース | 形式 |
|----------|------|
| **EventBridge Scheduler** | `{environment}-{servicename}-{function}-eventbridge-scheduler` |
| **Secrets Manager** | `{environment}-{servicename}-{function}/.env` |
| **SNS Topic** | `{environment}-{servicename}-{function}-topic` |

---

## 命名時の注意事項

### 1. 環境の表記

- **原則**: locals.tfで定義した `environment`（production, stagingなど）を使用
- **文字数制限がある場合**: `env`（prod, stgなど）を使用
- **`env` でも文字数制限に引っかかる場合**:
  - 環境ごとにAWSアカウントが分離されている場合: `{servicename}-{function}` をprefixとして利用

### 2. 一貫性の維持

同一コンポーネント内では、ALB・TG・ECS・SG で同じ `environment`/`env`・`servicename`・`function`（必要なら `customer`）の組み合わせを使う。

### 3. 接尾辞の使用

用途はハイフン区切りで末尾に付ける: `lb`, `ecs-service`, `rds`, `mysql`, `lambda`, `alb-log` 等。

### 4. Terraformリソース名（ラベル）

- コード上の `resource "aws_xxx" "yyy"` の `yyy` は **snake_case**（既存規約どおり）
- 本規約は **AWSコンソール上などのリソース名（name 属性）** の規則

### 5. 必須タグとの整合性

- 命名規則に加え、`servicename`, `environment`, `customer`, `terraform` のタグは policy で必須
- リソース名とタグの `environment`/`servicename`/`function` は一致させることを推奨

---

## 例外処理

### 文字数制限がある場合

1. **環境名の短縮**: `environment` → `env`
2. **substr関数の使用**: 例: `substr(local.ecs_service_name, 0, 32)`
3. **サービス名の短縮**: 必要に応じてサービス名を短縮
4. **環境接頭辞の省略**: 環境ごとにAWSアカウントが分離されている場合

### 対応例

```hcl
# Target Group（32文字制限）
resource "aws_lb_target_group" "this" {
  name = substr("${local.environment}-${local.servicename}-${local.function}", 0, 32)
  # 結果: production-canly-admin-api-lon → production-canly-admin-api-lo
}

# 環境接頭辞省略（アカウント分離時）
resource "aws_lb" "this" {
  name = "${local.servicename}-${local.function}"
  # 結果: canly-admin-api
}
```

---

## 関連ドキュメント

### プロジェクト内規約

- [Terraformコーディング規約](./terraform-coding-standards.md) - Terraformコードの記述ルール
- [ドキュメント記述規約](./documentation-standards.md) - ドキュメント作成時の規約

### テンプレート

- [クラウドサービスガイド用テンプレート](../../templates/playbooks/cloud-service-playbook.md)
- [Terraformモジュールコメント](../../templates/parts/module-comment-format.md)

### 実装ガイド

- [AWS IAM実装ガイド](../../playbook/aws/fundamentals/iam.md)
- [AWS セキュリティグループ実装ガイド](../../playbook/aws/fundamentals/security-group.md)

---

*最終更新: 2026-03-04*
