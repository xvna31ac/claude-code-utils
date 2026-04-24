| name | description |
|------|-------------|
| cloud-infrastructure-security | クラウドプラットフォームへのデプロイ・インフラ設定・IAMポリシー管理・ログ/監視のセットアップ・CI/CDパイプライン構築時に使用するスキル。ベストプラクティスに準拠したクラウドセキュリティチェックリストを提供する。 |

# クラウド＆インフラセキュリティスキル

クラウドインフラ・CI/CDパイプライン・デプロイ設定がセキュリティベストプラクティスと業界標準に従っていることを確保するスキル。

## 起動条件

- クラウドプラットフォームへのアプリケーションデプロイ（AWS・Vercel・Railway・Cloudflare）
- IAMロールと権限の設定
- CI/CDパイプラインのセットアップ
- Infrastructure as Codeの実装（Terraform・CloudFormation）
- ログ・モニタリングの設定
- クラウド環境でのシークレット管理
- CDNとエッジセキュリティの設定
- 災害復旧・バックアップ戦略の実装

## クラウドセキュリティチェックリスト

### 1. IAM・アクセス制御

#### 最小権限の原則

```yaml
# ✅ OK: 必要最小限の権限
iam_role:
  permissions:
    - s3:GetObject  # 読み取りのみ
    - s3:ListBucket
  resources:
    - arn:aws:s3:::my-bucket/*  # 特定のバケットのみ

# ❌ NG: 過剰に広い権限
iam_role:
  permissions:
    - s3:*  # S3のすべての操作
  resources:
    - "*"  # すべてのリソース
```

#### 多要素認証（MFA）

```bash
# ルート/管理者アカウントには必ずMFAを有効化する
aws iam enable-mfa-device \
  --user-name admin \
  --serial-number arn:aws:iam::123456789:mfa/admin \
  --authentication-code1 123456 \
  --authentication-code2 789012
```

#### 確認項目

- [ ] 本番環境でルートアカウントを使用していない
- [ ] すべての特権アカウントでMFAを有効化
- [ ] サービスアカウントは長期クレデンシャルではなくロールを使用
- [ ] IAMポリシーが最小権限に従っている
- [ ] 定期的なアクセスレビューを実施
- [ ] 未使用のクレデンシャルをローテーションまたは削除

### 2. シークレット管理

#### クラウドシークレットマネージャー

```typescript
// ✅ OK: クラウドシークレットマネージャーを使用する
import { SecretsManager } from '@aws-sdk/client-secrets-manager';

const client = new SecretsManager({ region: 'us-east-1' });
const secret = await client.getSecretValue({ SecretId: 'prod/api-key' });
const apiKey = JSON.parse(secret.SecretString).key;

// ❌ NG: 環境変数だけでは不十分（ローテーションなし・監査なし）
const apiKey = process.env.API_KEY;
```

#### シークレットのローテーション

```bash
# データベース認証情報の自動ローテーションを設定する
aws secretsmanager rotate-secret \
  --secret-id prod/db-password \
  --rotation-lambda-arn arn:aws:lambda:region:account:function:rotate \
  --rotation-rules AutomaticallyAfterDays=30
```

#### 確認項目

- [ ] すべてのシークレットをクラウドシークレットマネージャーで管理（AWS Secrets Manager・Vercel Secrets）
- [ ] データベース認証情報の自動ローテーションを有効化
- [ ] APIキーを少なくとも四半期ごとにローテーション
- [ ] コード・ログ・エラーメッセージにシークレットを含めない
- [ ] シークレットアクセスの監査ログを有効化

### 3. ネットワークセキュリティ

#### VPCとファイアウォールの設定

```terraform
# ✅ OK: 制限されたセキュリティグループ
resource "aws_security_group" "app" {
  name = "app-sg"
  
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/16"]  # 内部VPCのみ
  }
  
  egress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # HTTPSアウトバウンドのみ
  }
}

# ❌ NG: インターネットに開放されている
resource "aws_security_group" "bad" {
  ingress {
    from_port   = 0
    to_port     = 65535
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # 全ポート・全IPに開放
  }
}
```

#### 確認項目

- [ ] データベースをパブリックアクセス不可に設定
- [ ] SSH/RDPポートをVPN/踏み台サーバーのみに制限
- [ ] セキュリティグループが最小権限に従っている
- [ ] ネットワークACLを設定済み
- [ ] VPCフローログを有効化

### 4. ログ・モニタリング

#### CloudWatch/ログ設定

```typescript
// ✅ OK: 包括的なログ記録
import { CloudWatchLogsClient, CreateLogStreamCommand } from '@aws-sdk/client-cloudwatch-logs';

const logSecurityEvent = async (event: SecurityEvent) => {
  await cloudwatch.putLogEvents({
    logGroupName: '/aws/security/events',
    logStreamName: 'authentication',
    logEvents: [{
      timestamp: Date.now(),
      message: JSON.stringify({
        type: event.type,
        userId: event.userId,
        ip: event.ip,
        result: event.result,
        // 機密データは絶対にログに記録しない
      })
    }]
  });
};
```

#### 確認項目

- [ ] すべてのサービスでCloudWatch/ログを有効化
- [ ] 認証失敗をログに記録
- [ ] 管理者操作を監査
- [ ] ログ保持期間を設定（コンプライアンス要件に応じて90日以上）
- [ ] 不審なアクティビティに対するアラートを設定
- [ ] ログを一元管理・改ざん防止で保管

### 5. CI/CDパイプラインセキュリティ

#### セキュアなパイプライン設定

```yaml
# ✅ OK: セキュアなGitHub Actionsワークフロー
name: デプロイ

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read  # 最小限の権限
      
    steps:
      - uses: actions/checkout@v4
      
      # シークレットスキャン
      - name: シークレットスキャン
        uses: trufflesecurity/trufflehog@main
        
      # 依存関係の監査
      - name: 依存関係の監査
        run: npm audit --audit-level=high
        
      # 長期トークンではなくOIDCを使用する
      - name: AWSクレデンシャルの設定
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/GitHubActionsRole
          aws-region: us-east-1
```

#### サプライチェーンセキュリティ

```json
// package.json - ロックファイルと整合性チェックを使用する
{
  "scripts": {
    "install": "npm ci",  // 再現性のあるビルドのために ci を使用
    "audit": "npm audit --audit-level=moderate",
    "check": "npm outdated"
  }
}
```

#### 確認項目

- [ ] 長期クレデンシャルではなくOIDCを使用
- [ ] パイプラインにシークレットスキャンを組み込み
- [ ] 依存関係の脆弱性スキャンを実施
- [ ] コンテナイメージのスキャンを実施（該当する場合）
- [ ] ブランチ保護ルールを適用
- [ ] マージ前にコードレビューを必須化
- [ ] 署名付きコミットを強制

### 6. Cloudflare・CDNセキュリティ

#### Cloudflareセキュリティ設定

```typescript
// ✅ OK: セキュリティヘッダーを付与するCloudflare Workers
export default {
  async fetch(request: Request): Promise<Response> {
    const response = await fetch(request);
    
    // セキュリティヘッダーを追加する
    const headers = new Headers(response.headers);
    headers.set('X-Frame-Options', 'DENY');
    headers.set('X-Content-Type-Options', 'nosniff');
    headers.set('Referrer-Policy', 'strict-origin-when-cross-origin');
    headers.set('Permissions-Policy', 'geolocation=(), microphone=()');
    
    return new Response(response.body, {
      status: response.status,
      headers
    });
  }
};
```

#### WAFルール

```bash
# Cloudflare WAFのマネージドルールを有効化する
# - OWASPコアルールセット
# - Cloudflareマネージドルールセット
# - レートリミットルール
# - ボット保護
```

#### 確認項目

- [ ] OWASPルールを含むWAFを有効化
- [ ] レートリミットを設定済み
- [ ] ボット保護を有効化
- [ ] DDoS保護を有効化
- [ ] セキュリティヘッダーを設定済み
- [ ] SSL/TLSストリクトモードを有効化

### 7. バックアップ・障害復旧

#### 自動バックアップ

```terraform
# ✅ OK: RDSの自動バックアップ設定
resource "aws_db_instance" "main" {
  allocated_storage     = 20
  engine               = "postgres"
  
  backup_retention_period = 30  # 30日間保持
  backup_window          = "03:00-04:00"
  maintenance_window     = "mon:04:00-mon:05:00"
  
  enabled_cloudwatch_logs_exports = ["postgresql"]
  
  deletion_protection = true  # 誤削除を防止する
}
```

#### 確認項目

- [ ] 毎日の自動バックアップを設定済み
- [ ] バックアップ保持期間がコンプライアンス要件を満たしている
- [ ] ポイントインタイムリカバリを有効化
- [ ] 四半期ごとにバックアップのテストを実施
- [ ] 障害復旧計画を文書化済み
- [ ] RPOとRTOを定義・テスト済み

## 本番クラウドデプロイ前チェックリスト

本番クラウドデプロイの前に必ず確認する：

- [ ] **IAM**: ルートアカウント不使用・MFA有効・最小権限ポリシー
- [ ] **シークレット**: すべてのシークレットをクラウドシークレットマネージャーでローテーション設定済み
- [ ] **ネットワーク**: セキュリティグループを制限・データベースを非公開
- [ ] **ログ**: CloudWatch/ログを保持期間設定済みで有効化
- [ ] **モニタリング**: 異常検知アラートを設定済み
- [ ] **CI/CD**: OIDC認証・シークレットスキャン・依存関係監査を実施
- [ ] **CDN/WAF**: CloudflareのWAFをOWASPルールで有効化
- [ ] **暗号化**: 保存時・転送時のデータを暗号化済み
- [ ] **バックアップ**: 自動バックアップと復旧テストを実施済み
- [ ] **コンプライアンス**: GDPR/HIPAAの要件を満たしている（該当する場合）
- [ ] **ドキュメント**: インフラを文書化・ランブックを作成済み
- [ ] **インシデント対応**: セキュリティインシデント対応計画を策定済み

## よくあるクラウドセキュリティの設定ミス

### S3バケットの公開露出

```bash
# ❌ NG: 公開バケット
aws s3api put-bucket-acl --bucket my-bucket --acl public-read

# ✅ OK: プライベートバケットに特定のアクセス権を付与する
aws s3api put-bucket-acl --bucket my-bucket --acl private
aws s3api put-bucket-policy --bucket my-bucket --policy file://policy.json
```

### RDSのパブリックアクセス

```terraform
# ❌ NG
resource "aws_db_instance" "bad" {
  publicly_accessible = true  # 絶対にやってはいけない
}

# ✅ OK
resource "aws_db_instance" "good" {
  publicly_accessible = false
  vpc_security_group_ids = [aws_security_group.db.id]
}
```

## 参考リソース

- [AWS セキュリティベストプラクティス](https://aws.amazon.com/security/best-practices/)
- [CIS AWS Foundations Benchmark](https://www.cisecurity.org/benchmark/amazon_web_services)
- [Cloudflare セキュリティドキュメント](https://developers.cloudflare.com/security/)
- [OWASP クラウドセキュリティ](https://owasp.org/www-project-cloud-security/)
- [Terraform セキュリティベストプラクティス](https://www.terraform.io/docs/cloud/guides/recommended-practices/)

**クラウドの設定ミスはデータ侵害の最大の原因である。たったひとつの公開S3バケットや過剰なIAMポリシーがインフラ全体を危険にさらす。常に最小権限の原則と多層防御を徹底すること。**
