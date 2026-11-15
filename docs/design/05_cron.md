# 05. 定期処理一覧(cron 棚卸し)

## 1. 棚卸しの概要

- 実施日:2026-10-25
- 対象:20台(Linux のサーバー18台と端末機2台。proxyvm001b は Windows のため対象外)
- 確認したファイル:`/etc/crontab`、`/etc/cron.d/` 配下、root の crontab(`/var/spool/cron/crontabs/root`、logbackup001 は `/var/spool/cron/root`)
- 基準:ユーザーが管理するスケジュール表(crontabスケジュール.xlsx)

## 2. 実施日の基準(スケジュール表)

| 処理 | 実施日 | cron の日付条件 |
|---|---|---|
| パッケージ更新 | 第2土曜、第4土曜 | 8〜14日の土曜、22〜28日の土曜(`package_update.sh`) |
| パッケージ更新 | 月末日 | 28〜31日のうち翌日が1日の日(`package_fullupdate.sh`) |
| 再起動(翌日に行うホスト) | パッケージ更新の翌日 | 9〜15日の日曜、23〜29日の日曜、1日 |
| 再起動(当日に行うホスト) | パッケージ更新の当日 | 8〜14日の土曜、22〜28日の土曜、月末日 |
| バックアップ | 毎月1日 | 1日 |

- 再起動(`stop.sh reboot`)の1分前に `make_flagfile.sh` を実行する。

## 3. ホスト別の定期処理

月末日の再起動は、「翌日」のホストは1日、「当日」のホストは月末日の同じ時刻に行う。

2026-11-15 の変更(8章)を反映済み。

| ホスト | パッケージ更新 | 再起動 | バックアップ(毎月1日) |
|---|---|---|---|
| gps001b | 12:00 | 23:20(当日) | 1:00 |
| dns001b | 13:00 | 0:59(翌日) | 2:00 |
| dns002b | 13:00 | 1:59(翌日) | 3:00 |
| cache001b | 14:00 | 2:59(翌日) | 4:00 |
| dhcp001b | 15:00 | 3:59(翌日) | 5:00 |
| backup001 | 12:00 | 23:29(当日) | 1:50 |
| wap001b | 12:00 | 23:59(当日) | 1:00 |
| nas001b | 17:00 | 0:29(翌日) | 6:00 |
| proxy001b | 18:00 | 5:59(翌日) | 7:00 |
| ntp001 | 15:00 | 0:59(翌日) | 2:00 |
| ntp002 | 15:00 | 1:59(翌日) | 3:30(backup001 から取得) |
| ntp003 | 15:00 | 2:59(翌日) | 4:00(backup001 から取得) |
| ntp004 | 16:00 | 3:59(翌日) | 4:30(backup001 から取得) |
| logbackup001 | cron の定義なし | 23:59(月末日のみ) | なし |
| raspberrypi-desktop | 20:00(月末日のみ) | 21:59(月末日のみ、当日) | 23:00(月末日) |
| raspberrypi-zabbix | 14:00 | 1:59(翌日) | 3:00 |
| web001b | 16:00 | 3:59(翌日) | 5:00(backup001 から取得) |
| web002b | 16:00 | 3:59(翌日) | 5:30(backup001 から取得) |
| humidai001b | 12:00(毎週土曜) | 23:59(毎週土曜、当日) | 0:00 |
| humidai002 | 12:00(毎週土曜) | 23:59(毎週土曜、当日) | 0:00 |

## 4. スケジュール表との違い(記録のみ、修正しない)

| ホスト | スケジュール表 | 実機 |
|---|---|---|
| gps001b | 再起動 0:00(翌日) | 23:20(当日)。root の crontab で `system_restart.sh` を毎時59分に実行 |
| ntp002 | バックアップ 3:00 | 3:30(backup001 から取得) |
| web001b | バックアップ 毎月1日 | 1日と15日 |
| web002b | バックアップ 5:00(毎月1日) | 5:30(1日と15日) |
| ntp004 | バックアップ 4:30 | 定義なし |
| backup001 | バックアップ 1:00 | 1:50(root の crontab で `backup_file.sh` を実行) |
| proxy001b | (記載なし) | 第4土曜の翌日の再起動の前にも `make_flagfile.sh` を実行(他のホストは第2土曜の翌日と1日のみ) |
| humidai001b、humidai002 | パッケージ更新・再起動 第2土曜・第4土曜 | 毎週土曜(パッケージ更新 12:00、再起動 23:59)。従来どおりとする |

## 5. 修正対象(2026-11-07 までに修正)

| No | ホスト | ファイル | 修正内容 | 状態 |
|---|---|---|---|---|
| 1 | raspberrypi-zabbix、raspberrypi-desktop、dns001b、dhcp001b、nas001b、proxy001b、ntp001、ntp002、ntp003、ntp004、web001b、web002b、cache001b、dns002b | `/etc/crontab` | 翌日(日曜)の再起動と `make_flagfile.sh` の日付範囲を「9-15」「23-29」にする | 対応済み(2026-10-25) |
| 2 | cache001b(21-28)、web001b(22-29) | `/etc/crontab` | 第4土曜のパッケージ更新の日付範囲を「22-28」にする | 対応済み(2026-10-25) |
| 3 | wap001b | `/etc/crontab` | 第4土曜のパッケージ更新と再起動の日付範囲「22-27」を「22-28」にする | 対応済み(2026-10-25) |
| 4 | ntp001 | root の crontab | 毎週土曜のパッケージ更新、毎週日曜の再起動と `make_flagfile.sh` の3行をコメントアウトする | 対応済み(2026-10-25) |
| 5 | wap001b | root の crontab | バックアップの出力先を `>> /var/log/backup_file.log` にする | 対応済み(2026-10-25) |
| 6 | web001b、web002b | `/etc/crontab` | 第2土曜の翌日の `make_flagfile.sh` を 3:59 から 3:58 にする | 対応済み(2026-10-25) |

## 6. スケジュール表にない定期実行(記録のみ)

| ホスト | 内容 |
|---|---|
| dns001b、dns002b、proxy001b | `/usr/local/script/lan.sh` を5分ごと |
| web001b、web002b、gps001b | `/usr/local/script/lan.sh` を30分ごと |
| wap001b | `/home/script/lan.sh` を30分ごと |
| dns001b | `/usr/local/script/roothints.sh` を毎週土曜 23:59 |
| dns002b | `/usr/local/script/roothints.sh` を毎週日曜 23:59 |
| proxy001b | root の crontab で、`/home/script/chk_umnt.sh` を毎週日曜 0:20、`/home/script/chk_mnt.sh` を毎週日曜 0:40(nas001b の再起動に合わせた処理。2026-11-15 に 4:50・5:10 から変更) |
| gps001b | `/usr/local/script/system_restart.sh` を毎時59分 |
| logbackup001 | `/usr/bin/chronyc makestep` を30分ごと |
| backup001 | NFS ブートのホストのバックアップ(`backup_file_nfs.sh`)は、2026-11-15 に Rundeck のジョブへ移行した(ntp002 3:30、ntp003 4:00、ntp004 4:30、web001 5:00、web002 5:30、毎月1日)。root の crontab の毎月1日の4行はコメントにし、毎月15日の2行(web001・web002)は削除した |

## 7. OS 標準・パッケージ由来の定義(変更しない)

- `/etc/crontab` の `cron.hourly`、`cron.daily`、`cron.weekly`、`cron.monthly` の4行
- `/etc/cron.d/e2scrub_all`(logbackup001 以外の全ホスト)
- `/etc/cron.d/php`(raspberrypi-zabbix)
- `/etc/cron.d/sysstat`(dhcp001b)
- `/etc/cron.d/0hourly`(logbackup001)

## 8. 棚卸し後の変更(2026-11-15)

| ホスト | 変更前 | 変更後 | playbook |
|---|---|---|---|
| raspberrypi-desktop | パッケージ更新:第2土曜・第4土曜・月末日 14:00。再起動:9〜15日の日曜・23〜29日の日曜・毎月1日 1:59。バックアップ:毎月1日 3:00 | 月末日だけ実行する。20:00 `package_fullupdate.sh`、21:58 `make_flagfile.sh`、21:59 `stop.sh reboot`、23:00 `backup_file.sh` | `rundeck_desktop_cron_monthly.yml` |
| nas001b | フラグ作成 4:58、再起動 4:59 | フラグ作成 0:28、再起動 0:29(日付は変えない) | `rundeck_nas_cron_reboot_time.yml` |
| proxy001b | `chk_umnt.sh` 毎週日曜 4:50、`chk_mnt.sh` 毎週日曜 5:10 | `chk_umnt.sh` 毎週日曜 0:20、`chk_mnt.sh` 毎週日曜 0:40 | `rundeck_proxy_cron_nfs_time.yml` |
| ntp001 | バックアップ:root の crontab(毎月1日 2:00) | Rundeck のジョブ(毎月1日 2:00)。root の crontab の行はコメントにした | `rundeck_cron_backup_self_off.yml` |

- raspberrypi-desktop:Rundeck サーバーのため cron のまま運用する。日曜と毎月1日に再起動すると、同じ時刻の Rundeck のジョブ(1:57 起動のフラグ作成・再起動、毎月1日 2:00 のバックアップ)を実行できないため、月末日の 20:00 以降にまとめた(利用者の希望は月1回、月末)
- nas001b:再起動すると、NFS ブートのサーバー(ntp002、ntp003、ntp004、web001b、web002b)も再起動することがある(原因と対応は `06_rundeck.md` の9章)。NFS ブートのサーバーの予定の再起動(1:59〜3:59)より前に nas001b を再起動する順番にした
- proxy001b:2つの処理は nas001b の再起動に合わせたもの(利用者確認)。nas001b の再起動の9分前と11分後という間隔は変えていない
