# AIによるAWS環境構築手順

## 1. 目的

本書は、[AWS構成・構築手順書](AWS構成・構築手順.md)に記載されたAWS環境を、AIとTerraformを利用して安全かつ再現可能な形で構築するための進め方をまとめたものです。

AIにAWSコンソールを直接操作させて一括構築するのではなく、AIにTerraformコードの生成・検査を担当させ、人が実行内容を確認して段階的に適用します。

```text
AWS構成・構築手順書
  ↓ AIで設計値を整理
Terraformコード生成
  ↓ validate / lint / セキュリティ検査
terraform plan
  ↓ 人が差分を確認・承認
STG構築・試験
  ↓
PROD構築・試験
```

AWS CDKやCloudFormationでも実現できますが、本構成ではVPC、EC2、RDS、EFS、ALB、CloudFront、WAFなどを一括して管理しやすいTerraformの利用を推奨します。

## 2. AIへ依頼する前に確定する項目

元の手順書には未確定事項があるため、以下を確定してからコード生成を開始します。

| 項目 | 決定内容 | 決定例 |
|---|---|---|
| AWSアカウント | STGとPRODを同一アカウントにする | 1つのAWSアカウント内にSTGとPRODを構築し、リソース名、タグ、セキュリティグループ、Terraform Stateを環境別に分離する |
| リージョン | 使用するAWSリージョン | 東京リージョン`ap-northeast-1`を使用する。CloudFront用ACM証明書のみ`us-east-1`に作成する |
| AZ | 使用するAZと各リソースの配置先 | `ap-northeast-1a`と`ap-northeast-1c`の2AZを使用し、ALB、EFSマウントターゲット、RDSサブネットを2AZに配置する |
| PROD EC2 | 台数、インスタンスタイプ、AZ分散、Auto Scalingの有無 | `t3.small`を2台使用し、`1a`と`1c`へ1台ずつ配置する。Auto Scaling Groupは最小2台、希望2台、最大4台とする |
| RDS | PROD/STGを別インスタンスにするか、共用するか、可用性構成 | PRODとSTGでRDSインスタンスを分離する。PRODはMulti-AZ、STGはSingle-AZとし、Public Accessは無効にする |
| DBエンジン | エンジン、バージョン、インスタンスクラス、文字コード | Amazon RDS for MySQL 8.4、PRODは`db.t4g.small`、STGは`db.t4g.micro`、文字コードは`utf8mb4`とする |
| ドメイン | PROD/STGのFQDNとALBオリジン用ドメイン | PRODは`www.example.jp`、STGは`stg.example.jp`、オリジン用は`origin.example.jp`と`origin-stg.example.jp`とする |
| DNS | DNS管理サービス、ホストゾーン、レコード、切り替え方法 | Route 53で`example.jp`を管理し、CloudFrontへのAliasレコードを設定する。切り替え前に既存レコードのTTLを300秒へ短縮する |
| 管理アクセス | 踏み台SSHまたはSSM Session Manager、接続権限、監査方法 | 通常はSSM Session Managerを使用し、アプリEC2へPublic IPを付与しない。踏み台EC2を使用する場合は管理元固定IPからのSSHだけを許可する |
| NAT用EC2 | インスタンスタイプ、OS、台数、冗長化、復旧方法 | Amazon Linux 2023の`t3.small`を1台使用してEIPを割り当てる。CloudWatchで監視し、障害時はTerraformで再作成してPrivateルートを付け替える |
| アプリケーション | Webサーバー、言語・ランタイム、バージョン、配備方法 | Amazon Linux 2023、Nginx 1.26、PHP 8.3を使用する。CI/CDで配備し、設定値とパスワードはSecrets ManagerまたはParameter Storeから取得する |
| EFS | PROD/STGの分離、マウント先、格納データ、暗号化、バックアップ | PROD用とSTG用に別々のEFSを作成し、`/var/www/shared`へマウントする。アップロードファイルを格納し、保存時暗号化と自動バックアップを有効にする |
| バックアップ | RDS・EFS・EC2の保持期間、取得頻度、復元試験 | PROD RDSは35日、STG RDSは7日保持する。EFSはAWS Backupで日次取得して35日保持し、四半期ごとにSTGへ復元して確認する |
| 管理元IP | 踏み台EC2へのSSHを許可する固定グローバルIPと変更手順 | 運用拠点の固定IP`203.0.113.10/32`だけを許可する。構築時は実際の固定IPへ置き換える。SSMだけを使う場合はSSH受信ルールを作成しない |

特にRDSをPROD/STGで共用するかは、構築開始前に必ず決定します。安全性を重視する場合は別インスタンスとし、共用する場合でもDBおよびDBユーザーを分離します。

> **注意：** 上表の「決定例」は、そのまま採用する確定値ではありません。費用、可用性、アプリケーション要件、既存ドメイン、実際の管理元IPを確認して置き換えてください。`example.jp`および`203.0.113.10/32`は説明用の例です。

## 3. AI作業環境の準備

### 3.1 必要なツール

ローカルPCまたはCI環境へ以下を導入します。

```bash
aws --version
terraform version
git --version
```

必要に応じて、以下の検査ツールも導入します。

- TFLint
- Trivy
- Checkov

### 3.2 AWSへの認証

長期アクセスキーをAIへ渡さず、AWS IAM Identity Center（SSO）の利用を推奨します。

```bash
aws configure sso
aws sso login --profile tsp-infra-admin
aws sts get-caller-identity --profile tsp-infra-admin
```

最初は参照・検証用の権限で`terraform plan`まで実施し、`terraform apply`時だけ構築用ロールを使用します。

次の情報は、AIへのプロンプト、Terraformコード、Gitリポジトリへ記載しません。

- AWSアクセスキーとシークレットアクセスキー
- SSH秘密鍵
- DBパスワード
- 証明書の秘密鍵
- CloudFrontからALBへ渡す秘密のヘッダー値

## 4. Terraformの構成

### 4.1 推奨ディレクトリ構成

```text
infra/
├── versions.tf
├── providers.tf
├── backend.tf
├── variables.tf
├── outputs.tf
├── environments/
│   ├── stg.tfvars
│   └── prod.tfvars
├── modules/
│   ├── network/
│   ├── security-groups/
│   ├── bastion/
│   ├── nat-instance/
│   ├── application/
│   ├── efs/
│   ├── rds/
│   ├── alb/
│   ├── acm/
│   ├── cloudfront/
│   ├── waf/
│   ├── dns/
│   └── monitoring/
└── tests/
```

### 4.2 Stateの分離

STGとPRODは変数だけでなく、Terraform Stateも分離します。

```text
tsp/stg/terraform.tfstate
tsp/prod/terraform.tfstate
```

Stateは、バージョニング、暗号化、アクセス制御を設定したS3バケットなどで管理します。Terraform Stateには機密情報が含まれる可能性があるため、一般利用者から読み取れないようにします。

## 5. AIへの最初の依頼文

以下のような依頼文を使用します。未確定の値は、実際に決定した内容へ置き換えます。

```text
AWS構成・構築手順.mdを仕様書として、TerraformでAWS環境を構築してください。

条件:
- リージョン: ap-northeast-1
- AZ: ap-northeast-1a / ap-northeast-1c
- VPC: 10.0.0.0/16
- STGとPRODは変数およびStateを分離する
- NAT Gatewayは禁止し、Amazon Linux 2023のNAT EC2を使う
- 踏み台EC2とNAT EC2は分離する
- アプリEC2とRDSはPrivate subnetに配置する
- ALBは2つのAZを使用する
- EFSマウントターゲットは使用する各AZに作成する
- CloudFront用ACM証明書はus-east-1に作成する
- ALB用証明書はap-northeast-1に作成する
- WAFはCloudFrontへ関連付ける
- 秘密情報をコード、tfvars、User Dataへ直接記載しない
- すべてのリソースにProject、Environment、ManagedByタグを付ける
- 削除保護、バックアップ、ログ、CloudWatchアラームを設計する

いきなりapplyしないでください。
最初に以下を提示してください。
1. 未確定事項
2. リソース一覧
3. Terraformのディレクトリ構成
4. IAM権限案
5. 構築フェーズ
6. 費用および単一障害点
私の承認後にコードを作成してください。
```

AIへは、最初から`terraform apply`を実行しないよう明示します。

## 6. 段階的な構築手順

### 6.1 フェーズ1：ネットワーク

以下のリソースをコード化します。

- VPC
- Public/App/DBサブネット
- Internet Gateway
- PublicおよびPrivateルートテーブル
- VPC Flow Logs
- セキュリティグループ
- 必要なVPC Endpoint

この段階ではRDSやCloudFrontを作成しません。CIDRの重複、サブネットとルートテーブルの関連付け、意図しないインターネット公開がないことを確認します。

### 6.2 フェーズ2：管理アクセスとNAT用EC2

以下を構築します。

- 踏み台EC2またはSSM Session Managerによる接続経路
- NAT用EC2
- Elastic IP
- Source/Destination Checkの無効化
- IPv4フォワーディング
- NAT変換ルール
- Private-Appルートテーブルのデフォルトルート

NAT用EC2の設定はTerraformのUser Dataまたは管理されたAMIとしてコード化します。NAT用EC2は単一障害点になるため、CloudWatchアラーム、復旧手順、再作成後のルート切り替え方法も用意します。

### 6.3 フェーズ3：データ層

以下を構築します。

- RDSサブネットグループ
- RDS
- Secrets Manager
- PROD/STG用EFS
- 使用する各AZのEFSマウントターゲット
- AWS Backup
- CloudWatch LogsおよびCloudWatch Alarm

DBパスワードはTerraformコードや`tfvars`へ平文で記載せず、Secrets Managerで管理します。

### 6.4 フェーズ4：STG環境

以下を構築します。

- STGアプリケーションEC2
- Target Group
- ALB
- ACM証明書
- CloudFront
- WAF
- Route 53レコード

STG環境の構築と総合試験が完了するまで、PROD環境は構築しません。

### 6.5 フェーズ5：PROD環境

STGで確認済みのTerraformモジュールを、PROD用の変数で展開します。環境ごとに以下を変更します。

- EC2の台数とインスタンスタイプ
- ドメイン名
- RDS構成
- EFS構成
- WAFルール
- バックアップ保持期間
- 削除保護

## 7. コード生成後の検査

### 7.1 Terraform標準検査

AIがコードを生成したら、各フェーズで以下を実行します。

```bash
terraform fmt -check -recursive
terraform init
terraform validate
terraform plan -var-file=environments/stg.tfvars -out=stg.tfplan
terraform show stg.tfplan
```

### 7.2 追加検査

可能であれば、以下も実行します。

```bash
tflint --recursive
trivy config .
checkov -d .
```

### 7.3 AIへのPlanレビュー依頼

```text
terraform planの結果をレビューしてください。

以下を重点的に確認してください。
- 意図しない削除または再作成
- 0.0.0.0/0からのSSH、RDS、EFSアクセス
- Public IPが不要なEC2へ付与されていないか
- RDSがPublicになっていないか
- 暗号化されていないEBS、EFS、RDS、ログ
- PROD/STG間の通信経路
- NAT EC2のルーティング
- IAMのワイルドカード権限
- 削除保護とバックアップ
- 月額費用が大きいリソース
問題がなければ、その根拠をリソース単位で説明してください。
```

AIの回答だけで安全と判断せず、人が`terraform plan`の内容を確認します。

## 8. 適用と承認

`terraform plan`を人が確認した後、保存したPlanを適用します。

```bash
terraform apply stg.tfplan
```

少なくとも以下の操作には、人による明示的な承認を必要とします。

- RDSの作成、削除、置換
- Route 53の切り替え
- CloudFrontの公開
- セキュリティグループのインターネット公開
- PROD環境への適用
- Terraform Stateの操作
- IAM権限の変更

CloudFormationまたはAWS CDKを使用する場合も、Change Setで追加・変更・削除を確認してから適用します。

## 9. STG環境の試験

PROD構築前に、STGで以下を確認します。

- [ ] 踏み台またはSSMからアプリEC2へ接続できる
- [ ] アプリEC2にPublic IPが付与されていない
- [ ] Private EC2からNAT用EC2経由でHTTPS通信できる
- [ ] NAT用EC2の再起動後も転送設定とNATルールが復元される
- [ ] EC2からEFSをマウントできる
- [ ] EC2からRDSへ接続できる
- [ ] ALBのヘルスチェックが正常になる
- [ ] CloudFront経由で画面を表示できる
- [ ] HTTPからHTTPSへリダイレクトされる
- [ ] WAFで対象通信をブロックできる
- [ ] ALBへ直接アクセスした場合は拒否される
- [ ] CloudWatch Logsにログが保存される
- [ ] CloudWatch Alarmが動作する
- [ ] バックアップを取得できる
- [ ] バックアップから復元できる

## 10. CloudFrontと証明書の注意点

### 10.1 ACM証明書

- CloudFrontのビューワー証明書は`us-east-1`で作成します。
- ALBの証明書は、ALBと同じリージョンで作成します。
- 東京リージョンのALBの場合は`ap-northeast-1`です。

### 10.2 ALBへの直接アクセス制限

CloudFront経由のアクセスのみを許可する場合は、以下を組み合わせて制限します。

- CloudFrontからALBへ秘密のカスタムHTTPヘッダーを送信
- ALBリスナールールで、正しいヘッダーを持つ通信だけTarget Groupへ転送
- その他の通信には固定の`403`レスポンスを返す
- CloudFrontからALB間をHTTPSにする
- 必要に応じてCloudFrontのAWSマネージドプレフィックスリストでALBのSGを制限

秘密のヘッダー値は、コードへ直接記載せず安全に管理し、定期的にローテーションします。

## 11. 手順書へ追加すべき設計項目

Terraformコードを作成する前に、元の構築手順書へ以下を追加するとAIによるコード生成の精度が上がります。

- PROD/STGのサブネット分離方法
- PROD/STGのAWSアカウント分離方針
- RDSの分離方式
- Terraform Stateの保管場所とアクセス制御
- Secrets Managerの使用方針
- SSMを使うか、SSH踏み台を残すか
- VPC Flow Logs、ALB、CloudFront、WAFログの保存先と保持期間
- NAT用EC2障害時の復旧方法
- NAT用EC2のAMI更新と脆弱性対応
- CloudFrontからALBへの直接アクセス制限
- バックアップからの復元試験
- リソースごとの削除保護
- AWS Budgetsなどによる予算アラーム

## 12. 推奨する実行順序

最終的な実行順序は以下とします。

1. 未確定事項を決定する。
2. AIに設計内容とリソース一覧を整理させる。
3. AIにネットワークのTerraformコードを作成させる。
4. 静的検査と`terraform plan`を実行する。
5. 人がPlanを承認してネットワークを適用する。
6. NAT用EC2と管理アクセスを構築・試験する。
7. RDS、EFS、バックアップを構築・試験する。
8. STGを構築して総合試験を実施する。
9. STGで確認済みのコードからPROD用Planを作成する。
10. 人がPROD用PlanとDNS切り替え計画を承認する。
11. PRODを構築する。
12. DNSを切り替え、監視しながら公開する。
13. 構築後のTerraformコード、State、試験記録、復旧手順を保管する。

## 13. 参考資料

- [AWS公式：NATインスタンスを使用してVPC外部と通信する](https://docs.aws.amazon.com/ja_jp/vpc/latest/userguide/work-with-nat-instances.html)
- [AWS公式：CloudFormation Change Set](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-changesets.html)
- [AWS公式：CloudFrontからALBへのアクセス制限](https://docs.aws.amazon.com/ja_jp/AmazonCloudFront/latest/DeveloperGuide/restrict-access-to-load-balancer.html)
- [AWS公式：CloudFrontのSSL/TLS証明書要件](https://docs.aws.amazon.com/ja_jp/AmazonCloudFront/latest/DeveloperGuide/cnames-and-https-requirements.html)

---

作成日：2026年11月20日  
関連資料：[AWS構成・構築手順書](AWS構成・構築手順.md)
