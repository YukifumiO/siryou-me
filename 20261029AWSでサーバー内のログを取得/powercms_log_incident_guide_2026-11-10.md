# PowerCMS ログ保管場所一覧・問題発生時の対応ガイド（本番 / ステージング）

- **作成日**：2026年11月10日
- **AWSアカウントID**：`621675346662`
- **AWSリージョン**：`ap-northeast-1`（東京）※CloudFront用WAFのみ `us-east-1`（Global）

---

## 1. 【全体まとめ】どこに・何のログが保存されているか一覧

全8種類のログ（前段の **CloudFront / AWS WAF / ALB の3種** ＋ **EC2・RDS内の5種**）が以下のS3バケットおよびCloudWatch Logsに保存されています。

### 1.1 S3バケット一覧（全体像）

| S3バケット名 | 保存されているログの種類 | 対象環境 |
|---|---|---|
| **`aws-cf-logs-mchem-infratec-s3`** | **① CloudFront アクセスログ** | 本番（PROD）・ステージング（STG） |
| **`aws-waf-logs-mchem-infratec-s3`** | **② AWS WAF ログ** | 本番（PROD）・ステージング（STG） |
| **`aws-alb-logs-mchem-infratec-s3`** | **③ ALB（ロードバランサー）アクセスログ** | 本番（PROD）・ステージング（STG） |
| **`powercms-log-archive-logs-mchem-infratec-s3`** | **④ Apacheアクセスログ**<br>**⑤ OSシステムログ（sudo / ログイン）**<br>**⑥ OS監査ログ（auditd：ファイル改ざん検知）**<br>**⑦ PowerCMS 操作ログ（管理画面操作）**<br>**⑧ PowerCMS 再構築ログ（公開ファイル出力）** | 本番（PROD）・ステージング（STG） |

---

### 1.2 本番環境（PROD）のログ保管場所一覧

- **対象リソース**：
  - CloudFront：`mchem-infratec-cf-prod`（ディストリビューションID：`E18B2A7FLP0H7Q` / ドメイン：`d3gy5lazm05cpf.cloudfront.net`）
  - AWS WAF：`mchem-infratec-waf-prod`（ID：`65a2e50e-4399-4083-84f3-5cc9b9598030`）
  - ALB：`mchem-infratec-alb-prod`
  - EC2：`mtbs-ec2-prod`（`172.21.3.236` / インスタンスID：`i-0bd1b63ca374f1f78`）

| ログ種別 | 何のためのログか（記録内容） | ① S3 保存場所（バケット名とパス） | ② CloudWatch Logs ロググループ名<br>（ストリーム：`i-0bd1b63ca374f1f78`） | ③ EC2サーバー内の保存場所 |
|---|---|---|---|---|
| **① CloudFront** | **CDNエッジへの全アクセス記録**<br>（最前線の接続元IP、URL、キャッシュヒット、ステータス） | バケット：`aws-cf-logs-mchem-infratec-s3`<br>パス：`AWSLogs/621675346662/CloudFront/E18B2A7FLP0H7Q/{yyyy}/{MM}/{dd}/` | －（S3へ直接配信） | － |
| **② AWS WAF** | **WAFによる攻撃検知・ブロック記録**<br>（SQLi/XSS等の検知ルール、Allow/Block判定、リクエストヘッダ） | バケット：`aws-waf-logs-mchem-infratec-s3`<br>（`mchem-infratec-waf-prod` のログ） | －（S3へ直接配信） | － |
| **③ ALB** | **ロードバランサー通過時の通信記録**<br>（CloudFrontまたは外部からEC2へ転送された通信・応答時間・ステータス） | バケット：`aws-alb-logs-mchem-infratec-s3`<br>パス：`AWSLogs/621675346662/elasticloadbalancing/ap-northeast-1/prod/year={yyyy}/month={MM}/day={dd}/` | －（S3へ直接配信） | － |
| **④ Apacheアクセス** | **Webサーバー到着アクセス記録**<br>（EC2内のApacheが受け取ったURL、メソッド、ステータス200/404等） | バケット：`powercms-log-archive-logs-mchem-infratec-s3`<br>パス：`access-log/prod/` | `/powercms/mchem-infratec-rds-prod/access` | `/var/log/httpd/mchem-infratec_access.log` |
| **⑤ OSシステム** | **サーバーログイン・sudo実行・OS動作記録**<br>（SSH/SSMログイン、sudo実行コマンド、サービス起動停止） | バケット：`powercms-log-archive-logs-mchem-infratec-s3`<br>パス：`system-log/prod/` | `/powercms/mchem-infratec-rds-prod/system` | `journald`<br>（`sudo journalctl`） |
| **⑥ OS監査 (audit)** | **ファイル改ざん・作成・削除の証拠記録**<br>（`/var/www/html`、`/etc`、`/usr/local/bin`・`sbin` の変更と実行ユーザー） | バケット：`powercms-log-archive-logs-mchem-infratec-s3`<br>パス：`audit-log/prod/` | `/powercms/mchem-infratec-rds-prod/audit` | `/var/log/audit/audit.log` |
| **⑦ CMS操作** | **PowerCMS管理画面の操作記録**<br>（CMSログイン、記事・設定の変更、操作者ID、接続元IP） | バケット：`powercms-log-archive-logs-mchem-infratec-s3`<br>パス：`cms-activity/prod/` | `/powercms/mchem-infratec-rds-prod/cms-activity` | `/var/log/powercms/prod_mchem_infratec_activity.log`<br>（元DB：`prod_mchem_infratec.mt_log`） |
| **⑧ CMS再構築** | **PowerCMSによる公開ファイル出力記録**<br>（CMSがいつ・どのHTML/CSS等を再構築して書き出したか） | バケット：`powercms-log-archive-logs-mchem-infratec-s3`<br>パス：`cms-rebuild/prod/` | `/powercms/mchem-infratec-rds-prod/cms-rebuild` | `/var/log/powercms/prod_mchem_infratec_rebuild.log`<br>（元DB：`prod_mchem_infratec.mt_rebuildlog`） |

---

### 1.3 ステージング環境（STG）のログ保管場所一覧

- **対象リソース**：
  - CloudFront：`mchem-infratec-cf-stg`（ディストリビューションID：`E3IBSKX53053KL` / ドメイン：`d2841ofbbfw50e.cloudfront.net`）
  - AWS WAF：`mchem-infratec-waf-stg`
  - ALB：`mchem-infratec-alb-stg`
  - EC2：`mtbs-ec2`（`172.21.3.18` / インスタンスID：`i-01dccb8888791b2dd`）

| ログ種別 | 何のためのログか（記録内容） | ① S3 保存場所（バケット名とパス） | ② CloudWatch Logs ロググループ名<br>（ストリーム：`i-01dccb8888791b2dd`） | ③ EC2サーバー内の保存場所 |
|---|---|---|---|---|
| **① CloudFront** | **CDNエッジへの全アクセス記録**<br>（最前線の接続元IP、URL、キャッシュヒット、ステータス） | バケット：`aws-cf-logs-mchem-infratec-s3`<br>パス：`AWSLogs/621675346662/CloudFront/E3IBSKX53053KL/{yyyy}/{MM}/{dd}/` | －（S3へ直接配信） | － |
| **② AWS WAF** | **WAFによる攻撃検知・ブロック記録**<br>（SQLi/XSS等の検知ルール、Allow/Block判定、リクエストヘッダ） | バケット：`aws-waf-logs-mchem-infratec-s3`<br>（`mchem-infratec-waf-stg` のログ） | －（S3へ直接配信） | － |
| **③ ALB** | **ロードバランサー通過時の通信記録**<br>（CloudFrontまたは外部からEC2へ転送された通信・応答時間・ステータス） | バケット：`aws-alb-logs-mchem-infratec-s3`<br>パス：`AWSLogs/621675346662/elasticloadbalancing/ap-northeast-1/stg/year={yyyy}/month={MM}/day={dd}/` | －（S3へ直接配信） | － |
| **④ Apacheアクセス** | **Webサーバー到着アクセス記録**<br>（EC2内のApacheが受け取ったURL、メソッド、ステータス200/404等） | バケット：`powercms-log-archive-logs-mchem-infratec-s3`<br>パス：`access-log/stg/` | `/powercms/mchem-infratec-rds-stg/access` | `/var/log/httpd/mchem-infratec_access.log` |
| **⑤ OSシステム** | **サーバーログイン・sudo実行・OS動作記録**<br>（SSH/SSMログイン、sudo実行コマンド、サービス起動停止） | バケット：`powercms-log-archive-logs-mchem-infratec-s3`<br>パス：`system-log/stg/` | `/powercms/mchem-infratec-rds-stg/system` | `journald`<br>（`sudo journalctl`） |
| **⑥ OS監査 (audit)** | **ファイル改ざん・作成・削除の証拠記録**<br>（`/var/www/html`、`/etc`、`/usr/local/bin`・`sbin` の変更と実行ユーザー） | バケット：`powercms-log-archive-logs-mchem-infratec-s3`<br>パス：`audit-log/stg/` | `/powercms/mchem-infratec-rds-stg/audit` | `/var/log/audit/audit.log` |
| **⑦ CMS操作** | **PowerCMS管理画面の操作記録**<br>（CMSログイン、記事・設定の変更、操作者ID、接続元IP） | バケット：`powercms-log-archive-logs-mchem-infratec-s3`<br>パス：`cms-activity/stg/` | `/powercms/mchem-infratec-rds-stg/cms-activity` | `/var/log/powercms/stg_mchem_infratec_activity.log`<br>（元DB：`stg_mchem_infratec.mt_log`） |
| **⑧ CMS再構築** | **PowerCMSによる公開ファイル出力記録**<br>（CMSがいつ・どのHTML/CSS等を再構築して書き出したか） | バケット：`powercms-log-archive-logs-mchem-infratec-s3`<br>パス：`cms-rebuild/stg/` | `/powercms/mchem-infratec-rds-stg/cms-rebuild` | `/var/log/powercms/stg_mchem_infratec_rebuild.log`<br>（元DB：`stg_mchem_infratec.mt_rebuildlog`） |

---

## 2. 問題が起きたとき、どのログをどう見るか（症状別・初動対応表）

外部からの通信は **`①CloudFront` ➔ `②WAF` ➔ `③ALB` ➔ `④Apache(EC2)` ➔ `⑦⑧PowerCMS / ⑤⑥OS`** の順に流れます。

| 発生した問題・疑い | 最初に見るログ | 次に突き合わせるログ | 確認のポイント（調査の流れ） |
|---|---|---|---|
| **Webサイトの表示が改ざんされた**<br>（身に覚えのないページ・不審なファイルがある） | **⑧ CMS再構築**<br>**⑥ OS監査 (audit)** | ⑦ CMS操作<br>④ Apacheアクセス<br>①②③ CF/WAF/ALB | 1. **⑧CMS再構築**に記録があれば、CMSの正規機能によるファイル出力。<br>2. ⑧に記録がないのに**⑥OS監査(audit)**にファイル変更があれば、CMS外から直接書き換えられた改ざん。<br>3. ⑥から「いつ・どのユーザー/プロセスが書き換えたか」を特定し、同時間帯の①〜④の外部アクセスや⑤のSSHログインを辿る。 |
| **管理画面に不正ログイン・不正操作された疑い** | **⑦ CMS操作**<br>**④ Apacheアクセス** | ① CloudFront<br>② AWS WAF<br>③ ALB | 1. **⑦CMS操作**で、不審な時間帯のログイン・記事変更、操作者ID（`log_author_id`）、接続元IP（`log_ip`）を確認。<br>2. **④Apache**・**③ALB**・**①CloudFront**で、そのIPからのリクエスト経路と前後の不審な通信を確認。 |
| **サーバー内に不正侵入された・設定を変えられた疑い** | **⑤ OSシステム**<br>**⑥ OS監査 (audit)** | ④ Apacheアクセス<br>③ ALB | 1. **⑤OSシステム**で、誰がいつログイン（`session opened`）し、`sudo` で何を実行したかを確認。<br>2. **⑥OS監査(audit)**で、sudoを使わない操作も含め `/etc` や `/usr/local/sbin`、`/var/www/html` が実際に変更（`success=yes`）されたかを確認。 |
| **外部からサイバー攻撃・不審なスキャンを受けた**<br>（WAF検知、`.env` / `wp-config.php` 探査、DDoS等） | **② AWS WAF**<br>**① CloudFront** | ③ ALB<br>④ Apacheアクセス | 1. **②AWS WAF**で、攻撃リクエストがブロック（Block）されたか通過（Allow）したかを確認。<br>2. 通過していた場合は **①CloudFront ➔ ③ALB ➔ ④Apache** の順に辿り、EC2まで到達したか・HTTPステータスが何番（404か200か）だったかを確認。 |

> **【重要：問題発覚時の証拠保全】**
> 問題が発覚したら、保持期限でログが消える前に **対象期間のS3ファイル（上記4バケットの該当箇所）** と **CloudWatch Logs の抽出結果** を別フォルダへコピー（ダウンロード）し、ファイルの **SHA-256ハッシュ値・取得日時・取得者** を記録して原本として保全してください。

---

## 3. 各ログの詳細（何が記録されていて、どう読むか）

### ① CloudFront アクセスログ（`aws-cf-logs-mchem-infratec-s3`）
- **目的**：ユーザーや攻撃者が最初にアクセスするCDN（CloudFront）の全リクエストをJSON形式で記録する。
- **何がわかるか**：実際の接続元クライアントIP（`c-ip`）、リクエスト日時、URIパス・クエリ、HTTPステータスコード、キャッシュヒット結果（CloudFrontで返したか、裏のALB/EC2へ取りに行ったか）、User-Agent。

### ② AWS WAF ログ（`aws-waf-logs-mchem-infratec-s3`）
- **目的**：Webアプリケーションファイアウォール（WAF）による検査結果と攻撃遮断の記録。
- **何がわかるか**：アクセス元IP・国、リクエストヘッダ、一致したWAFルール名（`terminatingRuleId`）、最終アクション（`ALLOW`＝通過 / `BLOCK`＝遮断）。

### ③ ALB アクセスログ（`aws-alb-logs-mchem-infratec-s3`）
- **目的**：ロードバランサー（ALB）を通過してEC2へ向かった通信の記録（JSON形式、Hive互換日付パーティション）。
- **何がわかるか**：ターゲットEC2のIPとポート、ALBの応答コード（`elb_status_code`）とバックエンドEC2の応答コード（`target_status_code`）、処理時間、リクエストURL、`X-Forwarded-For`（元の接続元IP）。

### ④ Apacheアクセスログ（`access`）
- **目的**：EC2内のWebサーバー（Apache）に実際に到達したリクエストの記録。
- **何がわかるか**：リクエスト日時、メソッドとパス（例：`GET /index.html`、`POST /mt/mt.cgi`）、HTTPステータスコード（`200`, `403`, `404`, `500`）、レスポンスバイト数、Referer、User-Agent。

### ⑤ OSシステムログ（`system`）
- **目的**：OSへのログイン履歴、`sudo` コマンド実行履歴、システムサービスの異常を把握する。
- **何がわかるか**：`sshd` / `systemd-logind` のセッション開始・終了、`sudo` の実行記録（実行ユーザー、実行ディレクトリ、指定コマンド）。
- **注意点**：`sudo` ログは「コマンドを実行しようとした記録」であり、実際にファイル変更が成功したかどうかは **⑥ OS監査ログ（audit）** で裏付ける。

### ⑥ OS監査ログ（`audit`）
- **目的**：OS上の重要ファイルに対する作成・変更・権限変更・削除（`perm=wa`）をカーネルレベルで確実に捉える。
- **監視キー（フィルタリング用）**：
  - Web領域（`/var/www/html`）：本番は `prod_powercms_change` ／ ステージングは `stg_powercms_change`
  - OS設定（`/etc`）：本番は `prod_config_change` ／ ステージングは `stg_config_change`
  - スクリプト（`/usr/local/bin`, `/usr/local/sbin`）：本番は `prod_script_change` ／ ステージングは `stg_script_change`
- **主な項目と見方**：
  - 同じ操作に対して `msg=audit(秒.ミリ秒:シリアル番号)` が共通する複数の行（`SYSCALL`, `PATH`, `CWD`, `PROCTITLE`）が出力される。
  - `AUID`＝最初にログインしたユーザー、`UID/EUID`＝実行時の権限、`success=yes`＝操作成功、`PATH` の `name="..."`＝操作されたファイルパス、`nametype=CREATE/DELETE/NORMAL`＝作成・削除・更新の区別。

### ⑦ PowerCMS 操作ログ（`cms-activity`）
- **目的**：PowerCMS管理画面内の操作履歴（RDS `mt_log` テーブル）を1行1JSON形式で記録する。
- **何がわかるか**：`log_id`（連番ID）、`log_created_on`（発生日時）、`log_author_id`（操作したCMSユーザーID）、`log_ip`（操作元IP）、`log_category` / `log_class`（ログイン・記事保存等）、`log_message`（具体的な操作内容）。

### ⑧ PowerCMS 再構築ログ（`cms-rebuild`）
- **目的**：PowerCMSが再構築によってWeb公開ファイルを書き出した履歴（RDS `mt_rebuildlog` テーブル）を1行1JSON形式で記録する。
- **何がわかるか**：`rebuildlog_id`（連番ID）、`rebuildlog_created_on`（再構築日時）、`rebuildlog_created_by`（実行したCMSユーザーID）、`rebuildlog_file_path`（出力されたファイルのフルパス、例：`/var/www/html/sitemap.html`）。

---

## 4. すぐ使える検索クエリ集（CloudWatch Logs Insights）

EC2・RDS系の5種類のログ（④〜⑧）について、直近30日以内の調査はAWSコンソールの **CloudWatch ＞ ログ ＞ Logs Insights** で対象のロググループを選び、以下のクエリを貼り付けてすぐ検索できます。
（※①CloudFront、②WAF、③ALBのログは各S3バケットから対象日付フォルダのJSONファイルをダウンロードまたはAthena等で確認します）

### 4.1 【⑥ audit】Webファイルや設定ファイルが変更・削除された記録を探す
- **対象ロググループ**：`/powercms/mchem-infratec-rds-prod/audit`（検証は `-stg/audit`）
```sql
fields @timestamp, @message
| filter @message like /prod_powercms_change|stg_powercms_change|prod_config_change|stg_config_change|prod_script_change|stg_script_change/
| sort @timestamp desc
| limit 200
```
※見つかった行の `msg=audit(xxx.xxx:161364)` の番号（例：`161364`）から詳細一式（実行コマンド・パス・成否）を引く場合：
```sql
fields @timestamp, @message
| filter @message like /:161364\)/
| sort @timestamp asc
```

### 4.2 【⑦ cms-activity】CMS管理画面の操作履歴を新しい順に見る
- **対象ロググループ**：`/powercms/mchem-infratec-rds-prod/cms-activity`（検証は `-stg/cms-activity`）
```sql
fields @timestamp, @message
| sort @timestamp desc
| limit 100
```

### 4.3 【⑧ cms-rebuild】CMSが再構築で出力したファイル履歴を見る
- **対象ロググループ**：`/powercms/mchem-infratec-rds-prod/cms-rebuild`（検証は `-stg/cms-rebuild`）
```sql
fields @timestamp, @message
| sort @timestamp desc
| limit 100
```

### 4.4 【④ access】不審なURLアクセスや管理画面へのPOST通信を探す
- **対象ロググループ**：`/powercms/mchem-infratec-rds-prod/access`（検証は `-stg/access`）
```sql
fields @timestamp, @message
| filter @message like /\.env|wp-config|POST|mt\.cgi/
| sort @timestamp desc
| limit 200
```

### 4.5 【⑤ system】SSH/SSMログインや `sudo` コマンド実行履歴を探す
- **対象ロググループ**：`/powercms/mchem-infratec-rds-prod/system`（検証は `-stg/system`）
```sql
fields @timestamp, @message
| filter @message like /sudo|session opened|session closed|Accepted/
| sort @timestamp desc
| limit 200
```

---

## 5. ログがCloudWatch/S3に来ていないときの点検コマンド（EC2内）

万が一EC2からのログ（④〜⑧）が止まっている場合は、対象EC2（本番：`ssh mtbs-ec2-prod` ／ 検証：`ssh mtbs-ec2`）で以下を実行して状態を確認します。

```bash
# 1. CloudWatch Agent（ログ転送サービス）が稼働しているか
sudo systemctl status amazon-cloudwatch-agent --no-pager

# 2. CMSログ取得タイマー（毎分実行）と直近の実行結果が成功しているか
# （本番は prod_... / 検証は stg_...）
sudo systemctl status prod_mchem_infratec_log_export.timer --no-pager
sudo journalctl -u prod_mchem_infratec_log_export.service -n 20 --no-pager

# 3. OS監査（auditd）が有効で、取りこぼし（lost）が 0 かどうか
sudo auditctl -s
sudo auditctl -l
```
