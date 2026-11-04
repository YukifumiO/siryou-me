# PowerCMS本番ログ収集・監査・証拠保全プロジェクト — 会話引き継ぎ資料

- **作成日**：2026年11月4日（JST）
- **対象**：Amazon Linux 2023 上の PowerCMS 本番環境、CloudWatch Logs／Amazon Data Firehose／S3
- **資料の用途**：別のチャットで、過去の背景説明を繰り返さずに調査・設定を再開する
- **注意**：本資料は会話中のユーザー提示の実行結果と画面を主体にまとめたもの。AWS実環境へ直接アクセスして現時点の設定を再検証したものではない。**確認済み／推定／未確認**を区別すること。

## 0. 最重要サマリー（次のチャットで先に読む）

1. **目的**：PowerCMS本番への不正侵入、ファイル改ざん、機密情報閲覧・持ち出し等が起きたとき、操作主体・時刻・対象・結果を複数ログで説明し、警察に提出し得る証拠を適切に保全する。
2. **既存パイプライン**：本番の5ロググループ（Apache access、system、audit、CMS activity、CMS rebuild）をCloudWatch Logsで収集し、各ロググループからFirehose→S3へ保存する構成。実際の全配信・欠落なしは別途照合が必要。
3. **auditdの改善は完了段階**：初期設定の`-a task,never`をコメント化して解除し、`/etc`、`/usr/local/bin`、`/usr/local/sbin`、`/var/www/html`に**変更・属性変更（`perm=wa`）の永続ルール**を追加。設定反映時 `enabled 1 / lost 0 / backlog 0` を確認。PowerCMS実ファイルでの改ざん検知までは未検証。
4. **auditdのテスト成功**：EC2ローカル `/var/tmp/audit-test` で`touch`、`chmod`、`rm`、`cat`、`less`を監査、CloudWatchにも `SYSCALL`、`PATH`、`CWD`、`PROCTITLE` が到着。NFS `/var/www/audit-test`で作成・権限変更・削除を監査。NFSのCloudWatch到着と読み取りはまだ要照合。
5. **PowerCMSのWeb本体はNFSv4.1**：`findmnt -T /var/www/html`→`/var/www` が `127.0.0.1:/` の `nfs4`。EFSの可能性はあるが、この出力だけでは確定できない。他のEC2／NFSクライアントからの変更は当該EC2のauditdだけでは網羅しない。
6. **cms-rebuild自動取得は確認できた**：systemd timerが毎分起動、`prod_mchem_infratec_export_rebuild.py`が`0/SUCCESS`、`mt_rebuildlog`のJSON行が `/var/log/powercms/prod_mchem_infratec_rebuild.log` に出力され、CloudWatchの `/powercms/mchem-infratec-rds-prod/cms-rebuild` の実イベント表示をユーザーのスクリーンショットで確認。最初「ログストリーム0件」と誤認したが、**その後の画面で実際のログを確認して解消済み**。欠落なし・S3保存完了は別検証。
7. **最も新しい未解決タスク：S3ライフサイクル**。バケット `powercms-log-archive-logs-mchem-infratec-s3` のルール名は `delete-logs-after-400days`、有効、**設定された最新バージョン有効期限は実際には「3日」**（ルール名の400とは無関係）、バケット全体対象。ユーザーは「3日以上前のファイルが消えない」と相談。要確認：対象オブジェクトの **S3 Last modified**、ライフサイクル評価時刻・非同期遅延、バージョニング、非現行バージョン、Object Lock、ルールJSON。**勝手に削除・期限変更しない。**
8. **保持期間ポリシー**：ユーザーは構築・テストが全部終わるまで短期削除設定を維持し、最後に CloudWatch約30日、S3約400日へ切り替える意向。以前「1日削除テスト設定」との発言があったが、最新S3画面では **3日** のため「現在どちらが正しいか」をサービス別に確認する。証拠が消えるリスクは伝達済み。
9. **絶対に断定しない点**：`status=0`だけでクラウド転送完了とはいえない／`openat(O_RDONLY)`だけで情報流出とはいえない／`sudo USER_CMD`だけでファイル操作成功とはいえない／auditdの監視対象以外や他EC2の操作は検出保証なし／警察での証拠能力は保全手続と状況による。

---

## 1. プロジェクトの背景と優先目的

PowerCMS（会話では6.8.4と説明された）の本番Webサーバーで、アクセス・OS操作・CMS内部操作・ファイル改変の記録をAWS側へ転送する仕組みを構築している。警察への被害申告やフォレンジック調査を見据え、**誰のアカウント／どのプロセスが、いつ、何を操作し、そのシステムコールが成功したか**を確保する。加えて、外部アクセスやCMS操作のログと照合し、原本性・整合性・受渡履歴を維持する。

- 本番EC2：Amazon Linux 2023、ホスト名 `ip-172-21-3-236`、プライベートIP `172.21.3.236`、リージョン `ap-northeast-1`。
- PowerCMS Web配置：`/var/www/html`（`/var/www` は NFSv4.1）。
- DB：RDS MySQL 8.4系、`mt_log` と `mt_rebuildlog` をログエクスポータが読取。
- 本番とステージングのDBは共通RDSで、EC2は別という背景。ステージングのログパイプライン完成は未確認。
- 記録はセンシティブになり得るため、次チャットへ**パスワード、秘密鍵、DB接続文字列、個人情報の生値を貼らない**。過去の会話でDBパスワードが開示された可能性があり、必要ならローテーション検討。 

## 2. ログ収集アーキテクチャ

```text
[CloudFront / (ALBがあればALB) / Apache Webアクセス]
             └─ Apacheアクセスログ ─┐
[EC2 journald / sudo / PAM] ────────┤
[EC2 auditd / audit.log] ───────────┤
[PowerCMS RDS mt_log] ─ Python ─ file ┤─ CloudWatch Agent ─ CloudWatch Logs
[PowerCMS RDS mt_rebuildlog] ─ Python ─ file ┘                         │
                                                 ロググループ別 Subscription Filter
                                                                         │
                                                                   Amazon Data Firehose
                                                                         │
                                                                          S3
```

### 本番CloudWatch Logsの5種類

| 用途 | ロググループ | 主な元データ | 状態 |
|---|---|---|---|
| Apacheアクセス | `/powercms/mchem-infratec-rds-prod/access` | `/var/log/httpd/mchem-infratec_access.log` | 設定・過去収集あり |
| OSシステム | `/powercms/mchem-infratec-rds-prod/system` | journald（sudo、OS動作など） | 過去CloudWatchログ実例あり |
| OS監査 | `/powercms/mchem-infratec-rds-prod/audit` | `/var/log/audit/audit.log` | syscall/PATH等の実例あり |
| PowerCMS操作 | `/powercms/mchem-infratec-rds-prod/cms-activity` | RDS `mt_log` → Python → ローカルファイル | 定期スクリプト実行確認、CloudWatch個別の最新到着は別途照合 |
| PowerCMS再構築 | `/powercms/mchem-infratec-rds-prod/cms-rebuild` | RDS `mt_rebuildlog` → Python → ローカルファイル | **毎分実行／CloudWatchイベント到着確認済み** |

各ロググループから別々のFirehose配信ストリーム経由で、同じS3バケットの用途別プレフィックスへ保存する方針：

```text
access-log/prod/
system-log/prod/
audit-log/prod/
cms-activity/prod/
cms-rebuild/prod/
```

※「配信設定が存在」することと「各イベントがS3に欠落なく保存されていること」は別。最後にS3内のオブジェクトを実際に照合する。

### `cms-rebuild` 自動化の技術詳細（2026-11-04検証）

- systemd timer：`prod_mchem_infratec_log_export.timer`、`enabled` / `active (waiting)`、**毎分実行**。
- service：`prod_mchem_infratec_log_export.service`、タイマーから起動。終了後は`inactive (dead)`で正常なoneshot相当。
- `ExecStart`で2つのPythonを起動、両方 `status=0/SUCCESS` と表示：
  - `/usr/local/sbin/prod_mchem_infratec_export_activity.py`
  - `/usr/local/sbin/prod_mchem_infratec_export_rebuild.py`
- 2026-11-04 11:44～12:13 JST のjournalctlログでは、各分 `Starting` → `Deactivated successfully` → `Finished` の繰り返し。
- rebuildスクリプトのコード断片に `SELECT JSON_OBJECT(...) ... FROM mt_rebuildlog`、`OUTPUT.open("a", ...)`、`STATE`更新、ロックが存在。
- `STATE = Path("/var/lib/powercms-log-export/prod_mchem_infratec_rebuild.lastid")`。
- 出力ファイル：`/var/log/powercms/prod_mchem_infratec_rebuild.log`、2026-11-04 12:10頃 `7416` バイト、`root:root 0600`。
- 実ファイルには `{"table":"mt_rebuildlog","source":"powercms","rebuildlog_id":46,...}`、ID47、48などのJSON行。`rebuildlog_file_path`例：`/var/www/html/sitemap.html`、`/var/www/html/css/site-all.css`。これらは**ログの内容であり不正改ざんの証拠ではない**。
- Agent設定に `file_path="/var/log/powercms/prod_mchem_infratec_rebuild.log"`、`log_group_name="/powercms/mchem-infratec-rds-prod/cms-rebuild"`、`log_stream_name="i-0bd1b63ca374f1f78"`。
- Agent起動ログ：`[logagent] piping log from ... to cloudwatchlogs`、稼働中。
- **CloudWatch Consoleの「All events」画面で `mt_rebuildlog` のJSON行が6件程度可視**。例、2026-11-04 11:57:00、11:57:05、12:10:01（複数）、12:10:05 JST。
- 未確認：DBの最大IDとstateの差分がゼロか、失敗時にstateを早く進めて欠落しないか、再実行による重複、S3へ届いているか。

## 3. auditdの問題発見→改善→検証履歴

### 3.1 初期状態と原因

初期 `sudo auditctl -l`：`-a never,task`。元ファイル `/etc/audit/rules.d/audit.rules`：

```text
-D
-a task,never
```

`augenrules`が生成する `/etc/audit/audit.rules` にも `-a task,never`。これは新しく起動したタスクのシステムコール監査を抑える設定で、sudo/PAM由来の`USER_CMD`等は記録されても、`ec2-user`によるsudoなしファイル操作は観測されなかった。`auditctl -s`は当時 `enabled 1, lost 0`。

変更：`/etc/audit/rules.d/audit.rules`の当該行を `##-a task,never` としてコメント化（`-D`は残留）。`sudo augenrules --load`で反映し、`-a never,task`がないことを確認した。初期バックアップ：

```text
/etc/audit/rules.d/audit.rules.bk20261102
/etc/audit/audit.rules.bk20261102
```

注意：`augenrules --load`の標準出力 `No rules` は、実際にはルールがロードされた例がある。**常に `sudo auditctl -l` と `sudo auditctl -s` で確かめる**。

### 3.2 ローカルテスト結果

永続テストルール `/etc/audit/rules.d/30-powercms-test.rules`：

```bash
-a always,exit -F arch=b64 -F dir=/var/tmp/audit-test -F perm=wa -k powercms_file_test
```

テスト用ディレクトリ `/var/tmp/audit-test` はec2-user所有、700。以下の結果を`sudo ausearch -k powercms_file_test -i`とCloudWatchで確認：

| 操作 | 監査イベントID | 裏付け |
|---|---|---|
| `touch powercms_integrity_check.txt` | `161364` | `openat success=yes`、`PATH nametype=CREATE`、`AUID=ec2-user` |
| `chmod 600 ...` | `161393` | `fchmodat success=yes` |
| `rm ...` | `161394` | `unlinkat success=yes`、`PATH nametype=DELETE` |
| `vim`による別ファイル編集 | `161515`～`161521`付近 | 一時ファイルや`test_vvvvvvv.txt`作成等 |

`echo >` / `echo >>` は実行されたが、提示された`ausearch -k`の検索結果では対応イベントが分かれて現れず、**データ書き込み1回ずつを保証するものではない**。通常`perm=wa`は関連するopen/renameなどを監査するので、`write()`の中身や回数までは追えない。

一時的な読み取りルールを`auditctl`で追加：

```bash
-a always,exit -F arch=b64 -F dir=/var/tmp/audit-test -F perm=r -k powercms_read_test
```

`cat /var/tmp/audit-test/read_test.txt` → 監査イベント `161730`、`openat success=yes O_RDONLY`、`PATH`。`less` → `161736`、同上（途中の`readlink EINVAL`はファイル読取失敗を意味しない）。CloudWatchにも`SYSCALL`,`PATH`,`CWD`,`PROCTITLE`到着。**読み取り対象を開いた事実**は示せるが、内容を全部見たか／ネット送信したかの証明ではない。

### 3.3 NFSテスト結果

`findmnt -T /var/www/html -o TARGET,SOURCE,FSTYPE,OPTIONS`：

```text
TARGET   SOURCE       FSTYPE
/var/www 127.0.0.1:/  nfs4
```

`/var/www/audit-test` をec2-user所有・700で用意。一時ルール：

```bash
-a always,exit -F arch=b64 -F dir=/var/www/audit-test -F perm=wa -k powercms_nfs_test
```

| 操作 | 監査イベントID | 結果 |
|---|---|---|
| `touch /var/www/audit-test/nfs_test.txt` | `161974` | `openat success=yes`, `PATH CREATE`, `nfs_t` |
| `chmod 600 ...` | `161975` | `fchmodat success=yes` |
| `rm ...` | `161976` | `unlinkat success=yes`, `PATH DELETE` |

**NFSクライアント側での監査成功**。NFS側への`echo >`の独立`write`イベントは未確認。別EC2からの操作、NFSサーバー側の操作まではこのEC2の監査で網羅できない。

### 3.4 本番用に追加し反映確認した永続ルール

`/etc/audit/rules.d/40-powercms-os.rules`：

```bash
## PowerCMS production - OS configuration
-a always,exit -F arch=b64 -F dir=/etc -F perm=wa -k prod_config_change

## Custom management scripts
-a always,exit -F arch=b64 -F dir=/usr/local/bin -F perm=wa -k prod_script_change
-a always,exit -F arch=b64 -F dir=/usr/local/sbin -F perm=wa -k prod_script_change
```

`/etc/audit/rules.d/50-powercms-web.rules`：

```bash
## PowerCMS production - file modification auditing
-a always,exit -F arch=b64 -F dir=/var/www/html -F perm=wa -k prod_powercms_change
```

`sudo augenrules --check` → 更新必要、`sudo augenrules --load` → `sudo auditctl -l`で**4ルールが反映**されたことをユーザーが実測。最後の `auditctl -s` は `enabled 1, failure 1, pid 1652, rate_limit 0, backlog_limit 64, lost 0, backlog 0, backlog_wait_time 6000`。

`sudo ausearch -k prod_powercms_change -i` の提示結果は、2026-11-02 17:25:56 JST、`type=CONFIG_CHANGE op=add_rule res=yes`。同イベントの`PATH name=/var/www/html`は**監査ルール登録時の参照**であり、PowerCMSファイル改ざんが発生した証拠ではない。本番ファイルの実変更検知は未検証。

負荷確認時：`/var/log/audit/audit.log` 6.1 MB（計測区間不明）、`df -h /var/log` は30GB中11GB使用、20GB空き、36%。今後の増加率・転送量は要測定。

**注意・今後の改善**：
- `powercms_read_test`、`powercms_nfs_test`は`auditctl`による一時追加。`augenrules --load`で消える。永続化済みは30/40/50の`.rules`ファイル。
- `-D`により再読み込み時に既存ルールを削除して再設定するため、意図しない変更時は影響が出る。`auditctl -l`で毎回確認。
- 本番サーバー全体の `perm=rwa` は**非推奨**（Web/PHPの通常読取で大量ログ・CPU/ディスク/CloudWatch/Firehoseコスト、監査イベント喪失）。
- 機密ファイル（PowerCMS DB接続設定・SSH秘密鍵等）の**特定パスだけ `perm=rwa`** を今後追加するが、実ファイルパス未確定。秘密の中身を表示しない。
- `b64`のみ。必要に応じ32bitシステムコールが実行可能か確認し`b32`を設計。
- ログ取得は「ファイル操作を開いた／操作成功」の証拠。実際のファイル内容は別のバックアップ・ハッシュ比較で保全する。
- auditd監査ルールの構成変更自体の監査も重要。`auditctl -e 2`のimmutable化は運用影響が大きいので安易に行わない。

## 4. S3証拠保存・ライフサイクル：直近の未解決課題

### 4.1 ユーザーの最新相談

> 「このs3の設定で、3日以上前のファイルが消されていない。」

添付スクリーンショットで確認したライフサイクル画面：

- バケット：`powercms-log-archive-logs-mchem-infratec-s3`
- ルール名：`delete-logs-after-400days`（**あくまで名前**）
- ステータス：**有効**
- スコープ：**バケット全体**
- **最新バージョンの有効期限切れ：3日**
- **非現行バージョンのアクション：なし**
- オブジェクトサイズ・プレフィックス・タグのフィルターなしと表示。

ユーザーは以前「1日削除テストが残っている」と述べたが、この画像時点のS3ルールは**3日**。CloudWatch側の日数は別途確認が必要。完成後は CloudWatch 30日、S3 400日へ移行する意向。

### 4.2 未解決の原因候補

1. **オブジェクトのキー名にある日付**と**S3オブジェクトLastModified**は違うことがある（Firehose転送の実時刻が重要）。
2. S3 Lifecycleの期限判定はUTC日境界の切上げと非同期削除のため、厳密に72時間後ちょうどに一覧から消えるとは限らない。
3. **バケットのバージョニング有効**時は、現行オブジェクトの期限切れが削除マーカー作成となり、非現行バージョンの実体は残る場合がある。画像では非現行バージョン削除設定なし。
4. **Object Lockの保持・Legal Hold**などで削除が制限される可能性。
5. ライフサイクルの適用条件、他ルールとの競合、対象キー、対象オブジェクトの状態も要確認。

### 4.3 次に行う「読み取り専用」の調査

AWSコンソール：残っている具体的オブジェクトのキー、**最終更新日時（Last modified）**、バージョン履歴、Object Lock保持日時、ルール詳細を確認。古いファイルを手動削除しない。

```bash
# バケットのバージョニング
aws s3api get-bucket-versioning \
  --bucket powercms-log-archive-logs-mchem-infratec-s3 \
  --region ap-northeast-1

# 現行ライフサイクルの実設定
aws s3api get-bucket-lifecycle-configuration \
  --bucket powercms-log-archive-logs-mchem-infratec-s3 \
  --region ap-northeast-1

# 残っている例のキーが分かる場合、日時・メタ情報を確認（KEYを実値へ置換）
aws s3api head-object \
  --bucket powercms-log-archive-logs-mchem-infratec-s3 \
  --key 'KEY' \
  --region ap-northeast-1

# Object Lock設定確認（未設定の場合はエラーの可能性あり）
aws s3api get-object-lock-configuration \
  --bucket powercms-log-archive-logs-mchem-infratec-s3 \
  --region ap-northeast-1
```

参考：`head-object`応答に `Expiration` ヘッダーが出る場合、期限やルールIDの参考になる（表示されない条件もある）。必要に応じ`list-object-versions`で削除マーカーと非現行バージョンを確認。

**ユーザーが指示するまでは、ライフサイクル変更・Object Lock変更・手動削除を行わない。** S3の長期保全と削除試験の進め方を両立する必要がある。

## 5. セキュリティインシデント調査で話題になった事項

- 以前、Apacheログに外部から`.env`や`wp-config.php`を探す自動スキャンらしきアクセスがあった。**一部 `.env` が HTTP 200** だったため、何が実際に返されたか／秘密情報が露出したか／プロキシやキャッシュのどこで応答したかは別途優先調査が必要。HTTP 200だけで漏えい確定ではない。
- SELinuxのAVC例：`php-fpm`の `/var/www/html/mt/addons` 読取、RDSの3306宛接続、`httpd`による`/var/www/html/.htaccess`アクセス。`tcontext=nfs_t`、`permissive=1`。これは**SELinuxが潜在的な拒否を記録したが実際には遮断していない**状態であり、それだけで攻撃確定ではない。
- HTTPリクエストとPHP-FPMの特定PID・ファイル操作を直接1対1で結び付けるIDは未確認。今後、リクエストID、Apache/PHP-FPMアクセスログ、ALB/CloudFront/WAF、VPC Flow Logsの連携を検討。
- RDSからのデータ閲覧／抽出はauditdのファイル監査だけでは捕捉できない。PowerCMS操作ログ、DB監査、ネットワークとの突合が必要。
- `USER_CMD`（sudo/PAM）はコマンドが提示・sudo操作が開始された記録であって、**ファイル実変更の成功証明ではない**。`SYSCALL success=yes` と `PATH`（同じ`msg=audit(epoch:serial)`）を合わせる。
- `AUID`は元のログインユーザー、`UID/EUID`は実際の権限。`AUID=unset`のdaemonやcronは人格・攻撃者を特定するものではない。外部IPも人物そのものを示さない。

## 6. 警察提出を意識した証拠保全の残タスク

1. CloudWatchの5ロググループそれぞれの**到着確認**（イベント時刻、取り込み時刻、直近の件数）。
2. 全サブスクリプションフィルター → Firehose → **S3オブジェクトの実体**の保存・件数・欠落の検証。CloudWatchとS3で何らかのキー／IDによる照合が望ましい。
3. S3の保持期間を最後に400日へ、CloudWatchを30日へ変更（ユーザーの合意方針）。**テスト中の短期削除により証拠消滅リスクがある**ので、不審な事象があれば即座に別途保全。
4. **S3 Object Lock**・バージョニング・権限分離・可能なら別AWSアカウント／別バケットの保全領域を設計。Object LockのComplianceモードなど不可逆性の強い保持は事前了承必須。
5. AWS CloudTrail：継続Trail、管理イベント、必要なデータイベント、ログファイル整合性検証、削除・変更操作の調査可否を確認。
6. 元データ／EBS・NFS/EFS・RDSスナップショット、正常時ファイルハッシュ、調査用コピー、取得者／取得日時／ハッシュ／受け渡し履歴（chain of custody）を用意。
7. 本番稼働に影響を与えない範囲で監査の負荷、`sudo auditctl -s` の `lost`増加、`/var/log`容量、CloudWatch料金の実測値を確認。
8. 機密ファイルの特定パス `perm=rwa` とNFS上の読み取りテスト。`/`全体を`rwa`はしない。
9. CloudFront・Apache・OS・auditd・CMS操作・CMS再構築の横断調査に使うタイムゾーン、相関ID、処理主体の識別を整理。

## 7. 月額コストの過去試算（未実測）

以前、5ログの月間サイズ・費用を**仮定**で試算。

| ログ | 仮定1日 | 月30日 |
|---|---:|---:|
| Apache access | 100 MB | 3.0 GB |
| system | 20 MB | 0.6 GB |
| audit | 50 MB | 1.5 GB |
| cms-activity | 5 MB | 0.15 GB |
| cms-rebuild | 5 MB | 0.15 GB |
| **計** | **180 MB** | **5.4 GB** |

仮の料金レンジは**月約750～1,500円（$5～10、1ドル150円仮定）**。これは実測ではなく、CloudWatch Logs取り込み・保持、Firehose、S3等のざっくり試算。**監査ルール追加によるログ増加、Firehose小サイズレコード課金、CloudTrail等別費用を含めず、実額を保証しない**。料金単価は変動するため新チャットで現行単価を再確認してから正式な試算を行う。正確な見積もりは5ロググループごとにCloudWatch `IncomingBytes`（統計 Sum、1 day）の24時間実測を取得する。

## 8. よく使う確認コマンド・CloudWatch Logs Insights

### Linux auditd（閲覧のみ）

```bash
sudo auditctl -l
sudo auditctl -s
sudo ausearch -k prod_config_change -i
sudo ausearch -k prod_script_change -i
sudo ausearch -k prod_powercms_change -i
sudo ls -l /etc/audit/rules.d/
findmnt -T /var/www/html -o TARGET,SOURCE,FSTYPE,OPTIONS
sudo du -h /var/log/audit/audit.log
df -h /var/log
```

### CMS定期取得（閲覧のみ）

```bash
sudo systemctl status prod_mchem_infratec_log_export.timer --no-pager
sudo systemctl status prod_mchem_infratec_log_export.service --no-pager
sudo systemctl cat prod_mchem_infratec_log_export.timer
sudo systemctl cat prod_mchem_infratec_log_export.service
sudo journalctl -u prod_mchem_infratec_log_export.service --since '30 minutes ago' --no-pager
sudo ls -l /var/log/powercms/prod_mchem_infratec_rebuild.log
sudo grep -Rn -C 4 'cms-rebuild' /opt/aws/amazon-cloudwatch-agent/etc/
```

※`systemctl cat`やスクリプト全文にシークレットがあれば伏せること。

### CloudWatch Logs Insights

監査ロググループ `/powercms/mchem-infratec-rds-prod/audit`：

```sql
fields @timestamp, @message
| filter @message like /prod_config_change|prod_script_change|prod_powercms_change/
| sort @timestamp desc
| limit 200
```

特定の監査イベントを相関（同一serialの`SYSCALL`/`PATH`/`CWD`/`PROCTITLE`）：

```sql
fields @timestamp, @message
| filter @message like /161974|161975|161976/
| sort @timestamp asc
| limit 200
```

CMS再構築ロググループ `/powercms/mchem-infratec-rds-prod/cms-rebuild`：

```sql
fields @timestamp, @message
| sort @timestamp desc
| limit 30
```

```sql
stats count(*) as totalEvents,
      max(@timestamp) as latestEventTime,
      max(@ingestionTime) as latestIngestionTime
```

※再構築が発生しない間はログが増えなくてもtimer障害とは限らない。`@timestamp`と`@ingestionTime`を区別。ログタイムゾーンは **JST（UTC+9）** を明示して解釈する。

## 9. 次のチャットへの推奨タスク順序

**直近のユーザーの質問への継続**：S3の「3日削除になっているのに古いログが残っている」問題を、設定変更なしでまず調査する。

1. ユーザーが示す「残ったファイル」1件のS3 `Last modified`とキー（秘匿値が含まれる場合は適切に伏せる）を確認。
2. `get-bucket-versioning` と `get-bucket-lifecycle-configuration` を確認。
3. バージョニング有効なら削除マーカー／非現行バージョンの有無を確認。`head-object`の`Expiration`、Object Lockの設定も確認。
4. 3日経過の解釈をUTCの日境界込みで整理し、非同期処理による遅延か、ルール不適用かを判断。
5. 同時に、Firehose→S3の`cms-rebuild/prod`プレフィックスへの到着状況を確認（CloudWatchの収集はユーザーの最新スクリーンショットで確認済み）。
6. 後続で機密ファイルの読み取り監査、S3証拠保全、CloudTrail等に進む。

### 他のチャットに最初に送ると効率が良い依頼文

> この引き継ぎMarkdownを前提に、PowerCMS本番のログ収集・auditd・証拠保全の作業を継続してください。まず最新課題である、S3バケット `powercms-log-archive-logs-mchem-infratec-s3` のライフサイクルルールが「有効・3日」なのに3日超のログが残る原因を、読み取り専用で切り分けてください。既存設定やログは勝手に削除・変更せず、確認済みと未確認を明確に区別し、EC2/AWS CLIコマンドを日本語で段階的に示してください。

---

## 10. 作業時の安全・表現上の注意

- **本番**なので、既存CMSファイル、S3オブジェクト、stateファイル、監査ルールをむやみに変更・消去しない。読み取り専用の切り分けから始める。
- 共有NFS `/var/www` 上に安易にWeb公開可能なテストファイルを追加しない。以前のテストは`/var/www/audit-test`（`/var/www/html`外）で実施。
- 監査ルールの`-D`は全ルール再読み込みを伴う。実際の`auditctl -l`の結果で確認しながら運用する。
- S3ライフサイクルは非同期。Object Lockは保持期間中の削除制限が強く、コンプライアンス設計・権限分離を考慮する。
- CloudWatchのログを受信できていても、S3証拠保全は別の成功判定が必要。
- 最終成果は「犯罪事実を自動的に立証できる」ことではなく、複数の客観記録を整合的かつ改ざんに強い形で取得し、調査・提出可能にすること。

---

# 詳細運用付録 — 第2版（次チャット継続用）

> **この付録は先頭の要約（0〜10章）を置き換えず、調査・再現・検証手順を増補する。**
> 全てのコマンドは特記ない限り例示であり、実行済みの事実を示すものではない。
> 文中の `確認済み` はユーザーが貼ったログ／画面で裏付けられた事項、`未確認` は実環境での確証がない事項。
> `READ ONLY` は参照系コマンド、`CHANGE` は設定変更、`EVIDENCE` は証拠保全作業を表す。
> 特に本番EC2・共有NFS・S3の削除設定へ変更を加える前に、適用対象・戻し方・影響を整理する。
> この資料の行数は品質目標であり、行数だけのための架空イベントや架空設定は挿入しない。

## 11. 固定事実台帳（引き継ぎ先が再確認しなくてもよい情報）

この章の値は、**会話時点のスナップショット**。稼働状況・容量・PID・保存日数は変わり得る。

### F-001 リージョン

- **観測値**：`ap-northeast-1`

- **根拠**：会話中のCloudWatchとAWS画面

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-002 EC2ホスト名

- **観測値**：`ip-172-21-3-236`

- **根拠**：ユーザーが貼ったシェルプロンプト

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-003 EC2 private IP

- **観測値**：`172.21.3.236`

- **根拠**：会話で共有されたインフラ背景

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-004 OS

- **観測値**：`Amazon Linux 2023`

- **根拠**：会話中の操作環境

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-005 PowerCMSバージョン

- **観測値**：`6.8.4`

- **根拠**：過去のプロジェクト背景

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-006 DBエンジン

- **観測値**：`RDS MySQL 8.4系`

- **根拠**：過去のプロジェクト背景

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-007 PowerCMS Webルート

- **観測値**：`/var/www/html`

- **根拠**：findmntとauditd対象

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-008 NFSマウントポイント

- **観測値**：`/var/www`

- **根拠**：findmntのTARGET

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-009 NFS SOURCE

- **観測値**：`127.0.0.1:/`

- **根拠**：findmntのSOURCE

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-010 NFS FSTYPE

- **観測値**：`nfs4`

- **根拠**：findmntのFSTYPE

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-011 監査daemon PID

- **観測値**：`1652`

- **根拠**：auditctl -sの一時点

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-012 監査 enabled

- **観測値**：`1`

- **根拠**：auditctl -s

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-013 監査 failure

- **観測値**：`1`

- **根拠**：auditctl -s

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-014 監査 backlog_limit

- **観測値**：`64`

- **根拠**：auditctl -s

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-015 監査 lost

- **観測値**：`0`

- **根拠**：当該確認時点でのauditctl -s

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-016 監査 backlog

- **観測値**：`0`

- **根拠**：当該確認時点でのauditctl -s

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-017 監査 loginuid_immutable

- **観測値**：`0 unlocked`

- **根拠**：auditctl -s

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-018 ローカルauditログ容量

- **観測値**：`6.1M`

- **根拠**：du -hの一時点

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-019 ルートディスク

- **観測値**：`/dev/nvme0n1p1`

- **根拠**：df -h

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-020 ルートディスクサイズ

- **観測値**：`30G`

- **根拠**：df -h

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-021 ルートディスク使用

- **観測値**：`11G/36%`

- **根拠**：df -hの一時点

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-022 ルートディスク空き

- **観測値**：`20G`

- **根拠**：df -hの一時点

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-023 NFS df見かけ容量

- **観測値**：`8.0E`

- **根拠**：容量見積りに使用不可

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-024 テストローカルdir

- **観測値**：`/var/tmp/audit-test`

- **根拠**：成功したauditテスト

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-025 テストNFS dir

- **観測値**：`/var/www/audit-test`

- **根拠**：成功したNFSテスト

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-026 操作ログテストキー

- **観測値**：`powercms_file_test`

- **根拠**：auditctl -l

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-027 読取ログテストキー

- **観測値**：`powercms_read_test`

- **根拠**：一時auditctlルール

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-028 NFSテストキー

- **観測値**：`powercms_nfs_test`

- **根拠**：一時auditctlルール

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-029 本番OS設定監査キー

- **観測値**：`prod_config_change`

- **根拠**：永続ルール

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-030 本番スクリプト監査キー

- **観測値**：`prod_script_change`

- **根拠**：永続ルール

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-031 本番Web監査キー

- **観測値**：`prod_powercms_change`

- **根拠**：永続ルール

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-032 S3 bucket

- **観測値**：`powercms-log-archive-logs-mchem-infratec-s3`

- **根拠**：画面

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-033 S3ライフサイクル名

- **観測値**：`delete-logs-after-400days`

- **根拠**：画面

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-034 S3実際の現行期限

- **観測値**：`3日`

- **根拠**：2026-11-04の画面

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-035 S3ライフサイクル対象

- **観測値**：`バケット全体`

- **根拠**：2026-11-04の画面

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-036 S3非現行期限

- **観測値**：`設定なし（画面）`

- **根拠**：設定有無の追加CLI確認が必要

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-037 CMSログエクスポートtimer

- **観測値**：`prod_mchem_infratec_log_export.timer`

- **根拠**：systemctl

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-038 CMSログエクスポートservice

- **観測値**：`prod_mchem_infratec_log_export.service`

- **根拠**：systemctl

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-039 timer設定状態

- **観測値**：`enabled / active(waiting)`

- **根拠**：systemctl

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-040 timerスケジュール

- **観測値**：`毎分起動`

- **根拠**：systemctl list-timersとjournalctl

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-041 activity Python

- **観測値**：`/usr/local/sbin/prod_mchem_infratec_export_activity.py`

- **根拠**：service ExecStart

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-042 rebuild Python

- **観測値**：`/usr/local/sbin/prod_mchem_infratec_export_rebuild.py`

- **根拠**：service ExecStart

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-043 rebuild lastid

- **観測値**：`/var/lib/powercms-log-export/prod_mchem_infratec_rebuild.lastid`

- **根拠**：grepで確認

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-044 rebuild出力先

- **観測値**：`/var/log/powercms/prod_mchem_infratec_rebuild.log`

- **根拠**：fileとAgent設定

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-045 rebuildファイル権限

- **観測値**：`root:root 0600`

- **根拠**：ls/find結果

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-046 rebuildファイルサイズ

- **観測値**：`7416バイト`

- **根拠**：2026-11-04 12:10頃

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-047 rebuild CloudWatch stream

- **観測値**：`i-0bd1b63ca374f1f78`

- **根拠**：Agent設定とスクリーンショット

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-048 Agent起動

- **観測値**：`2026-11-02 10:49 JST頃`

- **根拠**：systemctl

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-049 Agent状態

- **観測値**：`active(running)`

- **根拠**：systemctl

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

### F-050 CloudWatch rebuild visible

- **観測値**：`2026-11-04 11:57と12:10頃`

- **根拠**：All eventsのスクリーンショット

- **使用上の注意**：変更有無を疑う場面では、現環境で読み取り専用の再確認をする。

- **引き継ぎ先の処理**：この既知情報の再質問を避け、状況変化の有無だけ問う。

## 12. 5系統の収集経路別・再現可能な運用手順

各ログ系統について、動作、CloudWatch、S3、障害時の疑点、セキュリティ上の限界を個別に記す。

### 12.1 Apacheアクセス（ACCESS）

- CloudWatchロググループ：`/powercms/mchem-infratec-rds-prod/access`

- ローカル発生源：`/var/log/httpd/mchem-infratec_access.log`

- データの起点：httpdのアクセス出力

- S3予定prefix：`access-log/prod/`

- 解釈上の警告：HTTP 200だけで攻撃成功、情報流出成功とは断定しない。xff等はプロキシ構成に依存。

#### 1. 発生源の存在を確認

- 対象：`/powercms/mchem-infratec-rds-prod/access`

- 確認目的：発生源が実際にデータを生成しているか。

- 判定注意：ローカル側にまだイベントがないだけならCloudWatchにゼロ件でも直ちに故障ではない。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 2. サービスの稼働を確認

- 対象：`/powercms/mchem-infratec-rds-prod/access`

- 確認目的：データ生成元のプロセスが起動または定期実行されているか。

- 判定注意：常駐型とoneshot型でactive/inactiveの解釈が違う。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 3. 時間軸を確認

- 対象：`/powercms/mchem-infratec-rds-prod/access`

- 確認目的：ホスト時刻、JST/UTC表示、対象イベント時刻を分離する。

- 判定注意：CloudWatch画面の表示時刻がローカルタイムか確認。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 4. Agent実効設定を確認

- 対象：`/powercms/mchem-infratec-rds-prod/access`

- 確認目的：Agentの生成済みtomlと構成断片を対照する。

- 判定注意：設定断片の存在だけでロード済みとは断定しない。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 5. Agentの処理ログ確認

- 対象：`/powercms/mchem-infratec-rds-prod/access`

- 確認目的：pathやgroup名に関連するエラーやデータ転送の状況を調べる。

- 判定注意：pipingログは送信成功の証拠ではない。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 6. CloudWatchストリーム確認

- 対象：`/powercms/mchem-infratec-rds-prod/access`

- 確認目的：ストリームが存在しているか、最新イベントが見えるかを確認。

- 判定注意：前回はrebuildの0件表示を別画面で解消したため見落とし注意。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 7. ログイベント中身確認

- 対象：`/powercms/mchem-infratec-rds-prod/access`

- 確認目的：期待するスキーマ・レコード識別子があるか。

- 判定注意：件数だけで意味内容や完全性を保証しない。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 8. Subscription filter確認

- 対象：`/powercms/mchem-infratec-rds-prod/access`

- 確認目的：groupからFirehoseへの配信設定を確認する。

- 判定注意：見えているフィルターと配信成功は別。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 9. Firehose監視

- 対象：`/powercms/mchem-infratec-rds-prod/access`

- 確認目的：配信成功・失敗・バッファ・エラーログを確認。

- 判定注意：転送遅延と失敗を分離する。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 10. S3オブジェクト確認

- 対象：`/powercms/mchem-infratec-rds-prod/access`

- 確認目的：対象prefixにオブジェクトが存在するか。

- 判定注意：オブジェクトの存在だけで全イベント完全性は証明しない。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 11. 保存期間の確認

- 対象：`/powercms/mchem-infratec-rds-prod/access`

- 確認目的：CloudWatch保持期間とS3ルールを別々に確認。

- 判定注意：現状S3実設定3日、CloudWatchは短期テストと過去申告。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 12. 証拠保全適性

- 対象：`/powercms/mchem-infratec-rds-prod/access`

- 確認目的：原本・取得時刻・SHA-256・受け渡し記録を残せるか。

- 判定注意：現時点ではObject Lock完了の証拠がない。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### AWS CLI：CloudWatch側の読み取り専用確認

```bash

aws logs describe-log-groups --region ap-northeast-1 --log-group-name-prefix "/powercms/mchem-infratec-rds-prod/access"

aws logs describe-log-streams --region ap-northeast-1 --log-group-name "/powercms/mchem-infratec-rds-prod/access"

aws logs describe-subscription-filters --region ap-northeast-1 --log-group-name "/powercms/mchem-infratec-rds-prod/access"

```

#### CloudWatch Logs Insights：直近イベントの確認

```sql
fields @timestamp, @ingestionTime, @logStream, @message
| sort @timestamp desc
| limit 50
```

#### CloudWatch Logs Insights：イベント件数の確認

```sql
stats count(*) as n by bin(1h)
| sort bin(1h) desc
```

#### CloudWatch Logs Insights：対象キーワードの調査

```sql
fields @timestamp, @message
| filter @message like /\.env|wp-config|POST|GET/
| sort @timestamp asc
| limit 200
```

#### Firehose/S3配信の評価メモ

- **証拠起点**：CloudWatchへのイベント到着は画面やCLIで確認する。

- **配信時刻**：S3のLastModifiedと元の@timestampを分けて保存する。

- **突合単位**：一定の狭い時間範囲と特徴的イベントIDを使う。

- **削除期限**：現在S3の現行版は実際3日、予定400日は未適用。

- **変更許可**：リトライ・バッファ・保存期間変更には明示的な承認を取る。

- **未検証**：S3へ全イベント欠落なしを保証する検証は未完了。

### 12.2 OSシステム（SYSTEM）

- CloudWatchロググループ：`/powercms/mchem-infratec-rds-prod/system`

- ローカル発生源：`journald（単一ファイルではない）`

- データの起点：systemd-journald、sudo、PAM

- S3予定prefix：`system-log/prod/`

- 解釈上の警告：sudoの成功ログは対象ファイルへの操作成功を直接証明しない。

#### 1. 発生源の存在を確認

- 対象：`/powercms/mchem-infratec-rds-prod/system`

- 確認目的：発生源が実際にデータを生成しているか。

- 判定注意：ローカル側にまだイベントがないだけならCloudWatchにゼロ件でも直ちに故障ではない。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 2. サービスの稼働を確認

- 対象：`/powercms/mchem-infratec-rds-prod/system`

- 確認目的：データ生成元のプロセスが起動または定期実行されているか。

- 判定注意：常駐型とoneshot型でactive/inactiveの解釈が違う。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 3. 時間軸を確認

- 対象：`/powercms/mchem-infratec-rds-prod/system`

- 確認目的：ホスト時刻、JST/UTC表示、対象イベント時刻を分離する。

- 判定注意：CloudWatch画面の表示時刻がローカルタイムか確認。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 4. Agent実効設定を確認

- 対象：`/powercms/mchem-infratec-rds-prod/system`

- 確認目的：Agentの生成済みtomlと構成断片を対照する。

- 判定注意：設定断片の存在だけでロード済みとは断定しない。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 5. Agentの処理ログ確認

- 対象：`/powercms/mchem-infratec-rds-prod/system`

- 確認目的：pathやgroup名に関連するエラーやデータ転送の状況を調べる。

- 判定注意：pipingログは送信成功の証拠ではない。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 6. CloudWatchストリーム確認

- 対象：`/powercms/mchem-infratec-rds-prod/system`

- 確認目的：ストリームが存在しているか、最新イベントが見えるかを確認。

- 判定注意：前回はrebuildの0件表示を別画面で解消したため見落とし注意。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 7. ログイベント中身確認

- 対象：`/powercms/mchem-infratec-rds-prod/system`

- 確認目的：期待するスキーマ・レコード識別子があるか。

- 判定注意：件数だけで意味内容や完全性を保証しない。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 8. Subscription filter確認

- 対象：`/powercms/mchem-infratec-rds-prod/system`

- 確認目的：groupからFirehoseへの配信設定を確認する。

- 判定注意：見えているフィルターと配信成功は別。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 9. Firehose監視

- 対象：`/powercms/mchem-infratec-rds-prod/system`

- 確認目的：配信成功・失敗・バッファ・エラーログを確認。

- 判定注意：転送遅延と失敗を分離する。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 10. S3オブジェクト確認

- 対象：`/powercms/mchem-infratec-rds-prod/system`

- 確認目的：対象prefixにオブジェクトが存在するか。

- 判定注意：オブジェクトの存在だけで全イベント完全性は証明しない。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 11. 保存期間の確認

- 対象：`/powercms/mchem-infratec-rds-prod/system`

- 確認目的：CloudWatch保持期間とS3ルールを別々に確認。

- 判定注意：現状S3実設定3日、CloudWatchは短期テストと過去申告。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 12. 証拠保全適性

- 対象：`/powercms/mchem-infratec-rds-prod/system`

- 確認目的：原本・取得時刻・SHA-256・受け渡し記録を残せるか。

- 判定注意：現時点ではObject Lock完了の証拠がない。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### AWS CLI：CloudWatch側の読み取り専用確認

```bash

aws logs describe-log-groups --region ap-northeast-1 --log-group-name-prefix "/powercms/mchem-infratec-rds-prod/system"

aws logs describe-log-streams --region ap-northeast-1 --log-group-name "/powercms/mchem-infratec-rds-prod/system"

aws logs describe-subscription-filters --region ap-northeast-1 --log-group-name "/powercms/mchem-infratec-rds-prod/system"

```

#### CloudWatch Logs Insights：直近イベントの確認

```sql
fields @timestamp, @ingestionTime, @logStream, @message
| sort @timestamp desc
| limit 50
```

#### CloudWatch Logs Insights：イベント件数の確認

```sql
stats count(*) as n by bin(1h)
| sort bin(1h) desc
```

#### CloudWatch Logs Insights：対象キーワードの調査

```sql
fields @timestamp, @message
| filter @message like /sudo|session opened|session closed/
| sort @timestamp asc
| limit 200
```

#### Firehose/S3配信の評価メモ

- **証拠起点**：CloudWatchへのイベント到着は画面やCLIで確認する。

- **配信時刻**：S3のLastModifiedと元の@timestampを分けて保存する。

- **突合単位**：一定の狭い時間範囲と特徴的イベントIDを使う。

- **削除期限**：現在S3の現行版は実際3日、予定400日は未適用。

- **変更許可**：リトライ・バッファ・保存期間変更には明示的な承認を取る。

- **未検証**：S3へ全イベント欠落なしを保証する検証は未完了。

### 12.3 Linux auditd（AUDIT）

- CloudWatchロググループ：`/powercms/mchem-infratec-rds-prod/audit`

- ローカル発生源：`/var/log/audit/audit.log`

- データの起点：auditd kernel record

- S3予定prefix：`audit-log/prod/`

- 解釈上の警告：同一msg=audit(...:serial)をまとめ、SYSCALLとPATHを組み合わせる。

#### 1. 発生源の存在を確認

- 対象：`/powercms/mchem-infratec-rds-prod/audit`

- 確認目的：発生源が実際にデータを生成しているか。

- 判定注意：ローカル側にまだイベントがないだけならCloudWatchにゼロ件でも直ちに故障ではない。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 2. サービスの稼働を確認

- 対象：`/powercms/mchem-infratec-rds-prod/audit`

- 確認目的：データ生成元のプロセスが起動または定期実行されているか。

- 判定注意：常駐型とoneshot型でactive/inactiveの解釈が違う。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 3. 時間軸を確認

- 対象：`/powercms/mchem-infratec-rds-prod/audit`

- 確認目的：ホスト時刻、JST/UTC表示、対象イベント時刻を分離する。

- 判定注意：CloudWatch画面の表示時刻がローカルタイムか確認。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 4. Agent実効設定を確認

- 対象：`/powercms/mchem-infratec-rds-prod/audit`

- 確認目的：Agentの生成済みtomlと構成断片を対照する。

- 判定注意：設定断片の存在だけでロード済みとは断定しない。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 5. Agentの処理ログ確認

- 対象：`/powercms/mchem-infratec-rds-prod/audit`

- 確認目的：pathやgroup名に関連するエラーやデータ転送の状況を調べる。

- 判定注意：pipingログは送信成功の証拠ではない。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 6. CloudWatchストリーム確認

- 対象：`/powercms/mchem-infratec-rds-prod/audit`

- 確認目的：ストリームが存在しているか、最新イベントが見えるかを確認。

- 判定注意：前回はrebuildの0件表示を別画面で解消したため見落とし注意。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 7. ログイベント中身確認

- 対象：`/powercms/mchem-infratec-rds-prod/audit`

- 確認目的：期待するスキーマ・レコード識別子があるか。

- 判定注意：件数だけで意味内容や完全性を保証しない。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 8. Subscription filter確認

- 対象：`/powercms/mchem-infratec-rds-prod/audit`

- 確認目的：groupからFirehoseへの配信設定を確認する。

- 判定注意：見えているフィルターと配信成功は別。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 9. Firehose監視

- 対象：`/powercms/mchem-infratec-rds-prod/audit`

- 確認目的：配信成功・失敗・バッファ・エラーログを確認。

- 判定注意：転送遅延と失敗を分離する。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 10. S3オブジェクト確認

- 対象：`/powercms/mchem-infratec-rds-prod/audit`

- 確認目的：対象prefixにオブジェクトが存在するか。

- 判定注意：オブジェクトの存在だけで全イベント完全性は証明しない。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 11. 保存期間の確認

- 対象：`/powercms/mchem-infratec-rds-prod/audit`

- 確認目的：CloudWatch保持期間とS3ルールを別々に確認。

- 判定注意：現状S3実設定3日、CloudWatchは短期テストと過去申告。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 12. 証拠保全適性

- 対象：`/powercms/mchem-infratec-rds-prod/audit`

- 確認目的：原本・取得時刻・SHA-256・受け渡し記録を残せるか。

- 判定注意：現時点ではObject Lock完了の証拠がない。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### AWS CLI：CloudWatch側の読み取り専用確認

```bash

aws logs describe-log-groups --region ap-northeast-1 --log-group-name-prefix "/powercms/mchem-infratec-rds-prod/audit"

aws logs describe-log-streams --region ap-northeast-1 --log-group-name "/powercms/mchem-infratec-rds-prod/audit"

aws logs describe-subscription-filters --region ap-northeast-1 --log-group-name "/powercms/mchem-infratec-rds-prod/audit"

```

#### CloudWatch Logs Insights：直近イベントの確認

```sql
fields @timestamp, @ingestionTime, @logStream, @message
| sort @timestamp desc
| limit 50
```

#### CloudWatch Logs Insights：イベント件数の確認

```sql
stats count(*) as n by bin(1h)
| sort bin(1h) desc
```

#### CloudWatch Logs Insights：対象キーワードの調査

```sql
fields @timestamp, @message
| filter @message like /SYSCALL|PATH|PROCTITLE|CONFIG_CHANGE/
| sort @timestamp asc
| limit 200
```

#### Firehose/S3配信の評価メモ

- **証拠起点**：CloudWatchへのイベント到着は画面やCLIで確認する。

- **配信時刻**：S3のLastModifiedと元の@timestampを分けて保存する。

- **突合単位**：一定の狭い時間範囲と特徴的イベントIDを使う。

- **削除期限**：現在S3の現行版は実際3日、予定400日は未適用。

- **変更許可**：リトライ・バッファ・保存期間変更には明示的な承認を取る。

- **未検証**：S3へ全イベント欠落なしを保証する検証は未完了。

### 12.4 PowerCMS操作（ACTIVITY）

- CloudWatchロググループ：`/powercms/mchem-infratec-rds-prod/cms-activity`

- ローカル発生源：`/var/log/powercms/prod_mchem_infratec_activity.log`

- データの起点：RDS mt_logからPythonでJSONL出力

- S3予定prefix：`cms-activity/prod/`

- 解釈上の警告：管理画面外・直接DB書き換え・OS側改ざんを網羅できない。

#### 1. 発生源の存在を確認

- 対象：`/powercms/mchem-infratec-rds-prod/cms-activity`

- 確認目的：発生源が実際にデータを生成しているか。

- 判定注意：ローカル側にまだイベントがないだけならCloudWatchにゼロ件でも直ちに故障ではない。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 2. サービスの稼働を確認

- 対象：`/powercms/mchem-infratec-rds-prod/cms-activity`

- 確認目的：データ生成元のプロセスが起動または定期実行されているか。

- 判定注意：常駐型とoneshot型でactive/inactiveの解釈が違う。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 3. 時間軸を確認

- 対象：`/powercms/mchem-infratec-rds-prod/cms-activity`

- 確認目的：ホスト時刻、JST/UTC表示、対象イベント時刻を分離する。

- 判定注意：CloudWatch画面の表示時刻がローカルタイムか確認。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 4. Agent実効設定を確認

- 対象：`/powercms/mchem-infratec-rds-prod/cms-activity`

- 確認目的：Agentの生成済みtomlと構成断片を対照する。

- 判定注意：設定断片の存在だけでロード済みとは断定しない。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 5. Agentの処理ログ確認

- 対象：`/powercms/mchem-infratec-rds-prod/cms-activity`

- 確認目的：pathやgroup名に関連するエラーやデータ転送の状況を調べる。

- 判定注意：pipingログは送信成功の証拠ではない。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 6. CloudWatchストリーム確認

- 対象：`/powercms/mchem-infratec-rds-prod/cms-activity`

- 確認目的：ストリームが存在しているか、最新イベントが見えるかを確認。

- 判定注意：前回はrebuildの0件表示を別画面で解消したため見落とし注意。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 7. ログイベント中身確認

- 対象：`/powercms/mchem-infratec-rds-prod/cms-activity`

- 確認目的：期待するスキーマ・レコード識別子があるか。

- 判定注意：件数だけで意味内容や完全性を保証しない。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 8. Subscription filter確認

- 対象：`/powercms/mchem-infratec-rds-prod/cms-activity`

- 確認目的：groupからFirehoseへの配信設定を確認する。

- 判定注意：見えているフィルターと配信成功は別。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 9. Firehose監視

- 対象：`/powercms/mchem-infratec-rds-prod/cms-activity`

- 確認目的：配信成功・失敗・バッファ・エラーログを確認。

- 判定注意：転送遅延と失敗を分離する。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 10. S3オブジェクト確認

- 対象：`/powercms/mchem-infratec-rds-prod/cms-activity`

- 確認目的：対象prefixにオブジェクトが存在するか。

- 判定注意：オブジェクトの存在だけで全イベント完全性は証明しない。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 11. 保存期間の確認

- 対象：`/powercms/mchem-infratec-rds-prod/cms-activity`

- 確認目的：CloudWatch保持期間とS3ルールを別々に確認。

- 判定注意：現状S3実設定3日、CloudWatchは短期テストと過去申告。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 12. 証拠保全適性

- 対象：`/powercms/mchem-infratec-rds-prod/cms-activity`

- 確認目的：原本・取得時刻・SHA-256・受け渡し記録を残せるか。

- 判定注意：現時点ではObject Lock完了の証拠がない。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### AWS CLI：CloudWatch側の読み取り専用確認

```bash

aws logs describe-log-groups --region ap-northeast-1 --log-group-name-prefix "/powercms/mchem-infratec-rds-prod/cms-activity"

aws logs describe-log-streams --region ap-northeast-1 --log-group-name "/powercms/mchem-infratec-rds-prod/cms-activity"

aws logs describe-subscription-filters --region ap-northeast-1 --log-group-name "/powercms/mchem-infratec-rds-prod/cms-activity"

```

#### CloudWatch Logs Insights：直近イベントの確認

```sql
fields @timestamp, @ingestionTime, @logStream, @message
| sort @timestamp desc
| limit 50
```

#### CloudWatch Logs Insights：イベント件数の確認

```sql
stats count(*) as n by bin(1h)
| sort bin(1h) desc
```

#### CloudWatch Logs Insights：対象キーワードの調査

```sql
fields @timestamp, @message
| filter @message like /mt_log/
| sort @timestamp asc
| limit 200
```

#### Firehose/S3配信の評価メモ

- **証拠起点**：CloudWatchへのイベント到着は画面やCLIで確認する。

- **配信時刻**：S3のLastModifiedと元の@timestampを分けて保存する。

- **突合単位**：一定の狭い時間範囲と特徴的イベントIDを使う。

- **削除期限**：現在S3の現行版は実際3日、予定400日は未適用。

- **変更許可**：リトライ・バッファ・保存期間変更には明示的な承認を取る。

- **未検証**：S3へ全イベント欠落なしを保証する検証は未完了。

### 12.5 PowerCMS再構築（REBUILD）

- CloudWatchロググループ：`/powercms/mchem-infratec-rds-prod/cms-rebuild`

- ローカル発生源：`/var/log/powercms/prod_mchem_infratec_rebuild.log`

- データの起点：RDS mt_rebuildlogからPythonでJSONL出力

- S3予定prefix：`cms-rebuild/prod/`

- 解釈上の警告：2026-11-04 CloudWatch All eventsで実ログ着信確認済み。DB→S3完全性は未検証。

#### 1. 発生源の存在を確認

- 対象：`/powercms/mchem-infratec-rds-prod/cms-rebuild`

- 確認目的：発生源が実際にデータを生成しているか。

- 判定注意：ローカル側にまだイベントがないだけならCloudWatchにゼロ件でも直ちに故障ではない。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 2. サービスの稼働を確認

- 対象：`/powercms/mchem-infratec-rds-prod/cms-rebuild`

- 確認目的：データ生成元のプロセスが起動または定期実行されているか。

- 判定注意：常駐型とoneshot型でactive/inactiveの解釈が違う。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 3. 時間軸を確認

- 対象：`/powercms/mchem-infratec-rds-prod/cms-rebuild`

- 確認目的：ホスト時刻、JST/UTC表示、対象イベント時刻を分離する。

- 判定注意：CloudWatch画面の表示時刻がローカルタイムか確認。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 4. Agent実効設定を確認

- 対象：`/powercms/mchem-infratec-rds-prod/cms-rebuild`

- 確認目的：Agentの生成済みtomlと構成断片を対照する。

- 判定注意：設定断片の存在だけでロード済みとは断定しない。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 5. Agentの処理ログ確認

- 対象：`/powercms/mchem-infratec-rds-prod/cms-rebuild`

- 確認目的：pathやgroup名に関連するエラーやデータ転送の状況を調べる。

- 判定注意：pipingログは送信成功の証拠ではない。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 6. CloudWatchストリーム確認

- 対象：`/powercms/mchem-infratec-rds-prod/cms-rebuild`

- 確認目的：ストリームが存在しているか、最新イベントが見えるかを確認。

- 判定注意：前回はrebuildの0件表示を別画面で解消したため見落とし注意。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 7. ログイベント中身確認

- 対象：`/powercms/mchem-infratec-rds-prod/cms-rebuild`

- 確認目的：期待するスキーマ・レコード識別子があるか。

- 判定注意：件数だけで意味内容や完全性を保証しない。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 8. Subscription filter確認

- 対象：`/powercms/mchem-infratec-rds-prod/cms-rebuild`

- 確認目的：groupからFirehoseへの配信設定を確認する。

- 判定注意：見えているフィルターと配信成功は別。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 9. Firehose監視

- 対象：`/powercms/mchem-infratec-rds-prod/cms-rebuild`

- 確認目的：配信成功・失敗・バッファ・エラーログを確認。

- 判定注意：転送遅延と失敗を分離する。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 10. S3オブジェクト確認

- 対象：`/powercms/mchem-infratec-rds-prod/cms-rebuild`

- 確認目的：対象prefixにオブジェクトが存在するか。

- 判定注意：オブジェクトの存在だけで全イベント完全性は証明しない。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 11. 保存期間の確認

- 対象：`/powercms/mchem-infratec-rds-prod/cms-rebuild`

- 確認目的：CloudWatch保持期間とS3ルールを別々に確認。

- 判定注意：現状S3実設定3日、CloudWatchは短期テストと過去申告。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### 12. 証拠保全適性

- 対象：`/powercms/mchem-infratec-rds-prod/cms-rebuild`

- 確認目的：原本・取得時刻・SHA-256・受け渡し記録を残せるか。

- 判定注意：現時点ではObject Lock完了の証拠がない。

- 運用上の処理：異常があっても最初は再起動せず、読取結果を記録してから変更案を作る。

#### AWS CLI：CloudWatch側の読み取り専用確認

```bash

aws logs describe-log-groups --region ap-northeast-1 --log-group-name-prefix "/powercms/mchem-infratec-rds-prod/cms-rebuild"

aws logs describe-log-streams --region ap-northeast-1 --log-group-name "/powercms/mchem-infratec-rds-prod/cms-rebuild"

aws logs describe-subscription-filters --region ap-northeast-1 --log-group-name "/powercms/mchem-infratec-rds-prod/cms-rebuild"

```

#### CloudWatch Logs Insights：直近イベントの確認

```sql
fields @timestamp, @ingestionTime, @logStream, @message
| sort @timestamp desc
| limit 50
```

#### CloudWatch Logs Insights：イベント件数の確認

```sql
stats count(*) as n by bin(1h)
| sort bin(1h) desc
```

#### CloudWatch Logs Insights：対象キーワードの調査

```sql
fields @timestamp, @message
| filter @message like /mt_rebuildlog/
| sort @timestamp asc
| limit 200
```

#### Firehose/S3配信の評価メモ

- **証拠起点**：CloudWatchへのイベント到着は画面やCLIで確認する。

- **配信時刻**：S3のLastModifiedと元の@timestampを分けて保存する。

- **突合単位**：一定の狭い時間範囲と特徴的イベントIDを使う。

- **削除期限**：現在S3の現行版は実際3日、予定400日は未適用。

- **変更許可**：リトライ・バッファ・保存期間変更には明示的な承認を取る。

- **未検証**：S3へ全イベント欠落なしを保証する検証は未完了。

## 13. auditdトラブルシューティング／検証の詳細台帳

次の手順は過去の実例に基づく。時刻、監査キー、イベント番号が揃う場合は個別イベントとして証拠保全する。

### A01 初期タスク抑止

- **対象パス**：`/etc/audit/rules.d/audit.rules`

- **実施操作・ルール**：`-a task,never`

- **ユーザーが示した結果**：-a never,task がauditctlに表示

- **認められる解釈**：sudoなしファイル操作を直接記録できない

- **注意・未証明部分**：初期状態の説明に使う

- **実務**：同じ監査シリアルの `SYSCALL`、`PATH`、`CWD`、`PROCTITLE` をまとめる。

- **証拠保全**：原本は改変せずコピーへハッシュを付け、JST表示と監査epochを保持する。

- **次回の確認**：異常がある場合は `sudo ausearch -k <監査キー> -i` を実行しログ損失指標と照合。

### A02 カーネル監査有効

- **対象パス**：`auditctl -s`

- **実施操作・ルール**：`enabled 1`

- **ユーザーが示した結果**：lost 0 / backlog 0

- **認められる解釈**：この観測時点で損失がない

- **注意・未証明部分**：将来の無損失保証ではない

- **実務**：同じ監査シリアルの `SYSCALL`、`PATH`、`CWD`、`PROCTITLE` をまとめる。

- **証拠保全**：原本は改変せずコピーへハッシュを付け、JST表示と監査epochを保持する。

- **次回の確認**：異常がある場合は `sudo ausearch -k <監査キー> -i` を実行しログ損失指標と照合。

### A03 設定ファイルバックアップ

- **対象パス**：`/etc/audit/rules.d/audit.rules.bk20261102`

- **実施操作・ルール**：`cp -a`

- **ユーザーが示した結果**：バックアップファイル存在

- **認められる解釈**：変更前状態の復元起点

- **注意・未証明部分**：バックアップファイルは監査対象と別に保全

- **実務**：同じ監査シリアルの `SYSCALL`、`PATH`、`CWD`、`PROCTITLE` をまとめる。

- **証拠保全**：原本は改変せずコピーへハッシュを付け、JST表示と監査epochを保持する。

- **次回の確認**：異常がある場合は `sudo ausearch -k <監査キー> -i` を実行しログ損失指標と照合。

### A04 タスク抑止解除

- **対象パス**：`/etc/audit/rules.d/audit.rules`

- **実施操作・ルール**：`##-a task,never`

- **ユーザーが示した結果**：auditctl -lでnever,task不在

- **認められる解釈**：新規プロセスのsyscall監査が可能

- **注意・未証明部分**：単体では監視対象が定まらない

- **実務**：同じ監査シリアルの `SYSCALL`、`PATH`、`CWD`、`PROCTITLE` をまとめる。

- **証拠保全**：原本は改変せずコピーへハッシュを付け、JST表示と監査epochを保持する。

- **次回の確認**：異常がある場合は `sudo ausearch -k <監査キー> -i` を実行しログ損失指標と照合。

### A05 ファイル作成

- **対象パス**：`/var/tmp/audit-test/powercms_integrity_check.txt`

- **実施操作・ルール**：`touch`

- **ユーザーが示した結果**：161364 / openat / CREATE / success=yes

- **認められる解釈**：作成成功

- **注意・未証明部分**：PROCTITLEとPATHを同じserialで保全

- **実務**：同じ監査シリアルの `SYSCALL`、`PATH`、`CWD`、`PROCTITLE` をまとめる。

- **証拠保全**：原本は改変せずコピーへハッシュを付け、JST表示と監査epochを保持する。

- **次回の確認**：異常がある場合は `sudo ausearch -k <監査キー> -i` を実行しログ損失指標と照合。

### A06 ファイル権限変更

- **対象パス**：`/var/tmp/audit-test/powercms_integrity_check.txt`

- **実施操作・ルール**：`chmod 600`

- **ユーザーが示した結果**：161393 / fchmodat / success=yes

- **認められる解釈**：属性変更成功

- **注意・未証明部分**：対象inodeとPATHを確認

- **実務**：同じ監査シリアルの `SYSCALL`、`PATH`、`CWD`、`PROCTITLE` をまとめる。

- **証拠保全**：原本は改変せずコピーへハッシュを付け、JST表示と監査epochを保持する。

- **次回の確認**：異常がある場合は `sudo ausearch -k <監査キー> -i` を実行しログ損失指標と照合。

### A07 ファイル削除

- **対象パス**：`/var/tmp/audit-test/powercms_integrity_check.txt`

- **実施操作・ルール**：`rm`

- **ユーザーが示した結果**：161394 / unlinkat / DELETE / success=yes

- **認められる解釈**：削除成功

- **注意・未証明部分**：削除前の内容は保全されていない

- **実務**：同じ監査シリアルの `SYSCALL`、`PATH`、`CWD`、`PROCTITLE` をまとめる。

- **証拠保全**：原本は改変せずコピーへハッシュを付け、JST表示と監査epochを保持する。

- **次回の確認**：異常がある場合は `sudo ausearch -k <監査キー> -i` を実行しログ損失指標と照合。

### A08 内容書き込み

- **対象パス**：`/var/tmp/audit-test/powercms_integrity_check.txt`

- **実施操作・ルール**：`echo >`

- **ユーザーが示した結果**：独立したイベント提示なし

- **認められる解釈**：書き込み内容を証明できない

- **注意・未証明部分**：openとwriteを分ける

- **実務**：同じ監査シリアルの `SYSCALL`、`PATH`、`CWD`、`PROCTITLE` をまとめる。

- **証拠保全**：原本は改変せずコピーへハッシュを付け、JST表示と監査epochを保持する。

- **次回の確認**：異常がある場合は `sudo ausearch -k <監査キー> -i` を実行しログ損失指標と照合。

### A09 内容追記

- **対象パス**：`/var/tmp/audit-test/powercms_integrity_check.txt`

- **実施操作・ルール**：`echo >>`

- **ユーザーが示した結果**：独立したイベント提示なし

- **認められる解釈**：追記内容を証明できない

- **注意・未証明部分**：内容差分の保全が必要

- **実務**：同じ監査シリアルの `SYSCALL`、`PATH`、`CWD`、`PROCTITLE` をまとめる。

- **証拠保全**：原本は改変せずコピーへハッシュを付け、JST表示と監査epochを保持する。

- **次回の確認**：異常がある場合は `sudo ausearch -k <監査キー> -i` を実行しログ損失指標と照合。

### A10 ファイル読み取り

- **対象パス**：`/var/tmp/audit-test/read_test.txt`

- **実施操作・ルール**：`cat`

- **ユーザーが示した結果**：161730 / openat / O_RDONLY / success=yes

- **認められる解釈**：読取目的のopen成功

- **注意・未証明部分**：外部流出の証拠ではない

- **実務**：同じ監査シリアルの `SYSCALL`、`PATH`、`CWD`、`PROCTITLE` をまとめる。

- **証拠保全**：原本は改変せずコピーへハッシュを付け、JST表示と監査epochを保持する。

- **次回の確認**：異常がある場合は `sudo ausearch -k <監査キー> -i` を実行しログ損失指標と照合。

### A11 ページャー読み取り

- **対象パス**：`/var/tmp/audit-test/read_test.txt`

- **実施操作・ルール**：`less`

- **ユーザーが示した結果**：161736 / openat / O_RDONLY / success=yes

- **認められる解釈**：読取目的のopen成功

- **注意・未証明部分**：readlink EINVALと混同しない

- **実務**：同じ監査シリアルの `SYSCALL`、`PATH`、`CWD`、`PROCTITLE` をまとめる。

- **証拠保全**：原本は改変せずコピーへハッシュを付け、JST表示と監査epochを保持する。

- **次回の確認**：異常がある場合は `sudo ausearch -k <監査キー> -i` を実行しログ損失指標と照合。

### A12 ファイル形式参照

- **対象パス**：`/var/tmp/audit-test/read_test.txt`

- **実施操作・ルール**：`file -b`

- **ユーザーが示した結果**：161735 / openat / O_RDONLY

- **認められる解釈**：lessが呼んだ補助コマンド

- **注意・未証明部分**：less操作と別プロセス

- **実務**：同じ監査シリアルの `SYSCALL`、`PATH`、`CWD`、`PROCTITLE` をまとめる。

- **証拠保全**：原本は改変せずコピーへハッシュを付け、JST表示と監査epochを保持する。

- **次回の確認**：異常がある場合は `sudo ausearch -k <監査キー> -i` を実行しログ損失指標と照合。

### A13 NFS監査ルール追加

- **対象パス**：`/var/www/audit-test`

- **実施操作・ルール**：`auditctl -a`

- **ユーザーが示した結果**：161965 CONFIG_CHANGE

- **認められる解釈**：一時ルールの登録成功

- **注意・未証明部分**：再起動で消える可能性

- **実務**：同じ監査シリアルの `SYSCALL`、`PATH`、`CWD`、`PROCTITLE` をまとめる。

- **証拠保全**：原本は改変せずコピーへハッシュを付け、JST表示と監査epochを保持する。

- **次回の確認**：異常がある場合は `sudo ausearch -k <監査キー> -i` を実行しログ損失指標と照合。

### A14 NFSファイル作成

- **対象パス**：`/var/www/audit-test/nfs_test.txt`

- **実施操作・ルール**：`touch`

- **ユーザーが示した結果**：161974 / CREATE / openat / success=yes

- **認められる解釈**：NFS上の作成成功

- **注意・未証明部分**：他クライアント操作は対象外

- **実務**：同じ監査シリアルの `SYSCALL`、`PATH`、`CWD`、`PROCTITLE` をまとめる。

- **証拠保全**：原本は改変せずコピーへハッシュを付け、JST表示と監査epochを保持する。

- **次回の確認**：異常がある場合は `sudo ausearch -k <監査キー> -i` を実行しログ損失指標と照合。

### A15 NFS権限変更

- **対象パス**：`/var/www/audit-test/nfs_test.txt`

- **実施操作・ルール**：`chmod`

- **ユーザーが示した結果**：161975 / fchmodat / success=yes

- **認められる解釈**：NFS上の属性変更成功

- **注意・未証明部分**：NFSの運用条件差を考慮

- **実務**：同じ監査シリアルの `SYSCALL`、`PATH`、`CWD`、`PROCTITLE` をまとめる。

- **証拠保全**：原本は改変せずコピーへハッシュを付け、JST表示と監査epochを保持する。

- **次回の確認**：異常がある場合は `sudo ausearch -k <監査キー> -i` を実行しログ損失指標と照合。

### A16 NFSファイル削除

- **対象パス**：`/var/www/audit-test/nfs_test.txt`

- **実施操作・ルール**：`rm`

- **ユーザーが示した結果**：161976 / unlinkat / DELETE / success=yes

- **認められる解釈**：NFS上の削除成功

- **注意・未証明部分**：共有先からの別経路操作は不明

- **実務**：同じ監査シリアルの `SYSCALL`、`PATH`、`CWD`、`PROCTITLE` をまとめる。

- **証拠保全**：原本は改変せずコピーへハッシュを付け、JST表示と監査epochを保持する。

- **次回の確認**：異常がある場合は `sudo ausearch -k <監査キー> -i` を実行しログ損失指標と照合。

### A17 本番OS設定監査

- **対象パス**：`/etc`

- **実施操作・ルール**：`perm=wa`

- **ユーザーが示した結果**：prod_config_change

- **認められる解釈**：ルール登録成功

- **注意・未証明部分**：改ざんイベントが実際に出るか別検証

- **実務**：同じ監査シリアルの `SYSCALL`、`PATH`、`CWD`、`PROCTITLE` をまとめる。

- **証拠保全**：原本は改変せずコピーへハッシュを付け、JST表示と監査epochを保持する。

- **次回の確認**：異常がある場合は `sudo ausearch -k <監査キー> -i` を実行しログ損失指標と照合。

### A18 管理スクリプト監査

- **対象パス**：`/usr/local/bin`

- **実施操作・ルール**：`perm=wa`

- **ユーザーが示した結果**：prod_script_change

- **認められる解釈**：ルール登録成功

- **注意・未証明部分**：ログ出力自体で内容保全はしない

- **実務**：同じ監査シリアルの `SYSCALL`、`PATH`、`CWD`、`PROCTITLE` をまとめる。

- **証拠保全**：原本は改変せずコピーへハッシュを付け、JST表示と監査epochを保持する。

- **次回の確認**：異常がある場合は `sudo ausearch -k <監査キー> -i` を実行しログ損失指標と照合。

### A19 管理スクリプト監査

- **対象パス**：`/usr/local/sbin`

- **実施操作・ルール**：`perm=wa`

- **ユーザーが示した結果**：prod_script_change

- **認められる解釈**：ルール登録成功

- **注意・未証明部分**：Pythonログエクスポートも対象に含む

- **実務**：同じ監査シリアルの `SYSCALL`、`PATH`、`CWD`、`PROCTITLE` をまとめる。

- **証拠保全**：原本は改変せずコピーへハッシュを付け、JST表示と監査epochを保持する。

- **次回の確認**：異常がある場合は `sudo ausearch -k <監査キー> -i` を実行しログ損失指標と照合。

### A20 本番PowerCMS監査

- **対象パス**：`/var/www/html`

- **実施操作・ルール**：`perm=wa`

- **ユーザーが示した結果**：prod_powercms_change / 162195 CONFIG_CHANGE

- **認められる解釈**：ルール登録成功

- **注意・未証明部分**：本番ファイル操作の実測は未確認

- **実務**：同じ監査シリアルの `SYSCALL`、`PATH`、`CWD`、`PROCTITLE` をまとめる。

- **証拠保全**：原本は改変せずコピーへハッシュを付け、JST表示と監査epochを保持する。

- **次回の確認**：異常がある場合は `sudo ausearch -k <監査キー> -i` を実行しログ損失指標と照合。

## 14. auditdの質問応答・誤解防止FAQ

### Q01. なぜsudoコマンドの証拠だけでは不足か

**回答**：USER_CMDはsudoセッションの管理イベント。SYSCALL success=yes とPATHを見なければ対象ファイル変更の成功は立証できない。

**区別すべき情報**：ユーザー提示の監査レコードと、推測・一般的動作を同じ強度の証拠として扱わない。

**次チャット対応**：新しいログがあれば、対象日時、key、msg=audit(...:serial)、success、PATHを確認する。

### Q02. なぜcatが初めはログにないか

**回答**：初期のテスト用ルールはperm=waで読み取りrを指定していなかったから。

**区別すべき情報**：ユーザー提示の監査レコードと、推測・一般的動作を同じ強度の証拠として扱わない。

**次チャット対応**：新しいログがあれば、対象日時、key、msg=audit(...:serial)、success、PATHを確認する。

### Q03. catのログが出たら流出確定か

**回答**：openat O_RDONLY の成功は開いた事実。内容取得量や外部送信を単独で証明しない。

**区別すべき情報**：ユーザー提示の監査レコードと、推測・一般的動作を同じ強度の証拠として扱わない。

**次チャット対応**：新しいログがあれば、対象日時、key、msg=audit(...:serial)、success、PATHを確認する。

### Q04. lessのreadlink失敗は異常か

**回答**：今回のlessはreadlink EINVALが複数出たが、別イベントのopenat O_RDONLY success=yesでファイルを開けている。

**区別すべき情報**：ユーザー提示の監査レコードと、推測・一般的動作を同じ強度の証拠として扱わない。

**次チャット対応**：新しいログがあれば、対象日時、key、msg=audit(...:serial)、success、PATHを確認する。

### Q05. cdを監査すべきか

**回答**：目的が改ざんと機密ファイル読取なら通常は必須でない。cdはシェル組込みで一般にファイル操作と一致しない。

**区別すべき情報**：ユーザー提示の監査レコードと、推測・一般的動作を同じ強度の証拠として扱わない。

**次チャット対応**：新しいログがあれば、対象日時、key、msg=audit(...:serial)、success、PATHを確認する。

### Q06. echoによる追記がログにないのはなぜか

**回答**：auditdのperm監査は個々のwrite()を必ず記録するものではなく、fdの継承や既存プロセス等も要考慮。内容差分の代替にはならない。

**区別すべき情報**：ユーザー提示の監査レコードと、推測・一般的動作を同じ強度の証拠として扱わない。

**次チャット対応**：新しいログがあれば、対象日時、key、msg=audit(...:serial)、success、PATHを確認する。

### Q07. auditctl -lのNo rulesは何か

**回答**：実効ルールがない状態で出る。augenrules --loadのNo rules文字列だけで成功失敗は判断せずauditctl -lを再実行する。

**区別すべき情報**：ユーザー提示の監査レコードと、推測・一般的動作を同じ強度の証拠として扱わない。

**次チャット対応**：新しいログがあれば、対象日時、key、msg=audit(...:serial)、success、PATHを確認する。

### Q08. 再ログインが必要か

**回答**：task,neverが効いた時代の既存プロセスの状態が残る可能性があるため、新規ログインセッションでテストする。

**区別すべき情報**：ユーザー提示の監査レコードと、推測・一般的動作を同じ強度の証拠として扱わない。

**次チャット対応**：新しいログがあれば、対象日時、key、msg=audit(...:serial)、success、PATHを確認する。

### Q09. EFSを監査できるか

**回答**：NFSv4.1の当該EC2経由でテスト成功。ただしEFS製品かどうかはこの出力だけでは不明、他クライアント操作は不可視。

**区別すべき情報**：ユーザー提示の監査レコードと、推測・一般的動作を同じ強度の証拠として扱わない。

**次チャット対応**：新しいログがあれば、対象日時、key、msg=audit(...:serial)、success、PATHを確認する。

### Q10. ファイル変更前の内容は残るか

**回答**：auditdは原則データ差分や内容を保存しない。バックアップ、ハッシュ、スナップショットが必要。

**区別すべき情報**：ユーザー提示の監査レコードと、推測・一般的動作を同じ強度の証拠として扱わない。

**次チャット対応**：新しいログがあれば、対象日時、key、msg=audit(...:serial)、success、PATHを確認する。

### Q11. 監査ユーザーがec2-userなら実人物も特定か

**回答**：ログイン主体のUID/AUIDと本人識別は別。SSM/SSH/IdPの認証記録、接続元を追加で照合。

**区別すべき情報**：ユーザー提示の監査レコードと、推測・一般的動作を同じ強度の証拠として扱わない。

**次チャット対応**：新しいログがあれば、対象日時、key、msg=audit(...:serial)、success、PATHを確認する。

### Q12. rootのauidがunsetは攻撃か

**回答**：systemdやサービスではunsetは一般的。単独で悪性とは評価しない。

**区別すべき情報**：ユーザー提示の監査レコードと、推測・一般的動作を同じ強度の証拠として扱わない。

**次チャット対応**：新しいログがあれば、対象日時、key、msg=audit(...:serial)、success、PATHを確認する。

### Q13. permissive=1のAVCは攻撃か

**回答**：SELinuxの許可されないアクセスが検出されたが強制ブロックではない。通常の設定不整合もあり得る。

**区別すべき情報**：ユーザー提示の監査レコードと、推測・一般的動作を同じ強度の証拠として扱わない。

**次チャット対応**：新しいログがあれば、対象日時、key、msg=audit(...:serial)、success、PATHを確認する。

### Q14. loss=0なら欠落なしを保証するか

**回答**：カーネル監査キューの観測指標であり、Agent以降、CloudWatch/S3の完全性は保証しない。

**区別すべき情報**：ユーザー提示の監査レコードと、推測・一般的動作を同じ強度の証拠として扱わない。

**次チャット対応**：新しいログがあれば、対象日時、key、msg=audit(...:serial)、success、PATHを確認する。

### Q15. backlog_limit=64は十分か

**回答**：現時点で設定値が64と分かるだけ。負荷下でのlostの推移を計測せず安全性は断定できない。

**区別すべき情報**：ユーザー提示の監査レコードと、推測・一般的動作を同じ強度の証拠として扱わない。

**次チャット対応**：新しいログがあれば、対象日時、key、msg=audit(...:serial)、success、PATHを確認する。

### Q16. 全サーバーrwaにすべきか

**回答**：本番ではログ量、性能、請求、監査遅延のリスクがあるため対象を絞る。

**区別すべき情報**：ユーザー提示の監査レコードと、推測・一般的動作を同じ強度の証拠として扱わない。

**次チャット対応**：新しいログがあれば、対象日時、key、msg=audit(...:serial)、success、PATHを確認する。

### Q17. 実行ファイルの改ざんが記録されるか

**回答**：指定したパスでの操作は監査対象。ただしシェル・監査経路・別マウントなどの適用境界がある。

**区別すべき情報**：ユーザー提示の監査レコードと、推測・一般的動作を同じ強度の証拠として扱わない。

**次チャット対応**：新しいログがあれば、対象日時、key、msg=audit(...:serial)、success、PATHを確認する。

### Q18. 機密情報の場所が不明なとき

**回答**：設定全体を表示して秘密値を貼らない。パスと権限を確認し対象ファイルを限定してrwa監査する。

**区別すべき情報**：ユーザー提示の監査レコードと、推測・一般的動作を同じ強度の証拠として扱わない。

**次チャット対応**：新しいログがあれば、対象日時、key、msg=audit(...:serial)、success、PATHを確認する。

### Q19. セッションとプロセスの違い

**回答**：AUID/SESはログイン起点、UID/EUIDは実行権限、PID/PPID/comm/exeは実行実体。

**区別すべき情報**：ユーザー提示の監査レコードと、推測・一般的動作を同じ強度の証拠として扱わない。

**次チャット対応**：新しいログがあれば、対象日時、key、msg=audit(...:serial)、success、PATHを確認する。

### Q20. CloudWatch PATHの意味

**回答**：audit SYSCALL単体には操作ファイル名が入らない場合があり、PATHが同じシリアルで必要。

**区別すべき情報**：ユーザー提示の監査レコードと、推測・一般的動作を同じ強度の証拠として扱わない。

**次チャット対応**：新しいログがあれば、対象日時、key、msg=audit(...:serial)、success、PATHを確認する。

### Q21. セキュリティ事件が起きたらまず何をする

**回答**：ログローテーション/削除設定で失われる前に原本保全し、可能ならスナップショット作成を関係者と調整。

**区別すべき情報**：ユーザー提示の監査レコードと、推測・一般的動作を同じ強度の証拠として扱わない。

**次チャット対応**：新しいログがあれば、対象日時、key、msg=audit(...:serial)、success、PATHを確認する。

## 15. 最新課題：S3ライフサイクル3日なのに残る問題（調査マニュアル）

2026-11-04の画像が唯一の最新設定根拠。ルール名は400daysだが設定は3日。バージョニングと実オブジェクト日時は未確認。

### S3-01 対象バケット名

- 値：`powercms-log-archive-logs-mchem-infratec-s3`

- 根拠の状態：既知

- 注意：設定画面のバケット名と一致するか。

- 次の調査は参照系APIを優先し、削除・保持期限変更を伴う操作をしない。

### S3-02 リージョン

- 値：`ap-northeast-1`

- 根拠の状態：既知のプロジェクトリージョン

- 注意：対象バケット実リージョンはAPI確認も可。

- 次の調査は参照系APIを優先し、削除・保持期限変更を伴う操作をしない。

### S3-03 ルール名

- 値：`delete-logs-after-400days`

- 根拠の状態：既知

- 注意：名称は実削除日数を決めない。

- 次の調査は参照系APIを優先し、削除・保持期限変更を伴う操作をしない。

### S3-04 ルールのステータス

- 値：`Enabled`

- 根拠の状態：画像で確認

- 注意：必要ならCLIで再確認する。

- 次の調査は参照系APIを優先し、削除・保持期限変更を伴う操作をしない。

### S3-05 対象prefix

- 値：`-（バケット全体）`

- 根拠の状態：画像で確認

- 注意：他のルールによる競合がないか別途確認。

- 次の調査は参照系APIを優先し、削除・保持期限変更を伴う操作をしない。

### S3-06 現行オブジェクト有効期限

- 値：`3日`

- 根拠の状態：画像で確認

- 注意：UTC境界と非同期実削除を考慮する。

- 次の調査は参照系APIを優先し、削除・保持期限変更を伴う操作をしない。

### S3-07 非現行バージョン期限

- 値：`未設定`

- 根拠の状態：画像で確認

- 注意：Versioning有効の場合は旧版が残る。

- 次の調査は参照系APIを優先し、削除・保持期限変更を伴う操作をしない。

### S3-08 Versioning

- 値：`未確認`

- 根拠の状態：未確認

- 注意：SuspendedとEnabledも区別する。

- 次の調査は参照系APIを優先し、削除・保持期限変更を伴う操作をしない。

### S3-09 Object Lock

- 値：`未確認`

- 根拠の状態：未確認

- 注意：期限経過後もオブジェクトを保持し得る。

- 次の調査は参照系APIを優先し、削除・保持期限変更を伴う操作をしない。

### S3-10 該当オブジェクトキー

- 値：`未提示`

- 根拠の状態：未確認

- 注意：ファイル名日付だけでなくLastModifiedを取得する。

- 次の調査は参照系APIを優先し、削除・保持期限変更を伴う操作をしない。

### S3-11 該当オブジェクトLastModified

- 値：`未提示`

- 根拠の状態：未確認

- 注意：S3 UTC基準時刻を記録する。

- 次の調査は参照系APIを優先し、削除・保持期限変更を伴う操作をしない。

### S3-12 削除マーカー

- 値：`未確認`

- 根拠の状態：未確認

- 注意：バージョニング時の非表示理由となる。

- 次の調査は参照系APIを優先し、削除・保持期限変更を伴う操作をしない。

### S3-13 非現行版サイズ

- 値：`未確認`

- 根拠の状態：未確認

- 注意：現行版のみと全バージョンを分けて確認。

- 次の調査は参照系APIを優先し、削除・保持期限変更を伴う操作をしない。

### S3-14 Firehoseバッファリング

- 値：`値未提示`

- 根拠の状態：未確認

- 注意：S3オブジェクト作成までの遅延を確認する。

- 次の調査は参照系APIを優先し、削除・保持期限変更を伴う操作をしない。

### S3-15 オブジェクト保持期間

- 値：`テストで短期`

- 根拠の状態：会話中のユーザー意向

- 注意：最終計画は400日、勝手に変更しない。

- 次の調査は参照系APIを優先し、削除・保持期限変更を伴う操作をしない。

### S3-16 CloudWatch保持期間

- 値：`会話上1日テスト継続`

- 根拠の状態：会話中のユーザー意向

- 注意：S3画面の3日とは別の設定。

- 次の調査は参照系APIを優先し、削除・保持期限変更を伴う操作をしない。

### S3-17 請求上の課金

- 値：`未確認`

- 根拠の状態：未確認

- 注意：削除期限の適格性と実物の削除表示は別。

- 次の調査は参照系APIを優先し、削除・保持期限変更を伴う操作をしない。

### S3-18 S3 Inventory

- 値：`未確認`

- 根拠の状態：未確認

- 注意：必要なら現行・非現行版の棚卸を検討。

- 次の調査は参照系APIを優先し、削除・保持期限変更を伴う操作をしない。

### S3-19 S3サーバーアクセスログ

- 値：`未確認`

- 根拠の状態：未確認

- 注意：削除主体や操作確認にはCloudTrail Data Events等。

- 次の調査は参照系APIを優先し、削除・保持期限変更を伴う操作をしない。

### S3-20 現在の手動削除実施

- 値：`不要`

- 根拠の状態：安全方針

- 注意：調査目的でログ原本を消さない。

- 次の調査は参照系APIを優先し、削除・保持期限変更を伴う操作をしない。

### S3-CLI-01 バージョニング状態 [READ ONLY]

- 目的：StatusがEnabled/Suspended/未設定かを確認

```bash
aws s3api get-bucket-versioning --bucket powercms-log-archive-logs-mchem-infratec-s3
```

- 実行前：AWS CLIプロファイルのアカウントと対象バケットを確認する。

- 実行後：個人情報を含むキーやメタデータは共有前に必要な範囲でマスクする。

- 判定基準：現時点のAPI応答を一次事実として記録し、画面と差があれば差分を調査する。

### S3-CLI-02 ライフサイクルのJSON [READ ONLY]

- 目的：Expiration Daysが3か、複数ルールやフィルターがあるか

```bash
aws s3api get-bucket-lifecycle-configuration --bucket powercms-log-archive-logs-mchem-infratec-s3
```

- 実行前：AWS CLIプロファイルのアカウントと対象バケットを確認する。

- 実行後：個人情報を含むキーやメタデータは共有前に必要な範囲でマスクする。

- 判定基準：現時点のAPI応答を一次事実として記録し、画面と差があれば差分を調査する。

### S3-CLI-03 対象バケット位置 [READ ONLY]

- 目的：指定リージョンの取り違えを防止

```bash
aws s3api get-bucket-location --bucket powercms-log-archive-logs-mchem-infratec-s3
```

- 実行前：AWS CLIプロファイルのアカウントと対象バケットを確認する。

- 実行後：個人情報を含むキーやメタデータは共有前に必要な範囲でマスクする。

- 判定基準：現時点のAPI応答を一次事実として記録し、画面と差があれば差分を調査する。

### S3-CLI-04 対象prefixの列挙 [READ ONLY]

- 目的：例示prefixのキーが見つかるか。実構成に応じてprefix訂正

```bash
aws s3api list-objects-v2 --bucket powercms-log-archive-logs-mchem-infratec-s3 --prefix cms-rebuild/prod/ --max-items 20
```

- 実行前：AWS CLIプロファイルのアカウントと対象バケットを確認する。

- 実行後：個人情報を含むキーやメタデータは共有前に必要な範囲でマスクする。

- 判定基準：現時点のAPI応答を一次事実として記録し、画面と差があれば差分を調査する。

### S3-CLI-05 対象キーの詳細 [READ ONLY]

- 目的：LastModified、Expiration、VersionId等を確認

```bash
aws s3api head-object --bucket powercms-log-archive-logs-mchem-infratec-s3 --key 'REPLACE_WITH_REAL_KEY'
```

- 実行前：AWS CLIプロファイルのアカウントと対象バケットを確認する。

- 実行後：個人情報を含むキーやメタデータは共有前に必要な範囲でマスクする。

- 判定基準：現時点のAPI応答を一次事実として記録し、画面と差があれば差分を調査する。

### S3-CLI-06 旧版とマーカー [READ ONLY]

- 目的：非現行バージョンとDeleteMarkersを照合

```bash
aws s3api list-object-versions --bucket powercms-log-archive-logs-mchem-infratec-s3 --prefix 'REPLACE_WITH_REAL_PREFIX' --max-items 50
```

- 実行前：AWS CLIプロファイルのアカウントと対象バケットを確認する。

- 実行後：個人情報を含むキーやメタデータは共有前に必要な範囲でマスクする。

- 判定基準：現時点のAPI応答を一次事実として記録し、画面と差があれば差分を調査する。

### S3-CLI-07 バケットObject Lock [READ ONLY]

- 目的：未設定時はエラーも想定し、出力を保存

```bash
aws s3api get-object-lock-configuration --bucket powercms-log-archive-logs-mchem-infratec-s3
```

- 実行前：AWS CLIプロファイルのアカウントと対象バケットを確認する。

- 実行後：個人情報を含むキーやメタデータは共有前に必要な範囲でマスクする。

- 判定基準：現時点のAPI応答を一次事実として記録し、画面と差があれば差分を調査する。

### S3-CLI-08 オブジェクト保持 [READ ONLY]

- 目的：保持日時とモードが付与されているか

```bash
aws s3api get-object-retention --bucket powercms-log-archive-logs-mchem-infratec-s3 --key 'REPLACE_WITH_REAL_KEY'
```

- 実行前：AWS CLIプロファイルのアカウントと対象バケットを確認する。

- 実行後：個人情報を含むキーやメタデータは共有前に必要な範囲でマスクする。

- 判定基準：現時点のAPI応答を一次事実として記録し、画面と差があれば差分を調査する。

### S3-CLI-09 オブジェクト法的ホールド [READ ONLY]

- 目的：LegalHoldの有無を確認

```bash
aws s3api get-object-legal-hold --bucket powercms-log-archive-logs-mchem-infratec-s3 --key 'REPLACE_WITH_REAL_KEY'
```

- 実行前：AWS CLIプロファイルのアカウントと対象バケットを確認する。

- 実行後：個人情報を含むキーやメタデータは共有前に必要な範囲でマスクする。

- 判定基準：現時点のAPI応答を一次事実として記録し、画面と差があれば差分を調査する。

### S3-CLI-10 現行オブジェクトの日時一覧 [READ ONLY]

- 目的：キーの埋め込み日付でなくLastModifiedを読む

```bash
aws s3api list-objects-v2 --bucket powercms-log-archive-logs-mchem-infratec-s3 --prefix audit-log/prod/ --query 'Contents[].[Key,LastModified,Size]' --output table
```

- 実行前：AWS CLIプロファイルのアカウントと対象バケットを確認する。

- 実行後：個人情報を含むキーやメタデータは共有前に必要な範囲でマスクする。

- 判定基準：現時点のAPI応答を一次事実として記録し、画面と差があれば差分を調査する。

### ケースA：名前は400days、実設定3days

- 見るべきもの：ルールJSON、S3オブジェクト LastModified、version ID、実際の表示状態。

- 説明：名前と実値の不一致は設計上誤読の原因。実値3日として計算する。

- 推奨順：まず参照系APIで確かめ、削除やルール編集は結論が出るまで行わない。

- 引き継ぎ先の報告方法：「観測」「解釈」「残る別解」の3段で日本語報告。

### ケースB：キーに古い年月日がある

- 見るべきもの：ルールJSON、S3オブジェクト LastModified、version ID、実際の表示状態。

- 説明：キーの日付は保存日時と一致するとは限らない。S3 LastModifiedを見る。

- 推奨順：まず参照系APIで確かめ、削除やルール編集は結論が出るまで行わない。

- 引き継ぎ先の報告方法：「観測」「解釈」「残る別解」の3段で日本語報告。

### ケースC：LastModifiedから暦3日後も存在

- 見るべきもの：ルールJSON、S3オブジェクト LastModified、version ID、実際の表示状態。

- 説明：UTC翌日00:00切り上げおよび実削除の非同期遅延を考慮。

- 推奨順：まず参照系APIで確かめ、削除やルール編集は結論が出るまで行わない。

- 引き継ぎ先の報告方法：「観測」「解釈」「残る別解」の3段で日本語報告。

### ケースD：Versioning有効

- 見るべきもの：ルールJSON、S3オブジェクト LastModified、version ID、実際の表示状態。

- 説明：期限切れで削除マーカーが追加されることがあり、実体の旧版は残り得る。

- 推奨順：まず参照系APIで確かめ、削除やルール編集は結論が出るまで行わない。

- 引き継ぎ先の報告方法：「観測」「解釈」「残る別解」の3段で日本語報告。

### ケースE：Versioning無効

- 見るべきもの：ルールJSON、S3オブジェクト LastModified、version ID、実際の表示状態。

- 説明：Expirationがオブジェクト削除を行うが、実行遅延や他の条件を確認する。

- 推奨順：まず参照系APIで確かめ、削除やルール編集は結論が出るまで行わない。

- 引き継ぎ先の報告方法：「観測」「解釈」「残る別解」の3段で日本語報告。

### ケースF：Object Lock保持中

- 見るべきもの：ルールJSON、S3オブジェクト LastModified、version ID、実際の表示状態。

- 説明：保持期限内の削除制限により残ることがある。

- 推奨順：まず参照系APIで確かめ、削除やルール編集は結論が出るまで行わない。

- 引き継ぎ先の報告方法：「観測」「解釈」「残る別解」の3段で日本語報告。

### ケースG：Legal Hold有効

- 見るべきもの：ルールJSON、S3オブジェクト LastModified、version ID、実際の表示状態。

- 説明：期限経過だけで除去できない場合がある。

- 推奨順：まず参照系APIで確かめ、削除やルール編集は結論が出るまで行わない。

- 引き継ぎ先の報告方法：「観測」「解釈」「残る別解」の3段で日本語報告。

### ケースH：同じキーが上書きされている

- 見るべきもの：ルールJSON、S3オブジェクト LastModified、version ID、実際の表示状態。

- 説明：LastModifiedが更新され、有効期限基準も変わり得る。

- 推奨順：まず参照系APIで確かめ、削除やルール編集は結論が出るまで行わない。

- 引き継ぎ先の報告方法：「観測」「解釈」「残る別解」の3段で日本語報告。

### ケースI：複数ルールが対象

- 見るべきもの：ルールJSON、S3オブジェクト LastModified、version ID、実際の表示状態。

- 説明：フィルター、優先度ではなく作用する遷移・削除操作を実際のJSONで評価する。

- 推奨順：まず参照系APIで確かめ、削除やルール編集は結論が出るまで行わない。

- 引き継ぎ先の報告方法：「観測」「解釈」「残る別解」の3段で日本語報告。

### ケースJ：同一prefixに古いオブジェクト

- 見るべきもの：ルールJSON、S3オブジェクト LastModified、version ID、実際の表示状態。

- 説明：オブジェクト単位でバージョン・Lock・LastModifiedを観測する。

- 推奨順：まず参照系APIで確かめ、削除やルール編集は結論が出るまで行わない。

- 引き継ぎ先の報告方法：「観測」「解釈」「残る別解」の3段で日本語報告。

### ケースK：オブジェクトが一覧にない

- 見るべきもの：ルールJSON、S3オブジェクト LastModified、version ID、実際の表示状態。

- 説明：プレフィックス違い、削除済み、権限不足、表示の更新を区別する。

- 推奨順：まず参照系APIで確かめ、削除やルール編集は結論が出るまで行わない。

- 引き継ぎ先の報告方法：「観測」「解釈」「残る別解」の3段で日本語報告。

### ケースL：403 AccessDenied

- 見るべきもの：ルールJSON、S3オブジェクト LastModified、version ID、実際の表示状態。

- 説明：IAMが参照を拒否している可能性。不可視を削除済みと誤認しない。

- 推奨順：まず参照系APIで確かめ、削除やルール編集は結論が出るまで行わない。

- 引き継ぎ先の報告方法：「観測」「解釈」「残る別解」の3段で日本語報告。

### ケースM：S3コンソールの旧版を表示

- 見るべきもの：ルールJSON、S3オブジェクト LastModified、version ID、実際の表示状態。

- 説明：現行版リストとは異なる存在が表示される可能性。

- 推奨順：まず参照系APIで確かめ、削除やルール編集は結論が出るまで行わない。

- 引き継ぎ先の報告方法：「観測」「解釈」「残る別解」の3段で日本語報告。

### ケースN：Firehose書込みが遅延

- 見るべきもの：ルールJSON、S3オブジェクト LastModified、version ID、実際の表示状態。

- 説明：CloudWatchイベント時刻とS3最終更新時刻を別々に確認。

- 推奨順：まず参照系APIで確かめ、削除やルール編集は結論が出るまで行わない。

- 引き継ぎ先の報告方法：「観測」「解釈」「残る別解」の3段で日本語報告。

### ケースO：テスト期間1日から3日に変化

- 見るべきもの：ルールJSON、S3オブジェクト LastModified、version ID、実際の表示状態。

- 説明：発言と画面の観測時点が異なる。最新画面の3日を優先。

- 推奨順：まず参照系APIで確かめ、削除やルール編集は結論が出るまで行わない。

- 引き継ぎ先の報告方法：「観測」「解釈」「残る別解」の3段で日本語報告。

## 16. PowerCMS再構築ログの完全性を確かめるための手順

再構築ログについては起動・ローカル出力・CloudWatch着信まで確認済み。以下はDB件数突合や欠落検知の未実施の手順。

### RB-01 毎分timer

- 参照操作：`systemctl list-timers --all`

- 確認観点：次回時刻と前回時刻を確認する。

- 成否判断：該当ログの実件数・実時刻を証拠とし、サービス終了コードのみで完全収集と断定しない。

- 注意：DB接続情報やSQL出力には秘密情報が含まれる場合があるため、共有前に必要なマスクをする。

### RB-02 timer有効化

- 参照操作：`systemctl is-enabled prod_mchem_infratec_log_export.timer`

- 確認観点：enabledか確認。

- 成否判断：該当ログの実件数・実時刻を証拠とし、サービス終了コードのみで完全収集と断定しない。

- 注意：DB接続情報やSQL出力には秘密情報が含まれる場合があるため、共有前に必要なマスクをする。

### RB-03 timer稼働

- 参照操作：`systemctl is-active prod_mchem_infratec_log_export.timer`

- 確認観点：activeか確認。

- 成否判断：該当ログの実件数・実時刻を証拠とし、サービス終了コードのみで完全収集と断定しない。

- 注意：DB接続情報やSQL出力には秘密情報が含まれる場合があるため、共有前に必要なマスクをする。

### RB-04 service本体

- 参照操作：`systemctl cat prod_mchem_infratec_log_export.service`

- 確認観点：ExecStartが2本あるか確認。

- 成否判断：該当ログの実件数・実時刻を証拠とし、サービス終了コードのみで完全収集と断定しない。

- 注意：DB接続情報やSQL出力には秘密情報が含まれる場合があるため、共有前に必要なマスクをする。

### RB-05 timer本体

- 参照操作：`systemctl cat prod_mchem_infratec_log_export.timer`

- 確認観点：正確なスケジュール表現を確認。

- 成否判断：該当ログの実件数・実時刻を証拠とし、サービス終了コードのみで完全収集と断定しない。

- 注意：DB接続情報やSQL出力には秘密情報が含まれる場合があるため、共有前に必要なマスクをする。

### RB-06 直近履歴

- 参照操作：`journalctl -u prod_mchem_infratec_log_export.service --since "30 min ago" --no-pager`

- 確認観点：毎分の成功・失敗を確認。

- 成否判断：該当ログの実件数・実時刻を証拠とし、サービス終了コードのみで完全収集と断定しない。

- 注意：DB接続情報やSQL出力には秘密情報が含まれる場合があるため、共有前に必要なマスクをする。

### RB-07 Python構文

- 参照操作：`python3 -m py_compile /usr/local/sbin/prod_mchem_infratec_export_rebuild.py`

- 確認観点：これはpyc生成の可能性があるため本番では変更承認後。

- 成否判断：該当ログの実件数・実時刻を証拠とし、サービス終了コードのみで完全収集と断定しない。

- 注意：DB接続情報やSQL出力には秘密情報が含まれる場合があるため、共有前に必要なマスクをする。

### RB-08 スクリプトの読取部

- 参照操作：`grep -n "FROM mt_rebuildlog" /usr/local/sbin/prod_mchem_infratec_export_rebuild.py`

- 確認観点：SELECT元を確認。

- 成否判断：該当ログの実件数・実時刻を証拠とし、サービス終了コードのみで完全収集と断定しない。

- 注意：DB接続情報やSQL出力には秘密情報が含まれる場合があるため、共有前に必要なマスクをする。

### RB-09 状態ファイルパス

- 参照操作：`grep -n "STATE" /usr/local/sbin/prod_mchem_infratec_export_rebuild.py`

- 確認観点：保存場所を確認。

- 成否判断：該当ログの実件数・実時刻を証拠とし、サービス終了コードのみで完全収集と断定しない。

- 注意：DB接続情報やSQL出力には秘密情報が含まれる場合があるため、共有前に必要なマスクをする。

### RB-10 状態ファイルの内容

- 参照操作：`sudo cat /var/lib/powercms-log-export/prod_mchem_infratec_rebuild.lastid`

- 確認観点：最後に処理したIDを確認。

- 成否判断：該当ログの実件数・実時刻を証拠とし、サービス終了コードのみで完全収集と断定しない。

- 注意：DB接続情報やSQL出力には秘密情報が含まれる場合があるため、共有前に必要なマスクをする。

### RB-11 ログファイル行数

- 参照操作：`sudo wc -l /var/log/powercms/prod_mchem_infratec_rebuild.log`

- 確認観点：複数JSON行を概数把握。

- 成否判断：該当ログの実件数・実時刻を証拠とし、サービス終了コードのみで完全収集と断定しない。

- 注意：DB接続情報やSQL出力には秘密情報が含まれる場合があるため、共有前に必要なマスクをする。

### RB-12 ログファイル末尾

- 参照操作：`sudo tail -n 5 /var/log/powercms/prod_mchem_infratec_rebuild.log`

- 確認観点：出力最新レコードを確認。

- 成否判断：該当ログの実件数・実時刻を証拠とし、サービス終了コードのみで完全収集と断定しない。

- 注意：DB接続情報やSQL出力には秘密情報が含まれる場合があるため、共有前に必要なマスクをする。

### RB-13 JSONの形式

- 参照操作：`1行1 JSONを複数行検査`

- 確認観点：不正行や破損を調べる。

- 成否判断：該当ログの実件数・実時刻を証拠とし、サービス終了コードのみで完全収集と断定しない。

- 注意：DB接続情報やSQL出力には秘密情報が含まれる場合があるため、共有前に必要なマスクをする。

### RB-14 重複ID

- 参照操作：`JSONL中rebuildlog_idの重複を比較`

- 確認観点：再試行が重複を作るか調べる。

- 成否判断：該当ログの実件数・実時刻を証拠とし、サービス終了コードのみで完全収集と断定しない。

- 注意：DB接続情報やSQL出力には秘密情報が含まれる場合があるため、共有前に必要なマスクをする。

### RB-15 欠番ID

- 参照操作：`DB ID配列とログID配列を突合`

- 確認観点：欠番=欠落とは限らず、DBの採番仕様に依存。

- 成否判断：該当ログの実件数・実時刻を証拠とし、サービス終了コードのみで完全収集と断定しない。

- 注意：DB接続情報やSQL出力には秘密情報が含まれる場合があるため、共有前に必要なマスクをする。

### RB-16 DB最大ID

- 参照操作：`読み取り専用SQLでMAX(rebuildlog_id)を確認`

- 確認観点：状態ファイルより大きいIDがあるか調べる。

- 成否判断：該当ログの実件数・実時刻を証拠とし、サービス終了コードのみで完全収集と断定しない。

- 注意：DB接続情報やSQL出力には秘密情報が含まれる場合があるため、共有前に必要なマスクをする。

### RB-17 DB新規レコード

- 参照操作：`トランザクション完了後の新規レコードを確認`

- 確認観点：未コミット状態を扱わない。

- 成否判断：該当ログの実件数・実時刻を証拠とし、サービス終了コードのみで完全収集と断定しない。

- 注意：DB接続情報やSQL出力には秘密情報が含まれる場合があるため、共有前に必要なマスクをする。

### RB-18 CloudWatch最大ID

- 参照操作：`Logs InsightsでJSONフィールドを抽出`

- 確認観点：DBと転送済みイベントを対照。

- 成否判断：該当ログの実件数・実時刻を証拠とし、サービス終了コードのみで完全収集と断定しない。

- 注意：DB接続情報やSQL出力には秘密情報が含まれる場合があるため、共有前に必要なマスクをする。

### RB-19 CloudWatch ingestion

- 参照操作：`@timestampと@ingestionTime`

- 確認観点：送信遅延の分離。

- 成否判断：該当ログの実件数・実時刻を証拠とし、サービス終了コードのみで完全収集と断定しない。

- 注意：DB接続情報やSQL出力には秘密情報が含まれる場合があるため、共有前に必要なマスクをする。

### RB-20 CloudWatch stream

- 参照操作：`i-0bd1b63ca374f1f78`

- 確認観点：対象インスタンスを確認。

- 成否判断：該当ログの実件数・実時刻を証拠とし、サービス終了コードのみで完全収集と断定しない。

- 注意：DB接続情報やSQL出力には秘密情報が含まれる場合があるため、共有前に必要なマスクをする。

### RB-21 S3再構築prefix

- 参照操作：`cms-rebuild/prod/`

- 確認観点：Firehose保存オブジェクトの有無を確認。

- 成否判断：該当ログの実件数・実時刻を証拠とし、サービス終了コードのみで完全収集と断定しない。

- 注意：DB接続情報やSQL出力には秘密情報が含まれる場合があるため、共有前に必要なマスクをする。

### RB-22 時刻の異なるサンプル

- 参照操作：`2026-11-04 11:57/12:10`

- 確認観点：画面ではCloudWatch着信確認済み。

- 成否判断：該当ログの実件数・実時刻を証拠とし、サービス終了コードのみで完全収集と断定しない。

- 注意：DB接続情報やSQL出力には秘密情報が含まれる場合があるため、共有前に必要なマスクをする。

### RB-23 障害復旧

- 参照操作：`stateをむやみに戻さない`

- 確認観点：重複・欠落を避ける。

- 成否判断：該当ログの実件数・実時刻を証拠とし、サービス終了コードのみで完全収集と断定しない。

- 注意：DB接続情報やSQL出力には秘密情報が含まれる場合があるため、共有前に必要なマスクをする。

### RB-24 サービス成功コード

- 参照操作：`0/SUCCESS`

- 確認観点：エクスポータ処理内部の整合性まで保証しない。

- 成否判断：該当ログの実件数・実時刻を証拠とし、サービス終了コードのみで完全収集と断定しない。

- 注意：DB接続情報やSQL出力には秘密情報が含まれる場合があるため、共有前に必要なマスクをする。

### RB-25 証拠保全

- 参照操作：`既存のJSONLをハッシュ付き保全`

- 確認観点：ローテーションや短期保持で消える前に複写。

- 成否判断：該当ログの実件数・実時刻を証拠とし、サービス終了コードのみで完全収集と断定しない。

- 注意：DB接続情報やSQL出力には秘密情報が含まれる場合があるため、共有前に必要なマスクをする。

## 17. 攻撃・障害シナリオ別の証拠相関手順

ケースは**将来の想定インシデント**。過去に起きた事実ではない。証拠台帳と時刻付きログを相関させるための標準手順。

### IR-01 Web経由の.php改ざん

- 検索順：Apacheアクセス→PowerCMS処理→auditd PATH / SYSCALL→ファイル差分

- 初動：事故想定時刻をJSTで明記し、UTCに換算した絞込範囲も併記する。

- 証拠単位：元の記録、受信時刻、収集元、ハッシュ値、取得者、対象インスタンスを記録。

- 誤判定防止：HTTPコード200は改ざん成功と同義でない。

- リスク：このケースは一般的な調査シナリオで、当該環境で攻撃が確認された意味ではない。

- 次回必要資料：関連するCloudWatch Logs Insightsの抽出結果と、改変前の保全コピー。

### IR-02 設定ファイルの無断閲覧

- 検索順：auditdの機密ファイルr→SSH/SSMログイン→CloudTrail認証関連

- 初動：事故想定時刻をJSTで明記し、UTCに換算した絞込範囲も併記する。

- 証拠単位：元の記録、受信時刻、収集元、ハッシュ値、取得者、対象インスタンスを記録。

- 誤判定防止：openat O_RDONLYだけでは持ち出しを確証できない。

- リスク：このケースは一般的な調査シナリオで、当該環境で攻撃が確認された意味ではない。

- 次回必要資料：関連するCloudWatch Logs Insightsの抽出結果と、改変前の保全コピー。

### IR-03 DB接続情報の悪用

- 検索順：設定ファイル読取→DBアクセスログ→VPC通信→IAM権限

- 初動：事故想定時刻をJSTで明記し、UTCに換算した絞込範囲も併記する。

- 証拠単位：元の記録、受信時刻、収集元、ハッシュ値、取得者、対象インスタンスを記録。

- 誤判定防止：RDSからの情報取得はファイルaudit単独では分からない。

- リスク：このケースは一般的な調査シナリオで、当該環境で攻撃が確認された意味ではない。

- 次回必要資料：関連するCloudWatch Logs Insightsの抽出結果と、改変前の保全コピー。

### IR-04 PowerCMS管理画面の不正操作

- 検索順：Apache access→mt_log→mt_rebuildlog→auditdのwa

- 初動：事故想定時刻をJSTで明記し、UTCに換算した絞込範囲も併記する。

- 証拠単位：元の記録、受信時刻、収集元、ハッシュ値、取得者、対象インスタンスを記録。

- 誤判定防止：CMSアカウントと実人物の識別を混同しない。

- リスク：このケースは一般的な調査シナリオで、当該環境で攻撃が確認された意味ではない。

- 次回必要資料：関連するCloudWatch Logs Insightsの抽出結果と、改変前の保全コピー。

### IR-05 再構築による公開ファイル変更

- 検索順：mt_rebuildlog→Webルートへのwa→正常バックアップとの差分

- 初動：事故想定時刻をJSTで明記し、UTCに換算した絞込範囲も併記する。

- 証拠単位：元の記録、受信時刻、収集元、ハッシュ値、取得者、対象インスタンスを記録。

- 誤判定防止：再構築ログ自体は不正改ざんを示さない。

- リスク：このケースは一般的な調査シナリオで、当該環境で攻撃が確認された意味ではない。

- 次回必要資料：関連するCloudWatch Logs Insightsの抽出結果と、改変前の保全コピー。

### IR-06 ec2-userアカウント濫用

- 検索順：OS login→audit AUID/SES→sudo USER_CMD→SYSCALL

- 初動：事故想定時刻をJSTで明記し、UTCに換算した絞込範囲も併記する。

- 証拠単位：元の記録、受信時刻、収集元、ハッシュ値、取得者、対象インスタンスを記録。

- 誤判定防止：共有アカウントなら人物断定は不可。

- リスク：このケースは一般的な調査シナリオで、当該環境で攻撃が確認された意味ではない。

- 次回必要資料：関連するCloudWatch Logs Insightsの抽出結果と、改変前の保全コピー。

### IR-07 root権限取得

- 検索順：sudo/PAM→auditのUID/EUID→重要ファイル変更

- 初動：事故想定時刻をJSTで明記し、UTCに換算した絞込範囲も併記する。

- 証拠単位：元の記録、受信時刻、収集元、ハッシュ値、取得者、対象インスタンスを記録。

- 誤判定防止：AUIDとEUIDを区別する。

- リスク：このケースは一般的な調査シナリオで、当該環境で攻撃が確認された意味ではない。

- 次回必要資料：関連するCloudWatch Logs Insightsの抽出結果と、改変前の保全コピー。

### IR-08 秘密鍵の読み取り

- 検索順：対象機密ファイルr→接続元セッション→ネットワークログ

- 初動：事故想定時刻をJSTで明記し、UTCに換算した絞込範囲も併記する。

- 証拠単位：元の記録、受信時刻、収集元、ハッシュ値、取得者、対象インスタンスを記録。

- 誤判定防止：読取アクセスの記録≠漏えい済み。

- リスク：このケースは一般的な調査シナリオで、当該環境で攻撃が確認された意味ではない。

- 次回必要資料：関連するCloudWatch Logs Insightsの抽出結果と、改変前の保全コピー。

### IR-09 EFSでの不正編集

- 検索順：接続EC2 audit→EFSアクセス経路→ハッシュ差分

- 初動：事故想定時刻をJSTで明記し、UTCに換算した絞込範囲も併記する。

- 証拠単位：元の記録、受信時刻、収集元、ハッシュ値、取得者、対象インスタンスを記録。

- 誤判定防止：別EC2からの変更は当該EC2 auditには出ない。

- リスク：このケースは一般的な調査シナリオで、当該環境で攻撃が確認された意味ではない。

- 次回必要資料：関連するCloudWatch Logs Insightsの抽出結果と、改変前の保全コピー。

### IR-10 他のEC2からのNFS操作

- 検索順：全EC2のaudit/SSM/SSH→EFS APIやバックアップ

- 初動：事故想定時刻をJSTで明記し、UTCに換算した絞込範囲も併記する。

- 証拠単位：元の記録、受信時刻、収集元、ハッシュ値、取得者、対象インスタンスを記録。

- 誤判定防止：単一EC2だけのログを全集合として扱わない。

- リスク：このケースは一般的な調査シナリオで、当該環境で攻撃が確認された意味ではない。

- 次回必要資料：関連するCloudWatch Logs Insightsの抽出結果と、改変前の保全コピー。

### IR-11 S3ログの削除

- 検索順：CloudTrail S3 data events→Object Lock→S3 versions

- 初動：事故想定時刻をJSTで明記し、UTCに換算した絞込範囲も併記する。

- 証拠単位：元の記録、受信時刻、収集元、ハッシュ値、取得者、対象インスタンスを記録。

- 誤判定防止：CloudTrailデータイベントが有効か未確認。

- リスク：このケースは一般的な調査シナリオで、当該環境で攻撃が確認された意味ではない。

- 次回必要資料：関連するCloudWatch Logs Insightsの抽出結果と、改変前の保全コピー。

### IR-12 CloudTrail停止

- 検索順：CloudTrail管理イベント→IAM権限→証跡保管

- 初動：事故想定時刻をJSTで明記し、UTCに換算した絞込範囲も併記する。

- 証拠単位：元の記録、受信時刻、収集元、ハッシュ値、取得者、対象インスタンスを記録。

- 誤判定防止：現環境の証跡設定は未確認。

- リスク：このケースは一般的な調査シナリオで、当該環境で攻撃が確認された意味ではない。

- 次回必要資料：関連するCloudWatch Logs Insightsの抽出結果と、改変前の保全コピー。

### IR-13 ログエクスポータ停止

- 検索順：systemd timer→service journal→Python例外→CloudWatch欠落

- 初動：事故想定時刻をJSTで明記し、UTCに換算した絞込範囲も併記する。

- 証拠単位：元の記録、受信時刻、収集元、ハッシュ値、取得者、対象インスタンスを記録。

- 誤判定防止：一時的な無イベントと障害を区別。

- リスク：このケースは一般的な調査シナリオで、当該環境で攻撃が確認された意味ではない。

- 次回必要資料：関連するCloudWatch Logs Insightsの抽出結果と、改変前の保全コピー。

### IR-14 CloudWatch Agent停止

- 検索順：systemd→Agentログ→log streamの最新取り込み

- 初動：事故想定時刻をJSTで明記し、UTCに換算した絞込範囲も併記する。

- 証拠単位：元の記録、受信時刻、収集元、ハッシュ値、取得者、対象インスタンスを記録。

- 誤判定防止：遅延/復旧後のバックフィルを考慮。

- リスク：このケースは一般的な調査シナリオで、当該環境で攻撃が確認された意味ではない。

- 次回必要資料：関連するCloudWatch Logs Insightsの抽出結果と、改変前の保全コピー。

### IR-15 Firehose配信障害

- 検索順：Subscription filter→Firehose metrics→S3オブジェクト

- 初動：事故想定時刻をJSTで明記し、UTCに換算した絞込範囲も併記する。

- 証拠単位：元の記録、受信時刻、収集元、ハッシュ値、取得者、対象インスタンスを記録。

- 誤判定防止：CloudWatchイベントの存在だけではS3保証なし。

- リスク：このケースは一般的な調査シナリオで、当該環境で攻撃が確認された意味ではない。

- 次回必要資料：関連するCloudWatch Logs Insightsの抽出結果と、改変前の保全コピー。

### IR-16 アクセススキャン

- 検索順：.env/wp-configへのHTTPリクエスト→対象の実ファイル存在→応答本文保全

- 初動：事故想定時刻をJSTで明記し、UTCに換算した絞込範囲も併記する。

- 証拠単位：元の記録、受信時刻、収集元、ハッシュ値、取得者、対象インスタンスを記録。

- 誤判定防止：200だけで秘密値漏洩と断定しない。

- リスク：このケースは一般的な調査シナリオで、当該環境で攻撃が確認された意味ではない。

- 次回必要資料：関連するCloudWatch Logs Insightsの抽出結果と、改変前の保全コピー。

### IR-17 SELinux AVC増加

- 検索順：AVC scontext/tcontext/permissive→Apache/PHP操作→動作への影響

- 初動：事故想定時刻をJSTで明記し、UTCに換算した絞込範囲も併記する。

- 証拠単位：元の記録、受信時刻、収集元、ハッシュ値、取得者、対象インスタンスを記録。

- 誤判定防止：permissive=1ならブロックしていない。

- リスク：このケースは一般的な調査シナリオで、当該環境で攻撃が確認された意味ではない。

- 次回必要資料：関連するCloudWatch Logs Insightsの抽出結果と、改変前の保全コピー。

### IR-18 DB情報の大量取得

- 検索順：DB監査/Performance Insights/アクセスログ/VPC情報

- 初動：事故想定時刻をJSTで明記し、UTCに換算した絞込範囲も併記する。

- 証拠単位：元の記録、受信時刻、収集元、ハッシュ値、取得者、対象インスタンスを記録。

- 誤判定防止：RDS監査は構築確認してから有効と記述。

- リスク：このケースは一般的な調査シナリオで、当該環境で攻撃が確認された意味ではない。

- 次回必要資料：関連するCloudWatch Logs Insightsの抽出結果と、改変前の保全コピー。

### IR-19 侵入後のコマンド履歴消去

- 検索順：auditd→shell history→journald→CloudTrail

- 初動：事故想定時刻をJSTで明記し、UTCに換算した絞込範囲も併記する。

- 証拠単位：元の記録、受信時刻、収集元、ハッシュ値、取得者、対象インスタンスを記録。

- 誤判定防止：historyは削除や編集が可能で単独証拠として弱い。

- リスク：このケースは一般的な調査シナリオで、当該環境で攻撃が確認された意味ではない。

- 次回必要資料：関連するCloudWatch Logs Insightsの抽出結果と、改変前の保全コピー。

### IR-20 管理者操作と攻撃の区別

- 検索順：操作申請/作業時間→AUID/SES→IPとSSM/SSH認証

- 初動：事故想定時刻をJSTで明記し、UTCに換算した絞込範囲も併記する。

- 証拠単位：元の記録、受信時刻、収集元、ハッシュ値、取得者、対象インスタンスを記録。

- 誤判定防止：業務操作の申告とOS証拠を照合。

- リスク：このケースは一般的な調査シナリオで、当該環境で攻撃が確認された意味ではない。

- 次回必要資料：関連するCloudWatch Logs Insightsの抽出結果と、改変前の保全コピー。

## 18. 証拠台帳フォーマットと受渡し手順

警察へ提出可能な状態を目指す場合、ログが存在することだけでなく、取得過程と保管履歴の一貫性を確保する。以下は運用用テンプレートで法的適格性の保証ではない。

### E-01 Evidence ID

- 入力内容：一意の管理番号

- 形式・参考例：EVID-YYYYMMDD-001 のような形式

- 注意：空欄不可

- 記入方針：事後の推測値を「実測」として埋めず、未取得は未取得のまま残す。

### E-02 インシデントID

- 入力内容：関連する調査番号

- 形式・参考例：INC-YYYYMMDD-001

- 注意：別の事件と混同しない

- 記入方針：事後の推測値を「実測」として埋めず、未取得は未取得のまま残す。

### E-03 証拠種類

- 入力内容：CloudWatch/S3/EC2/EBS/EFS/RDSなど

- 形式・参考例：audit.log

- 注意：実体形式を確認

- 記入方針：事後の推測値を「実測」として埋めず、未取得は未取得のまま残す。

### E-04 証拠名

- 入力内容：識別可能な名称

- 形式・参考例：audit-20261102-original.log

- 注意：原本とコピーを区別

- 記入方針：事後の推測値を「実測」として埋めず、未取得は未取得のまま残す。

### E-05 取得元サービス

- 入力内容：AWS/OS/CMS

- 形式・参考例：CloudWatch Logs

- 注意：収集元の階層を明示

- 記入方針：事後の推測値を「実測」として埋めず、未取得は未取得のまま残す。

### E-06 取得元AWSアカウント

- 入力内容：提出時は正確なアカウント

- 形式・参考例：アカウントIDは必要時のみ

- 注意：公開資料には載せない

- 記入方針：事後の推測値を「実測」として埋めず、未取得は未取得のまま残す。

### E-07 AWSリージョン

- 入力内容：対象サービスのリージョン

- 形式・参考例：ap-northeast-1

- 注意：S3の実リージョンも確認

- 記入方針：事後の推測値を「実測」として埋めず、未取得は未取得のまま残す。

### E-08 インスタンスID

- 入力内容：対象EC2 ID

- 形式・参考例：コンソール/CLI実値

- 注意：ホスト名との対応保持

- 記入方針：事後の推測値を「実測」として埋めず、未取得は未取得のまま残す。

### E-09 ホスト名

- 入力内容：収集元ホスト

- 形式・参考例：ip-172-21-3-236

- 注意：複数ホストを混在させない

- 記入方針：事後の推測値を「実測」として埋めず、未取得は未取得のまま残す。

### E-10 プライベートIP

- 入力内容：必要に応じて

- 形式・参考例：172.21.3.236

- 注意：IPだけで人物は分からない

- 記入方針：事後の推測値を「実測」として埋めず、未取得は未取得のまま残す。

### E-11 ファイルパス

- 入力内容：EC2上のパス

- 形式・参考例：/var/log/audit/audit.log

- 注意：特殊文字含む場合そのまま記録

- 記入方針：事後の推測値を「実測」として埋めず、未取得は未取得のまま残す。

### E-12 S3バケット

- 入力内容：保存先名称

- 形式・参考例：powercms-log-archive-logs-mchem-infratec-s3

- 注意：証拠保存先か通常ログ先か明記

- 記入方針：事後の推測値を「実測」として埋めず、未取得は未取得のまま残す。

### E-13 S3キー

- 入力内容：オブジェクトキー

- 形式・参考例：実キー

- 注意：秘匿値の含有に注意

- 記入方針：事後の推測値を「実測」として埋めず、未取得は未取得のまま残す。

### E-14 S3バージョンID

- 入力内容：バージョニング時

- 形式・参考例：VersionId

- 注意：未設定なら空欄

- 記入方針：事後の推測値を「実測」として埋めず、未取得は未取得のまま残す。

### E-15 監査イベント番号

- 入力内容：audit serial

- 形式・参考例：161394

- 注意：時刻と組み合わせる

- 記入方針：事後の推測値を「実測」として埋めず、未取得は未取得のまま残す。

### E-16 ロググループ

- 入力内容：CloudWatchグループ名

- 形式・参考例：/powercms/.../audit

- 注意：省略する場合は別途元名を保管

- 記入方針：事後の推測値を「実測」として埋めず、未取得は未取得のまま残す。

### E-17 ログストリーム

- 入力内容：EC2単位のstream

- 形式・参考例：i-0bd1b63ca374f1f78

- 注意：例はrebuild用

- 記入方針：事後の推測値を「実測」として埋めず、未取得は未取得のまま残す。

### E-18 イベント時刻

- 入力内容：元システムの時刻

- 形式・参考例：2026-11-02 17:00:14 JST

- 注意：UTC併記が望ましい

- 記入方針：事後の推測値を「実測」として埋めず、未取得は未取得のまま残す。

### E-19 取り込み時刻

- 入力内容：CloudWatch @ingestionTime

- 形式・参考例：実値

- 注意：元イベント時刻と分ける

- 記入方針：事後の推測値を「実測」として埋めず、未取得は未取得のまま残す。

### E-20 取得日時

- 入力内容：証拠ファイル抽出日時

- 形式・参考例：ISO 8601 TZ付き

- 注意：自動取得ならプロセスも記録

- 記入方針：事後の推測値を「実測」として埋めず、未取得は未取得のまま残す。

### E-21 取得者

- 入力内容：担当者・処理主体

- 形式・参考例：担当者名または運用ID

- 注意：共有する場合は個人情報に配慮

- 記入方針：事後の推測値を「実測」として埋めず、未取得は未取得のまま残す。

### E-22 取得方法

- 入力内容：CLI/UI/API等

- 形式・参考例：aws logs filter-log-events

- 注意：実際のパラメータを保存

- 記入方針：事後の推測値を「実測」として埋めず、未取得は未取得のまま残す。

### E-23 取得コマンド

- 入力内容：具体的な操作

- 形式・参考例：実際の履歴

- 注意：資格情報が含まれたら伏せる

- 記入方針：事後の推測値を「実測」として埋めず、未取得は未取得のまま残す。

### E-24 ハッシュ方式

- 入力内容：SHA-256

- 形式・参考例：SHA-256

- 注意：ハッシュ対象が何か明記

- 記入方針：事後の推測値を「実測」として埋めず、未取得は未取得のまま残す。

### E-25 ハッシュ値

- 入力内容：保全ファイルのハッシュ

- 形式・参考例：64桁16進数

- 注意：実計算値以外を記載しない

- 記入方針：事後の推測値を「実測」として埋めず、未取得は未取得のまま残す。

### E-26 バイト数

- 入力内容：原本のサイズ

- 形式・参考例：実値

- 注意：転送や圧縮後サイズと区別

- 記入方針：事後の推測値を「実測」として埋めず、未取得は未取得のまま残す。

### E-27 保全先

- 入力内容：WORMや読み取り専用保管先

- 形式・参考例：設定確認済みの実場所

- 注意：Object Lock設定前は強く断定しない

- 記入方針：事後の推測値を「実測」として埋めず、未取得は未取得のまま残す。

### E-28 保全担当

- 入力内容：取り扱い責任者

- 形式・参考例：運用担当

- 注意：いつ誰が触れたか継続記録

- 記入方針：事後の推測値を「実測」として埋めず、未取得は未取得のまま残す。

### E-29 アクセス履歴

- 入力内容：閲覧/複写/引渡し

- 形式・参考例：日時・目的・担当

- 注意：受渡し台帳と一致させる

- 記入方針：事後の推測値を「実測」として埋めず、未取得は未取得のまま残す。

### E-30 完全性検証

- 入力内容：再ハッシュ結果

- 形式・参考例：一致/不一致

- 注意：差が出たら原因調査

- 記入方針：事後の推測値を「実測」として埋めず、未取得は未取得のまま残す。

### E-31 削除期限

- 入力内容：CloudWatch/S3保持ポリシー

- 形式・参考例：現在短期テスト

- 注意：短期設定による失効リスク明記

- 記入方針：事後の推測値を「実測」として埋めず、未取得は未取得のまま残す。

### E-32 関連証拠

- 入力内容：他ログとのリンク

- 形式・参考例：アクセスIDやaudit serial

- 注意：相関は推測と事実を区別

- 記入方針：事後の推測値を「実測」として埋めず、未取得は未取得のまま残す。

### E-33 証拠説明

- 入力内容：何が分かるか

- 形式・参考例：操作の成功

- 注意：何が分からないかも記載

- 記入方針：事後の推測値を「実測」として埋めず、未取得は未取得のまま残す。

### E-34 共有先

- 入力内容：提出/閲覧先

- 形式・参考例：担当部署

- 注意：アクセス範囲を限定

- 記入方針：事後の推測値を「実測」として埋めず、未取得は未取得のまま残す。

## 19. 本番変更の安全確認チェックリスト

ここは次チャットで設定を変える前に必ず踏むチェック。

### CHG-01 auditdルール追加

- 事前参照：auditctl -l, auditctl -s, source .rules backup

- 主なリスク：アプリケーションI/O増加とログ量増

- 戻し方または安全条件：augenrules --loadで戻すが、-Dを含む再読込に注意

- 最低限の作業記録：変更前時刻・変更差分・実行者・実施コマンド・変更後検証結果。

- 終了条件：作業に対応するCloudWatch/S3の記録も、必要に応じて確保する。

### CHG-02 auditdのread監査追加

- 事前参照：対象ファイルとアクセス頻度

- 主なリスク：Apache/PHP-FPM読取でログ急増

- 戻し方または安全条件：高頻度領域のreadを外す

- 最低限の作業記録：変更前時刻・変更差分・実行者・実施コマンド・変更後検証結果。

- 終了条件：作業に対応するCloudWatch/S3の記録も、必要に応じて確保する。

### CHG-03 NFSディレクトリ監査

- 事前参照：findmntと複数クライアント

- 主なリスク：共有I/Oの観測漏れ

- 戻し方または安全条件：マウント跨ぎを調査

- 最低限の作業記録：変更前時刻・変更差分・実行者・実施コマンド・変更後検証結果。

- 終了条件：作業に対応するCloudWatch/S3の記録も、必要に応じて確保する。

### CHG-04 CloudWatch Agent再起動

- 事前参照：実効tomlとログ出力状態

- 主なリスク：一時欠落や再収集

- 戻し方または安全条件：必要時のみ変更承認して実施

- 最低限の作業記録：変更前時刻・変更差分・実行者・実施コマンド・変更後検証結果。

- 終了条件：作業に対応するCloudWatch/S3の記録も、必要に応じて確保する。

### CHG-05 PowerCMS exporterスクリプト修正

- 事前参照：スクリプトコピー/状態ファイル/ログ追跡

- 主なリスク：lastidによる重複/欠落

- 戻し方または安全条件：stateをむやみに編集しない

- 最低限の作業記録：変更前時刻・変更差分・実行者・実施コマンド・変更後検証結果。

- 終了条件：作業に対応するCloudWatch/S3の記録も、必要に応じて確保する。

### CHG-06 systemd timer変更

- 事前参照：unit設定とjournal

- 主なリスク：取得周期欠落

- 戻し方または安全条件：daemon-reloadやenableの前に差分確認

- 最低限の作業記録：変更前時刻・変更差分・実行者・実施コマンド・変更後検証結果。

- 終了条件：作業に対応するCloudWatch/S3の記録も、必要に応じて確保する。

### CHG-07 S3 lifecycle日数変更

- 事前参照：ルールJSON/バージョニング/Object Lock

- 主なリスク：早期削除・想定外保持

- 戻し方または安全条件：計画日になってから承認変更

- 最低限の作業記録：変更前時刻・変更差分・実行者・実施コマンド・変更後検証結果。

- 終了条件：作業に対応するCloudWatch/S3の記録も、必要に応じて確保する。

### CHG-08 S3 Object Lock導入

- 事前参照：対象バケット・版管理・モード

- 主なリスク：保持期間中の消去不能

- 戻し方または安全条件：専用バケット案を検討

- 最低限の作業記録：変更前時刻・変更差分・実行者・実施コマンド・変更後検証結果。

- 終了条件：作業に対応するCloudWatch/S3の記録も、必要に応じて確保する。

### CHG-09 CloudTrail有効化

- 事前参照：既存Trail・料金・範囲

- 主なリスク：重複課金や既存ルールとの競合

- 戻し方または安全条件：現行証跡を先に確認

- 最低限の作業記録：変更前時刻・変更差分・実行者・実施コマンド・変更後検証結果。

- 終了条件：作業に対応するCloudWatch/S3の記録も、必要に応じて確保する。

### CHG-10 Firehose配送設定変更

- 事前参照：subscription/filter/metrics

- 主なリスク：バッファ増、S3 prefix変更

- 戻し方または安全条件：既存配信先を保全

- 最低限の作業記録：変更前時刻・変更差分・実行者・実施コマンド・変更後検証結果。

- 終了条件：作業に対応するCloudWatch/S3の記録も、必要に応じて確保する。

### CHG-11 EBSスナップショット

- 事前参照：対象ボリュームと停止方針

- 主なリスク：IO一貫性/容量/料金

- 戻し方または安全条件：調査担当と実施判断

- 最低限の作業記録：変更前時刻・変更差分・実行者・実施コマンド・変更後検証結果。

- 終了条件：作業に対応するCloudWatch/S3の記録も、必要に応じて確保する。

### CHG-12 EFSバックアップ

- 事前参照：現在のバックアップポリシー

- 主なリスク：ネットワークI/Oと保存費

- 戻し方または安全条件：復元テスト方針

- 最低限の作業記録：変更前時刻・変更差分・実行者・実施コマンド・変更後検証結果。

- 終了条件：作業に対応するCloudWatch/S3の記録も、必要に応じて確保する。

### CHG-13 機密ファイルの監査

- 事前参照：対象パスだけを確認

- 主なリスク：シークレット誤露出

- 戻し方または安全条件：catで内容を無闇に共有しない

- 最低限の作業記録：変更前時刻・変更差分・実行者・実施コマンド・変更後検証結果。

- 終了条件：作業に対応するCloudWatch/S3の記録も、必要に応じて確保する。

### CHG-14 PowerCMSファイル更新

- 事前参照：作業計画と既存内容バックアップ

- 主なリスク：本番サイト改変

- 戻し方または安全条件：Web公開下に試験用PHPを置かない

- 最低限の作業記録：変更前時刻・変更差分・実行者・実施コマンド・変更後検証結果。

- 終了条件：作業に対応するCloudWatch/S3の記録も、必要に応じて確保する。

### CHG-15 ログ長期保持移行

- 事前参照：CloudWatch 30d、S3 400d

- 主なリスク：保持コスト増

- 戻し方または安全条件：テスト終了後に実施

- 最低限の作業記録：変更前時刻・変更差分・実行者・実施コマンド・変更後検証結果。

- 終了条件：作業に対応するCloudWatch/S3の記録も、必要に応じて確保する。

## 20. 障害発生時の優先順位別チェック

優先順位は本番影響、情報消失、証拠の時限性を考慮した一般的な案。

### INC-01 [P0] 証拠ログが削除期限で消えそう

- 初動：対象期間の現物を今すぐ別途保全、権限と変更履歴を記録

- リスク：設定変更の可否は運用権限者と協議

- エスカレーション資料：対象範囲、発生時刻、再現条件、一次ログ、保全先、未確定事項。

- 復旧の判定：技術的にイベントが再出現したことと、過去の欠落を回復できたことを分ける。

### INC-02 [P0] 侵入が進行中の疑い

- 初動：現行証拠を保全し、隔離/認証情報対処をIR責任者と協議

- リスク：調査のための無計画な再起動を避ける

- エスカレーション資料：対象範囲、発生時刻、再現条件、一次ログ、保全先、未確定事項。

- 復旧の判定：技術的にイベントが再出現したことと、過去の欠落を回復できたことを分ける。

### INC-03 [P1] 監査 lost が増加

- 初動：lost数/期間/CPU/IO/ログ量を採取

- リスク：watch範囲やbacklogの再設計

- エスカレーション資料：対象範囲、発生時刻、再現条件、一次ログ、保全先、未確定事項。

- 復旧の判定：技術的にイベントが再出現したことと、過去の欠落を回復できたことを分ける。

### INC-04 [P1] CloudWatch Agentが停止

- 初動：発生元ファイル保全→Agent設定と状態調査

- リスク：再起動前に既存オフセットを考慮

- エスカレーション資料：対象範囲、発生時刻、再現条件、一次ログ、保全先、未確定事項。

- 復旧の判定：技術的にイベントが再出現したことと、過去の欠落を回復できたことを分ける。

### INC-05 [P1] cms-rebuildに新規ログが出ない

- 初動：timer → service → DB新規レコード → output → Agent → CloudWatch

- リスク：再構築イベントがない正常状態と区別

- エスカレーション資料：対象範囲、発生時刻、再現条件、一次ログ、保全先、未確定事項。

- 復旧の判定：技術的にイベントが再出現したことと、過去の欠落を回復できたことを分ける。

### INC-06 [P1] Firehose配信エラー

- 初動：subscription→Firehose metrics→S3 permissions

- リスク：CloudWatchに残る期間中に再送可否を確認

- エスカレーション資料：対象範囲、発生時刻、再現条件、一次ログ、保全先、未確定事項。

- 復旧の判定：技術的にイベントが再出現したことと、過去の欠落を回復できたことを分ける。

### INC-07 [P1] S3で異常削除

- 初動：バージョン/CloudTrail data event/Object Lockを確認

- リスク：現物消失と不可視を区別

- エスカレーション資料：対象範囲、発生時刻、再現条件、一次ログ、保全先、未確定事項。

- 復旧の判定：技術的にイベントが再出現したことと、過去の欠落を回復できたことを分ける。

### INC-08 [P2] ログが重複

- 初動：JSONレコードID・監査serial・S3キーを比較

- リスク：検証中はstate変更禁止

- エスカレーション資料：対象範囲、発生時刻、再現条件、一次ログ、保全先、未確定事項。

- 復旧の判定：技術的にイベントが再出現したことと、過去の欠落を回復できたことを分ける。

### INC-09 [P2] タイムゾーンがずれる

- 初動：UTC/JSTと@timestamp/@ingestionTimeを分類

- リスク：epochを別の正しいTZで再計算

- エスカレーション資料：対象範囲、発生時刻、再現条件、一次ログ、保全先、未確定事項。

- 復旧の判定：技術的にイベントが再出現したことと、過去の欠落を回復できたことを分ける。

### INC-10 [P2] 監査イベントにPATHがない

- 初動：msg=auditシリアルを相関

- リスク：CONFIG_CHANGE系はPATHの意味が違う

- エスカレーション資料：対象範囲、発生時刻、再現条件、一次ログ、保全先、未確定事項。

- 復旧の判定：技術的にイベントが再出現したことと、過去の欠落を回復できたことを分ける。

### INC-11 [P2] S3の3日期限が効かない

- 初動：LastModified/versioning/Object Lock/非同期実行

- リスク：明確化まで手動削除しない

- エスカレーション資料：対象範囲、発生時刻、再現条件、一次ログ、保全先、未確定事項。

- 復旧の判定：技術的にイベントが再出現したことと、過去の欠落を回復できたことを分ける。

### INC-12 [P3] 月額見積りが必要

- 初動：5 group IncomingBytesを24h実測

- リスク：仮説MB/dayは根拠値としない

- エスカレーション資料：対象範囲、発生時刻、再現条件、一次ログ、保全先、未確定事項。

- 復旧の判定：技術的にイベントが再出現したことと、過去の欠落を回復できたことを分ける。

## 21. 過去の具体的な時刻・イベントと検証済み判定

過去の確認ログを単一のタイムラインにする。全てJST。提示されなかった中間イベントは創作しない。

### TIME-01 2026-11-02 10:49:36

- 出来事：CloudWatch Agent開始

- 資料：systemctl statusから後で確認

- この時刻はユーザーが提示したログの解釈。EventTimeとCloudWatch収集時刻は異なる場合がある。

- 次チャットでは事実の再質問を避ける。関連ログを追加取得する必要がある場合は限定した時間窓で照会する。

### TIME-02 2026-11-02 11:02:15

- 出来事：CMSログエクスポートtimer開始

- 資料：systemctl statusから後で確認

- この時刻はユーザーが提示したログの解釈。EventTimeとCloudWatch収集時刻は異なる場合がある。

- 次チャットでは事実の再質問を避ける。関連ログを追加取得する必要がある場合は限定した時間窓で照会する。

### TIME-03 2026-11-02 16:57:48

- 出来事：初期never,taskルール削除のCONFIG_CHANGE

- 資料：161280, remove_rule

- この時刻はユーザーが提示したログの解釈。EventTimeとCloudWatch収集時刻は異なる場合がある。

- 次チャットでは事実の再質問を避ける。関連ログを追加取得する必要がある場合は限定した時間窓で照会する。

### TIME-04 2026-11-02 16:58:36

- 出来事：ローカルfile_test追加

- 資料：161322, CONFIG_CHANGE

- この時刻はユーザーが提示したログの解釈。EventTimeとCloudWatch収集時刻は異なる場合がある。

- 次チャットでは事実の再質問を避ける。関連ログを追加取得する必要がある場合は限定した時間窓で照会する。

### TIME-05 2026-11-02 16:59:56

- 出来事：local touch CREATE

- 資料：161364, openat success=yes

- この時刻はユーザーが提示したログの解釈。EventTimeとCloudWatch収集時刻は異なる場合がある。

- 次チャットでは事実の再質問を避ける。関連ログを追加取得する必要がある場合は限定した時間窓で照会する。

### TIME-06 2026-11-02 17:00:09

- 出来事：local chmod

- 資料：161393, fchmodat success=yes

- この時刻はユーザーが提示したログの解釈。EventTimeとCloudWatch収集時刻は異なる場合がある。

- 次チャットでは事実の再質問を避ける。関連ログを追加取得する必要がある場合は限定した時間窓で照会する。

### TIME-07 2026-11-02 17:00:14

- 出来事：local rm DELETE

- 資料：161394, unlinkat success=yes

- この時刻はユーザーが提示したログの解釈。EventTimeとCloudWatch収集時刻は異なる場合がある。

- 次チャットでは事実の再質問を避ける。関連ログを追加取得する必要がある場合は限定した時間窓で照会する。

### TIME-08 2026-11-02 17:05頃

- 出来事：vimによるテスト編集

- 資料：161515-161521、過去の10分ログ内

- この時刻はユーザーが提示したログの解釈。EventTimeとCloudWatch収集時刻は異なる場合がある。

- 次チャットでは事実の再質問を避ける。関連ログを追加取得する必要がある場合は限定した時間窓で照会する。

### TIME-09 2026-11-02 17:14:50

- 出来事：local read rule追加

- 資料：161727, CONFIG_CHANGE

- この時刻はユーザーが提示したログの解釈。EventTimeとCloudWatch収集時刻は異なる場合がある。

- 次チャットでは事実の再質問を避ける。関連ログを追加取得する必要がある場合は限定した時間窓で照会する。

### TIME-10 2026-11-02 17:14:58

- 出来事：catでread open

- 資料：161730, openat O_RDONLY success=yes

- この時刻はユーザーが提示したログの解釈。EventTimeとCloudWatch収集時刻は異なる場合がある。

- 次チャットでは事実の再質問を避ける。関連ログを追加取得する必要がある場合は限定した時間窓で照会する。

### TIME-11 2026-11-02 17:14:58

- 出来事：lessでread open

- 資料：161736, openat O_RDONLY success=yes

- この時刻はユーザーが提示したログの解釈。EventTimeとCloudWatch収集時刻は異なる場合がある。

- 次チャットでは事実の再質問を避ける。関連ログを追加取得する必要がある場合は限定した時間窓で照会する。

### TIME-12 2026-11-02 17:21:43

- 出来事：NFSテストルール追加

- 資料：161965 CONFIG_CHANGE

- この時刻はユーザーが提示したログの解釈。EventTimeとCloudWatch収集時刻は異なる場合がある。

- 次チャットでは事実の再質問を避ける。関連ログを追加取得する必要がある場合は限定した時間窓で照会する。

### TIME-13 2026-11-02 17:21:53

- 出来事：NFS touch CREATE

- 資料：161974, openat success=yes

- この時刻はユーザーが提示したログの解釈。EventTimeとCloudWatch収集時刻は異なる場合がある。

- 次チャットでは事実の再質問を避ける。関連ログを追加取得する必要がある場合は限定した時間窓で照会する。

### TIME-14 2026-11-02 17:21:53

- 出来事：NFS chmod

- 資料：161975, fchmodat success=yes

- この時刻はユーザーが提示したログの解釈。EventTimeとCloudWatch収集時刻は異なる場合がある。

- 次チャットでは事実の再質問を避ける。関連ログを追加取得する必要がある場合は限定した時間窓で照会する。

### TIME-15 2026-11-02 17:21:53

- 出来事：NFS rm DELETE

- 資料：161976, unlinkat success=yes

- この時刻はユーザーが提示したログの解釈。EventTimeとCloudWatch収集時刻は異なる場合がある。

- 次チャットでは事実の再質問を避ける。関連ログを追加取得する必要がある場合は限定した時間窓で照会する。

### TIME-16 2026-11-02 17:25:56

- 出来事：本番Web監査ルール追加

- 資料：162195, CONFIG_CHANGE

- この時刻はユーザーが提示したログの解釈。EventTimeとCloudWatch収集時刻は異なる場合がある。

- 次チャットでは事実の再質問を避ける。関連ログを追加取得する必要がある場合は限定した時間窓で照会する。

### TIME-17 2026-11-04 11:44-12:13

- 出来事：CMS exporter毎分実行成功

- 資料：journalctlで各分のFinished確認

- この時刻はユーザーが提示したログの解釈。EventTimeとCloudWatch収集時刻は異なる場合がある。

- 次チャットでは事実の再質問を避ける。関連ログを追加取得する必要がある場合は限定した時間窓で照会する。

### TIME-18 2026-11-04 11:57

- 出来事：CloudWatch cms-rebuildイベント

- 資料：スクリーンショット

- この時刻はユーザーが提示したログの解釈。EventTimeとCloudWatch収集時刻は異なる場合がある。

- 次チャットでは事実の再質問を避ける。関連ログを追加取得する必要がある場合は限定した時間窓で照会する。

### TIME-19 2026-11-04 12:10

- 出来事：CloudWatch cms-rebuild複数イベント

- 資料：スクリーンショット

- この時刻はユーザーが提示したログの解釈。EventTimeとCloudWatch収集時刻は異なる場合がある。

- 次チャットでは事実の再質問を避ける。関連ログを追加取得する必要がある場合は限定した時間窓で照会する。

### TIME-20 2026-11-04

- 出来事：S3ライフサイクル3日が画面に存在

- 資料：ユーザー画像。非現行設定なし

- この時刻はユーザーが提示したログの解釈。EventTimeとCloudWatch収集時刻は異なる場合がある。

- 次チャットでは事実の再質問を避ける。関連ログを追加取得する必要がある場合は限定した時間窓で照会する。

## 22. 将来の確認が必要な論点（未確認台帳）

この一覧は**未完成な部分を未完成のまま明示**するためのもの。

### OPEN-01 S3オブジェクトが3日超でも残る理由

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：対象キーのLastModified、version、Lock、ルールJSON

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-02 S3バージョニングの有効状態

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：get-bucket-versioning

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-03 S3 Object Lock構成

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：get-object-lock-configuration

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-04 S3非現行バージョンの保持実体

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：list-object-versions

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-05 Firehoseへの配信成功

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：delivery metricsとS3オブジェクトの実確認

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-06 cms-rebuildのS3到着

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：cms-rebuild/prod/ 配下の実オブジェクト

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-07 cms-activityの最新CloudWatch着信

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：同ロググループAll eventsで確認

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-08 DB mt_rebuildlogとJSONLの件数一致

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：DB max(id)とstate/出力IDを比較

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-09 再構築ログexporter例外時の挙動

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：Python全コードの安全なレビュー

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-10 stateファイル更新タイミング

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：出力/flush完了前後の順序

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-11 Pythonログ出力のローテーション

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：logrotateまたは独自設計

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-12 CloudWatch Agentの再開オフセット

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：Agent実装とログで確認

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-13 監査ログのローテーション方針

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：auditd.confとlogrotate

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-14 lostの長期推移

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：監視としきい値

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-15 backlog_limit=64が足りるか

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：ピーク時の実測

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-16 EFSへの他クライアント接続

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：EFS/NFSマウント構成

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-17 EFS製品か別のNFS構成か

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：mount/プロセス/設定を調査

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-18 PowerCMS実ファイル変更の検知実績

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：安全な通常更新時にPATH/KEY照合

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-19 NFSからCloudWatchへのaudit詳細到着

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：指定serialで照合

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-20 NFS上の読み取り監査

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：必要な機密ファイルで限定検証

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-21 機密ファイルの所在

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：秘密値を表示せずパスだけを確認

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-22 秘密鍵への読み取り監査

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：対象を限定し負荷評価

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-23 PHP-FPMの処理単位相関

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：request-id設計

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-24 CloudFrontの実アクセス元

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：ViewerIPとXFF等の取り扱い

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-25 ALBの存在とログ

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：AWS構成を確認

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-26 WAFの存在とログ

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：AWS構成を確認

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-27 VPC Flow Logs

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：既存設定と収集範囲

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-28 CloudTrail管理イベント

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：Trailの設定と証跡の永続保管

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-29 CloudTrail S3 Data Events

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：対象バケット・コスト・必要性

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-30 CloudTrail integrity validation

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：有効状態とダイジェスト

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-31 S3の長期証拠保存

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：400日設定移行日・ルール

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-32 CloudWatchの30日保存

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：5グループの保持設定実値

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-33 S3削除対策

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：Object Lock/権限分離/別アカウント

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-34 CMSの重大イベント再現テスト

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：ステージングでの管理画面テスト

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-35 ステージング収集経路

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：ステージングEC2とDB userの状態

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-36 パスワード露出後のローテーション

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：秘密情報を新チャットに貼らず運用判断

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-37 月額ログ実測

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：5つのIncomingBytes 24h分

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-38 Firehoseの課金対象バイト

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：CloudWatchとは違う計量を確認

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-39 保全時のチェーンオブカストディ

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：実運用テンプレートと責任者

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-40 EBSスナップショットの保全手順

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：障害・インシデント体制

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-41 RDSスナップショット保全手順

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：DBのタイムライン

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-42 バックアップとハッシュ基準値

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：正常時のファイルベースライン

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

### OPEN-43 検知後の隔離と復旧の手順

- 状態：**未確認／未実施**。現状稼働済みと断定しない。

- 必要な証拠：権限者の承認フロー

- 依存関係：直近のS3ライフサイクル調査と、本番ログ保全の安全性を優先。

- 記録すべき結果：実施日時・観測された設定値・エラーメッセージ全文・次回対応。

## 23. 次のチャット向け即応質問・回答集

引き継ぎ先で事実を聞き直さないための短い答え。

### HANDOFF-01 何が一番新しい問題か

**引き継ぎ回答**：S3ルールが3日なのに3日以前と思われるログファイルが残る問題。

**補足**：最新のユーザー出力と不一致なら、最新の実測値を優先して差分を記録する。

### HANDOFF-02 ルールは本当に400日か

**引き継ぎ回答**：名前は400daysだが2026-11-04画面は3日。

**補足**：最新のユーザー出力と不一致なら、最新の実測値を優先して差分を記録する。

### HANDOFF-03 ユーザーは今すぐ400日に戻したいか

**引き継ぎ回答**：いいえ。全部構築完了後に戻す意向。

**補足**：最新のユーザー出力と不一致なら、最新の実測値を優先して差分を記録する。

### HANDOFF-04 S3は証拠保全用か

**引き継ぎ回答**：ログアーカイブ用。Object Lockや分離保管は未確認。

**補足**：最新のユーザー出力と不一致なら、最新の実測値を優先して差分を記録する。

### HANDOFF-05 cms-rebuildは動くか

**引き継ぎ回答**：毎分timerでPython2つが起動し0/SUCCESS、CloudWatch All eventsにJSON行が存在。

**補足**：最新のユーザー出力と不一致なら、最新の実測値を優先して差分を記録する。

### HANDOFF-06 cms-rebuildは欠落なしと言えるか

**引き継ぎ回答**：いいえ。DB/state/CloudWatch/S3の件数突合は未実施。

**補足**：最新のユーザー出力と不一致なら、最新の実測値を優先して差分を記録する。

### HANDOFF-07 CloudWatch log stream 0は確定か

**引き継ぎ回答**：いいえ。別画面のAll eventsでstream名とイベントが確認され、誤認が解消。

**補足**：最新のユーザー出力と不一致なら、最新の実測値を優先して差分を記録する。

### HANDOFF-08 現在のauditdの重点監視範囲は

**引き継ぎ回答**：/etc、/usr/local/bin、/usr/local/sbin、/var/www/html のwa。

**補足**：最新のユーザー出力と不一致なら、最新の実測値を優先して差分を記録する。

### HANDOFF-09 auditの最初の障害は

**引き継ぎ回答**：初期 task,never ルールでsyscall監査が抑制された。

**補足**：最新のユーザー出力と不一致なら、最新の実測値を優先して差分を記録する。

### HANDOFF-10 ファイルの読取監査は導入済みか

**引き継ぎ回答**：ローカルtest dirの一時rルールでcat/lessの検証成功。本番秘密ファイルへは未適用。

**補足**：最新のユーザー出力と不一致なら、最新の実測値を優先して差分を記録する。

### HANDOFF-11 NFSで監査できるか

**引き継ぎ回答**：このEC2の/var/www/audit-testでは作成/属性変更/削除を記録。

**補足**：最新のユーザー出力と不一致なら、最新の実測値を優先して差分を記録する。

### HANDOFF-12 NFSの正体は

**引き継ぎ回答**：findmntではnfs4, 127.0.0.1:/。EFSと断定できる材料ではない。

**補足**：最新のユーザー出力と不一致なら、最新の実測値を優先して差分を記録する。

### HANDOFF-13 本番PowerCMSの変更実績は

**引き継ぎ回答**：本番waルール登録成功のみで、実ファイル更新捕捉はまだ未確認。

**補足**：最新のユーザー出力と不一致なら、最新の実測値を優先して差分を記録する。

### HANDOFF-14 audit lostはいくつか

**引き継ぎ回答**：確認時点で0。将来の取りこぼしゼロ保証ではない。

**補足**：最新のユーザー出力と不一致なら、最新の実測値を優先して差分を記録する。

### HANDOFF-15 audit.logの大きさは

**引き継ぎ回答**：一時点で6.1MB。月換算に必要な時間基準なし。

**補足**：最新のユーザー出力と不一致なら、最新の実測値を優先して差分を記録する。

### HANDOFF-16 S3保存期間の現時点は

**引き継ぎ回答**：最新画面では3日、非現行期限なし。

**補足**：最新のユーザー出力と不一致なら、最新の実測値を優先して差分を記録する。

### HANDOFF-17 CloudWatch保存期間の現時点は

**引き継ぎ回答**：短期削除テスト継続というユーザー意向。ただし各groupの実値は再確認する。

**補足**：最新のユーザー出力と不一致なら、最新の実測値を優先して差分を記録する。

### HANDOFF-18 長期保存の目標は

**引き継ぎ回答**：CloudWatch約30日、S3約400日。

**補足**：最新のユーザー出力と不一致なら、最新の実測値を優先して差分を記録する。

### HANDOFF-19 料金の試算は

**引き継ぎ回答**：以前は5グループ計180MB/日という仮定で月5.4GB、約750〜1500円と言ったが実測ではない。

**補足**：最新のユーザー出力と不一致なら、最新の実測値を優先して差分を記録する。

### HANDOFF-20 警察に使えるか

**引き継ぎ回答**：操作の客観記録になり得るが、人物特定や法的評価を自動保証しない。

**補足**：最新のユーザー出力と不一致なら、最新の実測値を優先して差分を記録する。

### HANDOFF-21 sudo USER_CMDでファイル削除を証明できるか

**引き継ぎ回答**：単独では不十分。audit syscall+PATHなどが必要。

**補足**：最新のユーザー出力と不一致なら、最新の実測値を優先して差分を記録する。

### HANDOFF-22 catで流出を証明できるか

**引き継ぎ回答**：read openを証明しても外部送信を直接示せない。

**補足**：最新のユーザー出力と不一致なら、最新の実測値を優先して差分を記録する。

### HANDOFF-23 過去のAVCは攻撃か

**引き継ぎ回答**：permissive=1のSELinux警告は単独で侵入成功を示さない。

**補足**：最新のユーザー出力と不一致なら、最新の実測値を優先して差分を記録する。

### HANDOFF-24 `.env` への200は漏洩か

**引き継ぎ回答**：応答本文、ファイルの実在、HTTP層の経路検証なしに断定しない。

**補足**：最新のユーザー出力と不一致なら、最新の実測値を優先して差分を記録する。

### HANDOFF-25 最優先の次のコマンドは

**引き継ぎ回答**：S3 get-bucket-versioning、get-bucket-lifecycle-configuration、残存キーのhead-object。

**補足**：最新のユーザー出力と不一致なら、最新の実測値を優先して差分を記録する。

### HANDOFF-26 設定をいま変更してよいか

**引き継ぎ回答**：ユーザーはS3短期保持を維持中。原則先に読み取り専用で確認。

**補足**：最新のユーザー出力と不一致なら、最新の実測値を優先して差分を記録する。

## 24. 追加ログを受け取ったときの回答スタイル

- 日本語で具体的に答える。
- まず「何が確定したか」「何が未確認か」を明示する。
- 監査イベントの成功/失敗は `success=yes/no`、`PATH`、監査serialをセットで確認。
- `USER_CMD` や systemd `status=0/SUCCESS` に証明能力を持たせ過ぎない。
- コマンドは1〜3個程度の短い検証単位へ分け、ユーザーが結果を返したら次に進む。
- 本番に影響する変更や削除は自動的に実施しない。
- シークレットは表示も共有も避ける。
- DBの取得スクリプトはstate/lock/ファイル出力の関係があり、無断実行や初期化を避ける。
- 長いログを貼る場合は時間範囲を限定する。
- AWS画面で一度「0件」に見えても、フィルター・表示範囲・ページの違いを疑う。
- 事実と推定の間に境界を置く。
- 証拠保全優先度が高い事象では、短期保持により失効する前に保存を検討する。

### 24.1 コピペ用：現在の監査状態の総覧 [READ ONLY]

```bash
sudo auditctl -l
sudo auditctl -s
sudo ls -l /etc/audit/rules.d/
sudo grep -Rn 'task,never\|never,task' /etc/audit/rules.d /etc/audit/audit.rules
sudo du -h /var/log/audit/audit.log
df -h /var/log
```

### 24.2 コピペ用：CMS再構築エクスポータの生存確認 [READ ONLY]

```bash
sudo systemctl list-timers --all | grep prod_mchem
sudo systemctl status prod_mchem_infratec_log_export.timer --no-pager
sudo systemctl status prod_mchem_infratec_log_export.service --no-pager
sudo journalctl -u prod_mchem_infratec_log_export.service --since '30 minutes ago' --no-pager
sudo stat /var/log/powercms/prod_mchem_infratec_rebuild.log
```

### 24.3 コピペ用：S3の最新の問題 [READ ONLY]

```bash
aws s3api get-bucket-versioning --bucket powercms-log-archive-logs-mchem-infratec-s3
aws s3api get-bucket-lifecycle-configuration --bucket powercms-log-archive-logs-mchem-infratec-s3
```

### 24.4 最後に必ず確認すること

- CloudWatch 30日・S3 400日は**最終計画**であり、2026-11-04時点の完了事項ではない。
- 2026-11-04のS3画面は最新オブジェクトの有効期限「3日」。
- バージョニング、Object Lock、残存オブジェクトのLastModifiedは**未確認**。
- 今最初にやる作業はS3ライフサイクルの読取調査で、削除テストや保持期限の上書きではない。
- 次チャットではこの付録だけでなく最初の0〜10章も参照する。
