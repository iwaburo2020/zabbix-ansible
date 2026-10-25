# 監視内容

## 1. 監視項目の実施状況

| 監視項目 | 方法 |
|---|---|
| 死活(ICMP) | ICMP Ping(backup001 は ICMP-backup001、ネットワーク機器は Network Generic Device by SNMP) |
| リソース(CPU、メモリ、ディスク等) | Linux by Zabbix agent / Windows by Zabbix agent、NVR510・NVR500 は Net-Yamaha-RT |
| プロセス・サービス | Role-*、Time-*。proxyvm001b は Windows by Zabbix agent のサービス自動検出 |
| ログ・メッセージ | Base-Linux、Base-Windows(本書 2章) |
| メンテナンス期間 | gps001b-reboot(本書 3章) |
| ネットワーク機器(SNMP) | NVR510、NVR500、WLX212 |
| 通知 | 通知手段は使わない。ダッシュボードで確認する(本書 4章・5章) |

## 2. ログ監視

- 定義: roles/zabbix_templates/defaults/main.yml(zabbix_log_regexp、zabbix_eventlog_severity、zabbix_log_window)
- エージェントの Active チェックで収集する

| 対象 | テンプレート | 収集する内容 |
|---|---|---|
| Linux: /var/log/syslog、/var/log/messages | Base-Linux | 正規表現 `(?i)(error|fail|crit|panic|alert|emerg)` に一致する行(大文字小文字を区別しない) |
| Windows: イベントログ System、Application | Base-Windows | レベルが Error、Critical のもの |

- 収集した時点で障害(Log message detected)とする
- 10分間(zabbix_log_window: 10m)新しい行が無ければ、障害は自動で解決する
- Zabbix server(raspberrypi-zabbix)もログ監視の対象に含める
- ログを読むため、zabbix ユーザーを adm グループに追加している(zabbix_agent.yml)
- NFS ブートのホストは、NFS サーバー(nas001b)側の設定で zabbix ユーザーがログを読めるようにしている(zabbix_nfs.yml)

## 3. メンテナンス期間

- 定義: zabbix_maintenance.yml(ロール zabbix_maintenance、community.zabbix.zabbix_maintenance)

### 3.1 gps001b-reboot

gps001b の定期運用(OS 再起動と gpsd 停止)の時間帯に障害を抑制する。

| 項目 | 設定 |
|---|---|
| 対象 | gps001b |
| データ収集 | あり |
| 開始時刻・期間 | 23:15 から 2時間 |
| 有効期間 | 2026-10-01 00:00 〜 2037-12-31 00:00 |
| 期間 | 毎月第2週土曜、毎月第4週土曜、31日(1・3・5・7・8・10・12月)、30日(4・6・9・11月)、28日(2月)、29日(2月) |

- gps001b の定期運用: 第2土曜・第4土曜・毎月の末日(曜日問わず)の 23:20 に OS 再起動 → 翌 0:00 に OS 再起動 → 翌 1:00 に gpsd 起動(その間 gpsd 停止)
- Zabbix には「末日」の指定が無いため、月ごとの日数で指定している。うるう年は2月28日も対象になる

## 4. 通知

- メール等の通知手段は使わない
- 障害はダッシュボード(Global view)で確認する

## 5. ダッシュボード

- 定義: zabbix_dashboard.yml(ロール zabbix_dashboard)
- community.zabbix 4.1.0 にダッシュボード用のモジュールが無いため、humidai002(localhost)から ansible.builtin.uri で Zabbix API(dashboard.get / dashboard.update)を直接呼ぶ
- 対象ウィジェット以外(位置・設定)は変更しない。内容が同じ場合は更新しない(changed=0)

### 5.1 Global view の変更内容

| 項目 | 内容 |
|---|---|
| 対象ウィジェット | widgetid 1(初期データの Top hosts)。位置・大きさは初期データのまま |
| 名前 | サーバー・NW機器の状態 |
| 表示するホスト | 全22台(メンテナンス中のホストも表示)、ホスト名順 |
| 列 | ホスト名、死活(ICMP)、CPU 使用率、メモリ使用率、ディスク使用率(/)、ディスク使用率(C:) |
| その他のウィジェット(8個) | 変更なし |

- Global view の「ホスト稼働状況」は合計19台と表示される(初期設定ではエージェントのインターフェースだけを数えるため、ネットワーク機器3台は含まれない)

## 6. 障害として扱わないもの

| 対象 | 内容 | 対応 |
|---|---|---|
| proxyvm001b の自動起動サービス4件 | GoogleUpdater 2件、LMS、BtLockerService(停止が正常) | ホストマクロ {$SERVICE.NAME.NOT_MATCHES} で自動検出から除外 |
| gps001b の定期運用の時間帯 | OS 再起動・gpsd 停止 | メンテナンス期間 gps001b-reboot で抑制 |

## 7. 現在の状態(2026-10-25 時点)

- 全22台を登録。SNMP 3台は取得可能、ICMP は全台取得可能
- 取得不可のアイテム 13件: backup001 の usb0 と wap001b の wlan0 の speed、Zabbix server の内部プロセス 11件(内容は未調査)
- Log message detected の障害が複数のホストで発生している(2章の条件どおりの動作。内容の解析は未実施)
