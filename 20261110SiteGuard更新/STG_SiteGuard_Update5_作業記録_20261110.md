# STG SiteGuard Server Edition Update5 適用 作業記録

- 実施日時: 2026-11-10 15:57〜16:04（実作業 約7分。ビルド 約3分を含む）
- 対象: STG 27.133.156.74（ホスト名 STG）。**PROD は触っていない**
- 実施者: Claude Code（徳原さんの指示による）。root で SSH、接続元 118.243.16.51
- 手順書: `../SiteGuard_Update5_STG作業手順書.md` の手順1〜7（手順8のシグネチャ更新は未実施）
- 位置づけ: 一旦更新して確認し、**後で元に戻す予定**（戻し方は手順書の「9. 戻し方」）
- 事前のアーカイブ（手順0）は取っていない。完全に戻す必要がある場合は、イメージ単位の日次バックアップを使う

---

## 1. 更新前と更新後のバージョン

| 項目 | 更新前（15:57） | 更新後（16:03） | 確認コマンド |
|---|---|---|---|
| **SiteGuard Server Edition** | **8.00 Update 3 [64_1.29]** | **8.00 Update 5 [64_1.29]** | `cd /opt/jp-secure/siteguardlite && make show_webui_version` |
| `siteguardlite.ini` の version | `Nginx800-3` | `Nginx800-5` | `head -1 /opt/jp-secure/siteguardlite/conf/siteguardlite.ini` |
| SiteGuard モジュールのソース（`/opt/jp-secure/siteguardlite/nginx/`） | `siteguard.a` 2025-11-17、`ngx_http_siteguard_module.c` 2024-09-25 | `siteguard.a` 2026-09-28、`ngx_http_siteguard_module.c` 2026-09-28、`libpcre2-8.a` 追加 | `ls -la --time-style=long-iso /opt/jp-secure/siteguardlite/nginx/` |
| nginx | 1.29.8 | 1.29.8（変更なし） | `/opt/kusanagi/nginx129/sbin/nginx -V` |
| nginx の configure 引数 | （`nginx-V_before.txt`） | **完全に同じ**（diff で一致） | `diff <(…before) <(…new)` → `args-same` |
| nginx のビルドに使った gcc | 11.5.0-14 | 11.5.0-15 | `nginx -V` の 2行目 |
| nginx バイナリの sha256 | `c8cc95e8c3032588d003b764edd3426683a56e9343bb38b0516636d860e2c25f` | `30cac987769c53aae8b0508fa6b81491373ffa729432744e5183981ca3d50633` | `sha256sum /opt/kusanagi/nginx129/sbin/nginx` |
| OpenSSL（リンク先） | `/opt/kusanagi/openssl/lib64/libssl.so.3`（3.5.6 でビルド、3.5.7 で実行） | 同じ | `ldd … \| grep ssl` |
| トラステッド・シグネチャ | 2025-10-23_01 | 2025-10-23_01（今回は更新していない） | `cat conf/waf/header.ini` |
| `statistics` | `no` | `no`（make install で `yes` に戻されたので直した） | `grep '^statistics' conf/siteguardlite.ini` |
| 統計の cron | コメントアウト | コメントアウト（変更なし） | `crontab -l` |
| versionlock | なし | `kusanagi-nginx129-0:1.29.8-1.el9.*` | `dnf versionlock list` |
| php-fpm | inactive（メンテナンス表示） | inactive（変更なし） | `systemctl is-active php-fpm` |

## 2. 動作確認の結果

| 確認 | 更新前 | 更新後 |
|---|---|---|
| `/WAF-TEST-SIGNATURE`（Basic 認証あり） | **403**（detect.log に `url-waf-test-1` / `ACTION:BLOCK`） | **403**（同じ） |
| トップページ | ― | 503（メンテナンス表示。php-fpm 停止中のため想定どおり） |
| 管理画面 `https://127.0.0.1:9443/` | ― | 200 |
| `siteguardlite_admin` / `siteguardlite_notify` | active | active（make install で 15:58 に再起動） |
| `nginx -t` | OK | OK |
| SiteGuard の初期化 | ― | `siteguard: configuration normal completed. licensed=yes enabled=yes`（nginx の error.log、SiteGuard の info.log） |
| SiteGuard の error.log | ― | 出力なし |
| 設定の引き継ぎ | ― | `sig_custom.txt`、`access_filter.txt`（7件）、`sig_official.txt` は更新前と完全に同じ。`siteguardlite.ini` の違いは version の1行だけ |

ブラウザでの管理画面ログイン、WordPress・印刷機能の確認はまだ（php-fpm を起動して行う。手順書の手順8）。

## 3. 実施した作業（時刻順）

| 時刻 | 手順 | 内容 |
|---|---|---|
| 15:57:27 | 1 | 作業前の記録（バージョン、nginx -V、sha256、crontab、nginx.conf、WAF テスト） |
| 15:57:28 | 2 | 退避: `make backup`、SiteGuard 一式の tar（`siteguardlite_full_8.00-3.tar.gz` 100MB）、systemd のユニット2つ、OpenSSL フォルダの一覧、サービスの状態、目印ファイル |
| 15:57:44 | 3 | `dnf versionlock add kusanagi-nginx129` |
| 15:58:15 | 4 | パッケージの sha256 確認、展開 |
| 15:58:17〜36 | 4 | `make install`（`Update succeeded!`）。**nginx は再起動されなかった**（PID 3519979 のまま） |
| 15:58:49 | 4 | `/opt` の外で変わったファイルの洗い出し |
| 15:59:07 | 4 補足 | `statistics=yes` → `no` に戻した |
| 15:59:28 | 5-1, 5-2 | nginx ソースを `/usr/local/src/build-sg5/` に展開。OpenSSL のリンク（`libssl.so`、`libcrypto.so`）を作成 |
| 15:59:28〜41 | 5-3 | configure（今の `nginx -V` から引数を取り出して実行）。`ngx_http_siteguard_module was configured` |
| 15:59:54〜16:03:05 | 5-4 | make（エラー0件） |
| 16:03:24 | 5-5 | 引数の一致、ldd、`objs/nginx -t`、sha256 を確認 |
| 16:03:48 | 6 | `nginx.sg3_20261110` に退避、`mv` で差し替え |
| 16:03:49〜50 | 6 | `nginx -t` → `systemctl restart nginx129`（新しい PID 3903235） |
| 16:03:50〜56 | 7 | 動作確認 |

## 4. 作業で変わったもの（戻すときの対象）

| 場所 | 変更内容 | 戻し方（手順書） |
|---|---|---|
| `/opt/jp-secure/siteguardlite/` | Update5 に更新（`statistics=no` は維持） | 9-2（`siteguardlite_full_8.00-3.tar.gz` から戻す） |
| `/opt/kusanagi/nginx129/sbin/nginx` | Update5 のモジュール入りに差し替え | 9-1（`nginx.sg3_20261110` に戻す） |
| `/opt/kusanagi/nginx129/sbin/nginx.sg3_20261110` | 新規（Update3 版の退避） | 戻した後は消してよい |
| `/etc/systemd/system/siteguardlite_admin.service`、`_notify.service` | make install で書き直し（**中身は同じ**） | 不要（9-2 で念のため元に戻す） |
| `/etc/opt/kusanagi/nginx/nginx.conf`、`nginx.conf.bak` | make install で書き直し（nginx.conf は中身が同じ。.bak は作り直し・大きさは同じ 3285 バイト） | 不要 |
| `/etc/dnf/plugins/versionlock.list` | `kusanagi-nginx129` を登録 | 9-4（完全に戻すときだけ） |
| `/opt/kusanagi/openssl/lib64/libssl.so`、`libcrypto.so` | シンボリックリンクを新規作成 | 9-4 |
| `/usr/local/src/siteguard-8.00-5/siteguard-server-edition-8.00-5.nginx.x86_64/` | パッケージを展開（約150MB） | 9-4（削除） |
| `/usr/local/src/build-sg5/` | nginx のビルド（約70MB） | 9-4（削除） |
| `/root/sg5_work/` | 記録と退避 | 戻した後、Mac に保存済みなら消してよい |

## 5. 分かったこと（PROD の手順書に反映する）

1. **パッケージの `make install` では nginx は再起動されない。** SiteGuard の管理画面と通知サービスは再起動される。所要時間は約20秒
2. **`make install` は `statistics=yes` に戻す。** 直後に `no` に戻す手順が必須（統計の cron はコメントアウトのままだった）
3. `make install` は `nginx.conf`、`nginx.conf.bak`、systemd のユニット2つを書き直すが、中身は変わらない。`dbupdate_waf_url.conf` も変わらない（`_sp` なしのまま）
4. **ビルドは約3分**（CPU 1コア）。見積の「SiteGuard 再ビルド・STG 適用・検知確認 8H」よりかなり短く済む
5. nginx の差し替え（restart）で止まったのは約1秒
6. 作業中、monit が STG のアクセスログの 503（動作確認のトップページ）を異常と判断して、16:03:57 に `kusanagi restart` を実行した（11/9 に確認済みの「monit の再起動ループ」）。php-fpm は止まったまま、nginx は新しいバイナリのまま動いていることを 16:04:38 に確認した
7. nginx の error.log の `detected a LuaJIT version which is not OpenResty's` は、9/7 から出ている既存の警告で、今回の作業によるものではない

## 6. エビデンス

| ファイル | 中身 |
|---|---|
| `01_step1-3.log` | 手順1〜3の実行コマンドと出力 |
| `02_step4.log` | 手順4（パッケージの make install）の実行コマンドと出力 |
| `03_step4_statistics.log` | `statistics` を no に戻した記録 |
| `04_step5_configure.log` | 手順5-1〜5-3 |
| `05_step5_make.log` | make の開始 |
| `06_step5_verify.log` | 手順5-5（ビルド結果の確認） |
| `07_step6-7.log` | 手順6（差し替え）と手順7（動作確認） |
| `08_luajit_check.txt` | LuaJIT の警告が既存のものかの確認 |
| `sg5_work_STG/` | STG の `/root/sg5_work/` のコピー（`evidence.log` は全コマンドの記録。`siteguardlite_full_8.00-3.tar.gz` と `siteguard_backup_8.00-3.tar.gz` は大きいので STG にだけ置いている） |

- コマンドの記録の形式: `### 日時 [root@STG] $ コマンド` → 出力 → `### exit=終了コード`
- WAF テストの curl では、Basic 認証のパスワードを変数（`$BA`）にしているため、記録には残っていない

## 7. 残っていること

- [ ] ブラウザで管理画面（`https://stg.centrair-sake.com:9443/`）にログインし、[状態] のバージョンと設定を確認する
- [ ] 手順8（シグネチャの更新先URLの修正・更新、php-fpm を起動してのサイト確認）を行うか決める
- [x] **元に戻す**（手順書 9-1 → 9-2 → 9-4）。16:14〜16:15 に実施（下の「8. 元に戻した記録」）
- [ ] 上の「5. 分かったこと」を手順書と PROD の手順に反映する

---

## 8. 元に戻した記録（2026-11-10 16:14〜16:15）

徳原さんの指示で、手順書の 9-2 → 9-1 → 9-4 の順に、STG を更新前の状態に戻した。

### 8-1. 実施した作業

| 時刻 | 手順 | 内容 |
|---|---|---|
| 16:14:24 | ― | 戻す前の状態を記録（8.00 Update 5、nginx `30cac987…`） |
| 16:14:25 | 9-2 | `siteguardlite_admin` / `_notify` を停止。`/opt/jp-secure/siteguardlite` を `siteguardlite.sg5_rollback` に移し、`siteguardlite_full_8.00-3.tar.gz` から展開。ログと統計は移したフォルダからコピー（作業中のログも残すため） |
| 16:14:27 | 9-2 | systemd のユニット2つは退避と同じだったので、そのまま。`daemon-reload` |
| 16:14:28 | 9-1 | `nginx.sg3_20261110` から nginx を戻した（sha256 `c8cc95e8…`）。`nginx -t` → `systemctl restart nginx129` |
| 16:14:29 | 9-2 | `siteguardlite_admin` / `_notify` を起動 |
| 16:14:52 | 9-4 | `dnf versionlock delete kusanagi-nginx129` |
| 16:15:04 | 9-4 | OpenSSL のリンク（`libssl.so`、`libcrypto.so`）を削除。`/usr/local/src/build-sg5` と、展開したパッケージを削除。`nginx.sg3_20261110` を削除（戻した nginx と同じもの） |
| 16:15:04〜11 | ― | 更新前の記録と突き合わせ（下の表） |
| 16:15:38 | ― | 退避していた Update5 のフォルダ（`siteguardlite.sg5_rollback`、238MB）を削除 |

### 8-2. 更新前の記録との突き合わせ

| 確認したもの | 結果 | 確認方法 |
|---|---|---|
| SiteGuard のバージョン | **8.00 Update 3 [64_1.29]**（更新前と同じ） | `make show_webui_version` の結果を `version_before.txt` と diff → 一致 |
| nginx のバイナリ | **`c8cc95e8…e2c25f`**（更新前と同じ） | `sha256sum` の結果を `nginx_sha_before.txt` と diff → 一致 |
| nginx -V | 更新前と同じ | `nginx-V_before.txt` と diff → 一致 |
| `/opt/jp-secure/siteguardlite` の中身 | 退避した tar と**内容が一致**。違いは、サービス起動時に書き直される設定ファイル（`configured.module`、`tomcat/conf/server.xml`、`web.xml` など）の**更新時刻**と、pid ファイルだけ | `tar dzf siteguardlite_full_8.00-3.tar.gz -C /opt/jp-secure` |
| SiteGuard モジュールのソース | 2025-11-17 / 2024-09-25 版（更新前と同じ） | `ls -la …/siteguardlite/nginx/` |
| シグネチャ | 2025-10-23_01 | `header.ini` |
| 更新先URL | `_sp` なし（更新前と同じ） | `dbupdate_waf_url.conf` |
| `statistics` / ini の version | `no` / `Nginx800-3` | `siteguardlite.ini` |
| crontab | 更新前と同じ | `crontab_before.txt` と diff → 一致 |
| versionlock | なし（更新前と同じ） | `dnf versionlock list`、`versionlock.list` |
| OpenSSL のフォルダ | ファイルは更新前と同じ。違いはフォルダ自体の更新時刻だけ | `openssl_lib64_before.txt` と diff |
| サービス | nginx129=active、php-fpm=inactive（更新前と同じ）。siteguardlite_admin / _notify=active | `services_before.txt` と diff → 一致 |
| nginx.conf / systemd のユニット | 更新前と同じ | diff → 一致 |
| `/WAF-TEST-SIGNATURE` | 403。detect.log に 16:15:09 の検出が記録された | curl、`tail detect.log` |
| 管理画面（9443） | 200 | curl |
| `nginx -t` | OK | |

- 16:15:09 の確認では、curl の直後に `tail detect.log` を見たため、まだ書き込まれる前で、前回（16:03）の行が表示された。16:15:38 に見直して、16:15:09 の検出が記録されていることを確認した（`13_rollback_detectlog.log`、`14_rollback_cleanup.log`）

### 8-3. 完全には元に戻っていないもの

| もの | 内容 | 影響 |
|---|---|---|
| ファイルの更新時刻 | `/opt/jp-secure/siteguardlite` の設定ファイルの一部、`/etc/opt/kusanagi/nginx/nginx.conf`・`nginx.conf.bak`、systemd のユニット、各フォルダの更新時刻が 11/10 になっている | なし（中身は同じ） |
| `nginx.conf.bak` | make install で作り直された（3285 バイトで大きさは更新前と同じ）。更新前の .bak そのものは退避していなかったので、中身の完全一致は確認できていない | なし（nginx は使っていない） |
| ログ | 作業中の SiteGuard のログ（detect.log、info.log など）、nginx・monit のログが増えている | なし |
| `/root/sg5_work/` | 記録と退避（SiteGuard 一式の tar 100MB を含む）を残している | 次の作業でも使う。不要になったら消す |
| `/usr/local/src/siteguard-8.00-5/` | Update5 のパッケージ（tar.gz と sha256.txt）。更新前の時点でもあったもの | なし |
| `/opt/jp-secure/siteguardlite/backup.tar.gz` | 手順2の `make backup` で作り直された（Update3 の設定。中身は 14:22 のものと同じはず） | なし |

### 8-4. エビデンス

| ファイル | 中身 |
|---|---|
| `11_rollback_restore.log` | 9-2・9-1（SiteGuard 本体と nginx を戻す） |
| `12_rollback_verify.log` | 9-4 と、更新前の記録との突き合わせ |
| `13_rollback_detectlog.log` | 検出ログの書き込み先の確認 |
| `14_rollback_cleanup.log` | 検出ログの再確認、退避フォルダの削除、最終状態 |
| `sg5_work_STG/` | 戻した後の記録（`*_after_rollback.txt`）を含めて取り直した |
