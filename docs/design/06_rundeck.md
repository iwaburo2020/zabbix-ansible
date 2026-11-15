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

### 5.1 raspberrypi-desktop(cron のまま、月1回にまとめる)

| 処理 | 実施日 | 時刻 |
|---|---|---|
| パッケージ更新(`package_fullupdate.sh`) | 第3土曜(15〜21日の土曜) | 14:00 |
| フラグ作成(`make_flagfile.sh`) | 第3土曜の翌日(16〜22日の日曜) | 1:58 |
| 再起動(`stop.sh reboot`) | 第3土曜の翌日(16〜22日の日曜) | 1:59 |
| バックアップ(`backup_file.sh`) | 第3土曜の翌日(16〜22日の日曜) | 3:00 |

- 決定の経緯:生成AIの推奨案で確定(2026-11-03)
- 第3土曜とその翌日の日曜は、他ホストのパッケージ更新(第2土曜・第4土曜・月末日)と再起動(9〜15日の日曜、23〜29日の日曜、1日)の日と重ならない
- パッケージ更新は、月3回(第2土曜・第4土曜・月末日)から月1回になる
- バックアップの現在の定義は root の crontab:`00 3 1 * *   /usr/local/script/backup_file.sh   >> /var/log/backup_file.log`(2026-11-03 に `crontab -l` で確認)

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

### 7.3 ノード(7台)

ntp001、ntp002、ntp003、ntp004、web001b、web002b、backup001(backup001 の接続先は 192.168.200.22 のポート 2222)

### 7.4 ジョブ(19個、すべてスケジュール有効)

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

- フラグ作成・再起動のジョブは、予定の時刻より前に起動し、対象ノードの時計で予定の時刻まで待ってから実行する。予定の時刻を過ぎていても実行する。フラグ作成が失敗しても再起動は実行する(cron と同じ動き)
- 日曜のジョブは毎週起動し、日付が条件に合わない日は `skip: day <日>` と表示して終わる

### 7.5 ジョブのコマンドの書き方(実機で確認)

- Rundeck のコマンドのステップは、引用符を外して実行する。引用符が必要なコマンド(`sudo /bin/sh -c '…'` など)は使えない
- `>`、`;`、`|`、`$( )`、`$(( ))`、`case`、`set -o pipefail` は使える
- パッケージ更新:`sudo /bin/sh -c /usr/local/script/package_update.sh > /dev/null`
- バックアップ:`set -o pipefail; sudo /usr/local/script/backup_file_nfs.sh <ホスト> | sudo tee -a /var/log/backup_file_<ホスト>.log > /dev/null`(cron の `>>` と同じログファイルに追記する。スクリプトのエラー出力は Rundeck の実行ログに残る)

### 7.6 cron の変更

| ホスト | ファイル | 変更 | 変更前のコピー |
|---|---|---|---|
| ntp004 | `/etc/crontab` | パッケージ更新3行、フラグ作成・再起動5行をコメントにした | `/etc/crontab.bk20261113`、`/etc/crontab.bk20261114` |
| ntp001、ntp002、ntp003、web001b、web002b | `/etc/crontab` | パッケージ更新3行、フラグ作成・再起動5行をコメントにした | `/etc/crontab.bk20261114` |
| backup001 | root の crontab | 毎月1日の `backup_file_nfs.sh` 4行(ntp002・ntp003・web001・web002)をコメントにし、毎月15日の2行(web001・web002)を削除した(2026-11-15) | なし(crontab コマンドで変更) |

### 7.7 実行結果

| 日付 | 内容 | 結果 |
|---|---|---|
| 2026-11-14 | パッケージ更新(第2土曜、15:00 と 16:00) | 6台とも成功 |
| 2026-11-15 | フラグ作成・再起動(日曜) | 4つのジョブとも成功。Zabbix の「has been restarted」の検知時刻は、cron で実行した 2026-11-01 とほぼ同じ(差は最大10秒) |

まだ実行されていないジョブ:第4土曜と月末日のパッケージ更新(初回 2026-11-28、2026-11-30)、毎月1日のフラグ作成・再起動とバックアップ(初回 2026-12-01)

## 8. 引き継ぎ課題(2026-11-15 時点)

| No | 内容 |
|---|---|
| 1 | NFS ブートのサーバー(ntp002、ntp003、ntp004、web001b、web002b)のうち数台が、予定の再起動のあと 5:00 ごろにもう一度再起動している。Zabbix の検知は、2026-11-15 が ntp002・ntp003・web001b・web002b(5:02 ごろ)、2026-11-01 が ntp002・ntp004・web001b・web002b(5:02〜5:03 ごろ)、2026-10-25 が ntp002(5:01)。cron で運用していたときからある動きで、Rundeck への移行が原因ではない。原因は未調査 |
| 2 | ntp001 のバックアップ(root の crontab、毎月1日 2:00、`backup_file.sh`)は cron のまま。raspberrypi-desktop が毎月1日の 1:59 に再起動するため、2:00 の時点では Rundeck が動いておらず、2:00 のジョブは実行されない。5.1 の raspberrypi-desktop の定期処理の変更は未実施 |
| 3 | 2026-12-01 の初回の実行結果の確認(毎月1日のフラグ作成・再起動のジョブ4つ、バックアップのジョブ5つ)。`backup_file_nfs.sh` を Rundeck から実行するのは初めてで、引数 `ntp004` での実行も初めて |
| 4 | 2026-11-28(第4土曜)と 2026-11-30(月末日)のパッケージ更新のジョブの初回の実行結果の確認 |
| 5 | 残りのサーバーのジョブ化(raspberrypi-zabbix、dns001b、dns002b、cache001b、dhcp001b、nas001b、proxy001b、gps001b、wap001b、backup001 自身、logbackup001)。backup001 自身のバックアップ(毎月1日 1:50)も cron のまま |
| 6 | 6台の `/usr/local/script` 以外の定期実行(`lan.sh` など)は、4章の決定どおり cron のまま |
