# 06. Rundeck(ジョブ管理ツール)

## 1. 導入の判断

- 判断日:2026-11-03
- 結果:導入する

## 2. 公式資料で確認した内容(2026-11-03)

| 項目 | 内容 |
|---|---|
| 版 | 6.3.0(Debian/Ubuntu 用は `rundeck_6.3.0-20261022-1_all.deb`。CPU に依存しない Java アプリ) |
| Java | 17 以上 25 まで。Ubuntu 26.04 amd64 には openjdk-21-jre-headless がある |
| 公式の最小構成 | 2 CPU、メモリ 8GB(Java に 4GB)。専用サーバーを前提とした構成で、どの候補ホストも満たさない |
| Java ヒープの初期設定 | 最大 1024MB・初期 256MB(インストーラーの初期設定) |
| DB | H2(組み込み DB)はテスト用。本番は PostgreSQL などを使う |
| 接続方式 | Rundeck サーバーから各ホストへ SSH で接続してコマンドを実行する。管理されるホストには何もインストールしない |
| スケジュール | Quartz 形式の cron 式。第2土曜は `7#2`、第4土曜は `7#4`、月末日は `L` で書ける。「第2・第4土曜の翌日(日曜)」は cron 式1つでは書けない |

## 3. 候補ホストのメモリ(2026-11-03 実測、free -m)

| ホスト | total | available |
|---|---|---|
| raspberrypi-zabbix | 7818MB | 7220MB |
| raspberrypi-desktop | 7116MB | 6312MB(/ の空き 44G) |
| humidai002 | 8058MB | 7586MB |

## 4. 決定事項(2026-11-03)

| 項目 | 内容 |
|---|---|
| Rundeck を動かすホスト | raspberrypi-desktop。Java のヒープは初期設定(最大 1024MB)で開始し、導入後のメモリの状態は Zabbix(Linux by Zabbix agent)で確認する |
| DB | raspberrypi-desktop の既存の PostgreSQL 18 に、Zabbix とは別のデータベース `rundeck` を作る |
| SSH 接続 | Rundeck 用の SSH 鍵を新しく作る。公開鍵は Ansible の inventory と同じユーザー・ポートで、各ホストへ Ansible で配る |
| ジョブにする処理(初回) | パッケージ更新、OS 再起動、再起動に必要なフラグ作成(`make_flagfile.sh`)、月次バックアップ(backup001 が NFS ブートのホストのバックアップを取る処理を含む) |
| ジョブにしない処理(初回) | `lan.sh` などのその他の定期実行、OS 標準・パッケージ由来の定義、端末機2台(humidai001b、humidai002)。crontab のまま運用する |
| 移行の方針 | 稼働後、ジョブに移行できる処理は段階的にジョブへ移行する(すべては移行しない) |
| raspberrypi-zabbix | ジョブにする(元の設計どおり) |
| raspberrypi-desktop | cron のまま運用する。Rundeck 本体と DB が動くため、ジョブで再起動すると実行中に Rundeck が止まり、結果を記録できない |
| crontab の変更時期 | Rundeck の導入作業の中で、ジョブへの切り替えと同じタイミングで行う |

## 5. 定期処理の変更(ジョブへの切り替えと同じタイミングで反映)

### 5.1 raspberrypi-desktop(cron のまま、月末日だけ実行する)

| 処理 | 実施日 | 時刻 |
|---|---|---|
| パッケージ更新(`package_fullupdate.sh`) | 月末日 | 20:00 |
| フラグ作成(`make_flagfile.sh`) | 月末日 | 21:58 |
| 再起動(`stop.sh reboot`) | 月末日 | 21:59 |
| バックアップ(`backup_file.sh`) | 月末日 | 23:00 |

- 決定の経緯:2026-11-03 に生成AIの推奨案で「第3土曜とその翌日」と記録していたが、利用者の希望は「月1回、月末」だった。2026-11-15 に上の内容を利用者が承認し、同日に実施した(`rundeck_desktop_cron_monthly.yml`)
- 日曜と毎月1日に再起動しなくなったため、同じ時刻の Rundeck のジョブ(1:57 起動のフラグ作成・再起動、毎月1日 2:00 の ntp001 のバックアップ)と重ならない
- 月末日の 20:00〜22:00 に起動する Rundeck のジョブは無い(パッケージ更新は 12:00〜18:00 に起動する)
- バックアップは、利用者の指示で再起動の後にした(今までと同じく再起動の61分後)
- パッケージ更新は、月3回(第2土曜・第4土曜・月末日)から月1回になった
- `/etc/crontab`:今までの8行は行頭 `#` でコメントにし、月末日の3行を追加した。月末日の判定は `test $( date -d '+1 day' +\%d ) -eq 1`。変更前のコピーは `/etc/crontab.bk20261115`
- root の crontab:`00 3 1 * *` の `backup_file.sh` の行を `00 23 28-31 * * test $( date -d '+1 day' +\%d ) -eq 1 && /usr/local/script/backup_file.sh >> /var/log/backup_file.log` に置き換えた

### 5.2 backup001 の web001b・web002b の毎月15日のバックアップ

- 廃止する(2026-11-03 決定)。毎月15日のジョブは作らない
- backup001 の root の crontab にある毎月15日の2行(web001 5:00、web002 5:30)は、ジョブへの切り替えと同じタイミングで削除する
- 毎月1日のバックアップ(ntp002 3:30、ntp003 4:00、web001 5:00、web002 5:30)は、月次バックアップとしてジョブにする

## 6. 未決定事項

- ジョブの詳細設計(対象ホスト、ジョブの分け方、スケジュールの書き方、再起動ジョブの扱いなど)

## 7. 導入と移行の結果(2026-11-10〜2026-11-15)

### 7.1 Rundeck 本体

| 項目 | 内容 |
|---|---|
| ホスト | raspberrypi-desktop(`http://192.168.200.4:4440`) |
| 版 | Rundeck 6.3.0(apt のパッケージ `rundeck`。hold で固定)、Java は openjdk 17 |
| DB | raspberrypi-desktop の PostgreSQL 18 のデータベース `rundeck`(接続ユーザー `rundeck`) |
| 自動起動 | `rundeckd`。raspberrypi-desktop の再起動後に自動で起動することを確認(2026-11-15) |
| API トークン | humidai002 の `~/.rundeck_api_token`(権限 600)。リポジトリには含めない |
| プロジェクト | `infra`。ノードは inventory のグループ `rundeck_nodes` から作る |
| SSH | Rundeck 用の鍵 `/var/lib/rundeck/.ssh/id_rsa`。公開鍵は各ノードの接続ユーザーの `authorized_keys` に追加 |

### 7.2 playbook(実行順)

| playbook | 内容 |
|---|---|
| `rundeck_db.yml` | データベースと接続ユーザーの作成 |
| `rundeck_server.yml` | Java と Rundeck 本体の導入 |
| `rundeck_admin.yml` | admin のパスワード変更 |
| `rundeck_ssh.yml` | SSH 鍵の作成と、`rundeck_nodes` への公開鍵の配布 |
| `rundeck_api_token.yml` | API トークンの設定 |
| `rundeck_project.yml` | プロジェクトの作成とノードの登録(ノードを増やしたときは再実行する) |
| `rundeck_node_check.yml` | Rundeck から全ノードへコマンドを実行できるかの確認 |
| `rundeck_jobs.yml` | ジョブの登録・更新(定義は `vars/rundeck_jobs.yml`) |
| `rundeck_jobs_next.yml` | 各ジョブの次回実行と直近の実行結果の表示 |
| `rundeck_cron_package_off.yml` | `/etc/crontab` のパッケージ更新の行をコメントにする |
| `rundeck_cron_reboot_off.yml` | `/etc/crontab` のフラグ作成・再起動の行をコメントにする |
| `rundeck_cron_backup_off.yml` | backup001 の root の crontab のバックアップの行を止める |
| `rundeck_cron_backup_self_off.yml` | 指定したホストの root の crontab で、自分自身のバックアップ(毎月1日の `backup_file.sh`)の行をコメントにする |
| `rundeck_desktop_cron_monthly.yml` | raspberrypi-desktop の定期処理を月末日だけの実行に変更する(5.1) |
| `rundeck_nas_cron_reboot_time.yml` | nas001b のフラグ作成・再起動を 0:28・0:29 に変更する(9章) |
| `rundeck_proxy_cron_nfs_time.yml` | proxy001b の `chk_umnt.sh`・`chk_mnt.sh` を 0:20・0:40 に変更する(9章) |

### 7.3 ノード(7台)

ntp001、ntp002、ntp003、ntp004、web001b、web002b、backup001(backup001 の接続先は 192.168.200.22 のポート 2222)

### 7.4 ジョブ(20個、すべてスケジュール有効)

| ジョブ名 | 起動 | 対象ノード | 内容 |
|---|---|---|---|
| `update-2sat-1500`、`update-4sat-1500` | 第2土曜・第4土曜 15:00 | ntp001、ntp002、ntp003 | `package_update.sh` |
| `fullupdate-eom-1500` | 月末日 15:00 | ntp001、ntp002、ntp003 | `package_fullupdate.sh` |
| `update-2sat-1600`、`update-4sat-1600` | 第2土曜・第4土曜 16:00 | ntp004、web001b、web002b | `package_update.sh` |
| `fullupdate-eom-1600` | 月末日 16:00 | ntp004、web001b、web002b | `package_fullupdate.sh` |
| `flag-reboot-sun-0058` | 日曜 0:57 | ntp001 | 0:58 に `make_flagfile.sh`(9〜15日)、0:59 に `stop.sh reboot`(9〜15日・23〜29日) |
| `flag-reboot-sun-0158` | 日曜 1:57 | ntp002 | 同じ処理を 1:58・1:59 に実行 |
| `flag-reboot-sun-0258` | 日曜 2:57 | ntp003 | 同じ処理を 2:58・2:59 に実行 |
| `flag-reboot-sun-0358` | 日曜 3:57 | ntp004、web001b、web002b | 同じ処理を 3:58・3:59 に実行 |
| `flag-reboot-1st-0058`〜`0358`(4個) | 毎月1日 0:57・1:57・2:57・3:57 | 日曜のジョブと同じ | x:58 に `make_flagfile.sh`、x:59 に `stop.sh reboot` |
| `backup-nfs-ntp002-1st-0330` | 毎月1日 3:30 | backup001 | `backup_file_nfs.sh ntp002` |
| `backup-nfs-ntp003-1st-0400` | 毎月1日 4:00 | backup001 | `backup_file_nfs.sh ntp003` |
| `backup-nfs-ntp004-1st-0430` | 毎月1日 4:30 | backup001 | `backup_file_nfs.sh ntp004`(cron には定義が無かった。時刻はスケジュール表の 4:30) |
| `backup-nfs-web001-1st-0500` | 毎月1日 5:00 | backup001 | `backup_file_nfs.sh web001` |
| `backup-nfs-web002-1st-0530` | 毎月1日 5:30 | backup001 | `backup_file_nfs.sh web002` |
| `backup-ntp001-1st-0200` | 毎月1日 2:00 | ntp001 | `backup_file.sh`(ntp001 自身のバックアップ。2026-11-15 に追加) |

- フラグ作成・再起動のジョブは、予定の時刻より前に起動し、対象ノードの時計で予定の時刻まで待ってから実行する。予定の時刻を過ぎていても実行する。フラグ作成が失敗しても再起動は実行する(cron と同じ動き)
- 日曜のジョブは毎週起動し、日付が条件に合わない日は `skip: day <日>` と表示して終わる

### 7.5 ジョブのコマンドの書き方(実機で確認)

- Rundeck のコマンドのステップは、引用符を外して実行する。引用符が必要なコマンド(`sudo /bin/sh -c '…'` など)は使えない
- `>`、`;`、`|`、`$( )`、`$(( ))`、`case`、`set -o pipefail` は使える
- パッケージ更新:`sudo /bin/sh -c /usr/local/script/package_update.sh > /dev/null`
- バックアップ:`set -o pipefail; sudo /usr/local/script/backup_file_nfs.sh <ホスト> | sudo tee -a /var/log/backup_file_<ホスト>.log > /dev/null`(cron の `>>` と同じログファイルに追記する。スクリプトのエラー出力は Rundeck の実行ログに残る)
- 自分自身のバックアップ:`set -o pipefail; sudo /usr/local/script/backup_file.sh | sudo tee -a /var/log/backup_file.log > /dev/null`

### 7.6 cron の変更

| ホスト | ファイル | 変更 | 変更前のコピー |
|---|---|---|---|
| ntp004 | `/etc/crontab` | パッケージ更新3行、フラグ作成・再起動5行をコメントにした | `/etc/crontab.bk20261113`、`/etc/crontab.bk20261114` |
| ntp001、ntp002、ntp003、web001b、web002b | `/etc/crontab` | パッケージ更新3行、フラグ作成・再起動5行をコメントにした | `/etc/crontab.bk20261114` |
| backup001 | root の crontab | 毎月1日の `backup_file_nfs.sh` 4行(ntp002・ntp003・web001・web002)をコメントにし、毎月15日の2行(web001・web002)を削除した(2026-11-15) | なし(crontab コマンドで変更) |
| ntp001 | root の crontab | 毎月1日 2:00 の `backup_file.sh` の行をコメントにした(2026-11-15) | なし(crontab コマンドで変更) |
| raspberrypi-desktop | `/etc/crontab`、root の crontab | 月末日だけの実行に変更した(2026-11-15。内容は 5.1) | `/etc/crontab.bk20261115` |
| nas001b | `/etc/crontab` | フラグ作成2行を 4:58 から 0:28 に、再起動3行を 4:59 から 0:29 に変更した(2026-11-15) | `/etc/crontab.bk20261115` |
| proxy001b | root の crontab | `chk_umnt.sh` を毎週日曜 4:50 から 0:20 に、`chk_mnt.sh` を毎週日曜 5:10 から 0:40 に変更した(2026-11-15) | なし(crontab コマンドで変更) |

### 7.7 実行結果

| 日付 | 内容 | 結果 |
|---|---|---|
| 2026-11-14 | パッケージ更新(第2土曜、15:00 と 16:00) | 6台とも成功 |
| 2026-11-15 | フラグ作成・再起動(日曜) | 4つのジョブとも成功。Zabbix の「has been restarted」の検知時刻は、cron で実行した 2026-11-01 とほぼ同じ(差は最大10秒) |

まだ実行されていないジョブ:第4土曜と月末日のパッケージ更新(初回 2026-11-28、2026-11-30)、毎月1日のフラグ作成・再起動とバックアップ(初回 2026-12-01。ntp001 のバックアップを含む)

変更後の cron の初回の実行:nas001b の再起動 2026-11-29 0:29(proxy001b の `chk_umnt.sh` 0:20、`chk_mnt.sh` 0:40)、raspberrypi-desktop の更新・再起動・バックアップ 2026-11-30 20:00〜23:00

## 8. 引き継ぎ課題(2026-11-15 時点)

| No | 内容 |
|---|---|
| 1 | 2026-11-29(日)の結果の確認。nas001b が 0:29 に再起動したこと、NFS ブートのサーバー(ntp002、ntp003、ntp004、web001b、web002b)の再起動の時刻(Zabbix の「has been restarted」)、5:00 ごろの再起動が無くなったこと |
| 2 | 2026-11-30(月末日)の結果の確認。パッケージ更新のジョブの初回の実行結果(2026-11-28 の第4土曜の分を含む)、raspberrypi-desktop の 20:00 の更新・21:59 の再起動・23:00 のバックアップ |
| 3 | 2026-12-01 の初回の実行結果の確認(毎月1日のフラグ作成・再起動のジョブ4つ、バックアップのジョブ6つ)。`backup_file_nfs.sh` と `backup_file.sh` を Rundeck から実行するのは初めてで、引数 `ntp004` での実行も初めて |
| 4 | 残りのサーバーのジョブ化(raspberrypi-zabbix、dns001b、dns002b、cache001b、dhcp001b、nas001b、proxy001b、gps001b、wap001b、backup001 自身、logbackup001)。backup001 自身のバックアップ(毎月1日 1:50)も cron のまま |
| 5 | 6台の `/usr/local/script` 以外の定期実行(`lan.sh` など)は、4章の決定どおり cron のまま |

## 9. NFS ブートのサーバーが 5:00 ごろにもう一度再起動していた件(2026-11-15)

### 9.1 事象

NFS ブートのサーバー(ntp002、ntp003、ntp004、web001b、web002b)のうち数台が、予定の再起動のあと 5:00 ごろにもう一度再起動していた。Zabbix の検知は、2026-11-15 が ntp002・ntp003・web001b・web002b(5:02 ごろ)、2026-11-01 が ntp002・ntp004・web001b・web002b(5:02〜5:03 ごろ)、2026-10-25 が ntp002(5:01)。cron で運用していたときからある動きで、Rundeck への移行が原因ではない。

### 9.2 確認した事実

| 項目 | 内容 |
|---|---|
| nas001b の再起動 | 4:59 に定期再起動し、2026-11-15 は 5:01 から起動の記録がある(利用者確認) |
| NFS ブートの4台の起動時刻 | 2026-11-15 の 5:00:36〜5:01:39(`uptime -s`) |
| watchdog の設定 | ntp001〜ntp004、web001b、web002b の6台とも `RuntimeWatchdogUSec=1min`、`RebootWatchdogUSec=2min`。dmesg に `Watchdog running with a hardware timeout of 1min.` |
| 設定の出どころ | `/usr/lib/systemd/system.conf.d/40-rpi-enable-watchdog.conf`(OS 標準のファイル)。`/etc` 側に追加の設定は無い |
| ntp001 | NFS ブートではない(microSD 起動)。watchdog の設定は同じだが、5:00 ごろの再起動は無い |

### 9.3 原因の判断

nas001b が再起動で止まっている約2分の間、`/` を nas001 に置いている NFS ブートのサーバーは `/` を読み書きできない。その間に systemd が1分を超えて止まると、watchdog が強制的に再起動する。時刻・対象・設定が一致するため、原因は nas001b の定期再起動と判断した。

未確認:強制的な再起動だったことを直接示すログ、日によって再起動しない台がある理由。

### 9.4 対応(2026-11-15 に利用者が承認、同日実施)

| 項目 | 内容 |
|---|---|
| watchdog | 変更しない(OS 標準の設定のまま) |
| nas001b | フラグ作成・再起動を 4:58・4:59 から 0:28・0:29 に変更した。NFS ブートのサーバーの予定の再起動(1:59〜3:59)より前に nas001b を再起動する |
| proxy001b | nas001b の再起動に合わせた `chk_umnt.sh`・`chk_mnt.sh`(毎週日曜)を、4:50・5:10 から 0:20・0:40 に変更した |

- 毎月1日 5:00 のバックアップ(web001、dhcp001b)が、nas001b の再起動(今までは 4:59)の直後に始まる状態も無くなった
- NFS ブートのサーバーが nas001b の再起動時(0:30 ごろ)に再起動すること自体は残る。その後に予定の再起動を行うため、予定の再起動のあとにもう一度再起動することは無くなる
- 23〜29日の日曜は、NFS ブートのサーバーの予定の再起動が行われない場合がある。0:30 ごろに再起動すると起動時に `/etc/rc.local` が `/var/tmp/reboot` を削除し、この日は `make_flagfile.sh` を実行しないため(`stop.sh reboot` は `/var/tmp/reboot` があるときだけ再起動する)。その場合は、0:30 ごろの再起動が更新後の再起動になる
