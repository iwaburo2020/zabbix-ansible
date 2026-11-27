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

- 定義: roles/zabbix_templates/defaults/main.yml(zabbix_log_regexp、zabbix_log_exclude、zabbix_eventlog_severity、zabbix_eventlog_exclude、zabbix_log_window)
- エージェントの Active チェックで収集する

| 対象 | テンプレート | 収集する内容 |
|---|---|---|
| Linux: /var/log/syslog、/var/log/messages | Base-Linux | 正規表現 `(?i)(error\|fail\|crit\|panic\|alert\|emerg)` に一致する行(大文字小文字を区別しない)。ただし 2.1 の外す行を除く |
| Windows: イベントログ System、Application | Base-Windows | レベルが Error、Critical のもの。ただし 2.1 の外す行を除く |

- 収集した時点で障害(Log message detected)とする
- 10分間(zabbix_log_window: 10m)新しい行が無ければ、障害は自動で解決する
- Zabbix server(raspberrypi-zabbix)もログ監視の対象に含める
- ログを読むため、zabbix ユーザーを adm グループに追加している(zabbix_agent.yml)
- NFS ブートのホストは、NFS サーバー(nas001b)側の設定で zabbix ユーザーがログを読めるようにしている(zabbix_nfs.yml)

### 2.1 ログ監視から外す行

- 調べた結果、対応が不要と判断した行は、Zabbix のグローバル正規表現(管理 > 一般設定 > 正規表現)で収集の対象から外す。OS・アプリ側の設定は変更しない
- 外す行の一覧は roles/zabbix_templates/defaults/main.yml の zabbix_log_exclude(Linux)、zabbix_eventlog_exclude(Windows)。1行ごとに何の行かをコメントで書く
- 一覧を変更したら ansible-playbook zabbix_templates.yml を実行する(項目のキーは変わらない)

| グローバル正規表現 | 式 | 参照する項目のキー |
|---|---|---|
| Log-Linux | zabbix_log_regexp(結果が TRUE)と、zabbix_log_exclude の各行(結果が FALSE) | `log[/var/log/syslog,@Log-Linux,,,skip]`、`log[/var/log/messages,@Log-Linux,,,skip]` |
| Eventlog-Windows | zabbix_eventlog_exclude の各行(結果が FALSE) | `eventlog[System,@Eventlog-Windows,"Error\|Critical",,,,skip]`、`eventlog[Application,@Eventlog-Windows,"Error\|Critical",,,,skip]` |

- すべての式を満たした行だけが収集される(Zabbix の仕様)。大文字小文字は区別しない
- 項目の uuid は、グローバル正規表現に替える前のキーから生成した値のままにしている(同じ項目として更新され、履歴が残る)

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

## 7. 現在の状態(2026-11-02 時点)

- 全22台を登録。SNMP 3台は取得可能、ICMP は全台取得可能

### 7.1 取得不可のアイテム(13件、対応しない)

| ホスト | 件数 | アイテム | 原因(エラーメッセージで確認) |
|---|---|---|---|
| Zabbix server | 10 | connector manager・connector worker・ipmi manager・ipmi poller・java poller・report manager・report writer・snmp trapper・vmware collector の稼働率、vmware buffer | 該当するプロセスが起動していない(No "…" processes started) |
| Zabbix server | 1 | zabbix[connector_queue] | connector が起動していない(StartConnectors 未設定) |
| backup001 | 1 | /sys/class/net/usb0/speed | カーネルが速度を返さない(invalid argument) |
| wap001b | 1 | /sys/class/net/wlan0/speed | カーネルが速度を返さない(invalid argument) |

### 7.2 Log message detected の対応

| 日付 | 内容 | 定義 |
|---|---|---|
| 2026-10-31 | トリガーの式を count(…,10m)>0 から nodata(…,10m)=0 に変更。2章のとおり、10分間新しい行が無ければ障害は自動で解決する(変更前は新しい行が来ない間は再計算されず、障害のまま残っていた) | roles/zabbix_templates/templates/custom_template.yml.j2 |
| 2026-11-01 | logrotate の失敗(17台):初期構築時の退避ファイル /etc/logrotate.d/rsyslog.org と /etc/logrotate.d/rsyslog の定義が重複していたため、rsyslog.org を /etc/logrotate.d.org/ へ移動 | logrotate_fix.yml |
| 2026-11-02 | logrotate の失敗(raspberrypi-desktop):/etc/logrotate.d/rsyslog・rsyslog-custom の su syslog adm(グループ adm では /var/log に書き込めない)の行を削除。rsyslog は OS 標準の内容に戻った | logrotate_error_fix.yml |
| 2026-11-02 | logrotate 実行時の squid の ERROR の行(proxy001b):/etc/squid/squid.conf の logformat combined(組み込みの書式の再定義で、squid は無視していた)の行を削除 | logrotate_error_fix.yml |
| 2026-11-02 | squid の DNS エラー(proxy001b):pwm-image.trendmicro.com・pwm-cdn.trendmicro.com は宛先側の DNS に A レコードが無く(CloudFront への CNAME のみ)、squid が DNS failure の ERROR を出していたため、/etc/squid/squid.conf の「http_access allow lan」の直前に、2 つのホスト名への要求を DNS で調べる前に拒否する「acl dns_nodata_dst dstdomain …」と「http_access deny dns_nodata_dst」を追加 | proxy001b の /etc/squid/squid.conf(手作業で変更。別チャット「squid DNS エラー対応」) |
| 2026-11-27 | 調べた結果、対応が不要と判断した行をログ監視から外した(2.1)。Linux は 29 件、Windows は 1 件の条件。2026-11-03〜11-24 に収集した 41,867 行のうち 5,928 行が対象 | roles/zabbix_templates/defaults/main.yml(zabbix_log_exclude、zabbix_eventlog_exclude) |
| 2026-11-27 | stork-server の Kea Leases puller のエラー(dhcp001b。約1分ごとに出続け、/var/log/syslog の障害が 10/18 から解決しなかった):Stork サーバーが全リースの取得(リーストラッキング)をエージェントに要求し、エージェントでその機能を有効にしていないため「This feature is not enabled.」で失敗していた。使っていない機能のため、Stork の画面「Configuration > Settings」の「Kea Leases Puller Interval」を 60 から 0 にして取得を止めた(戻すときは 60)。09:54:55 の行を最後に止まり、障害も解決 | dhcp001b の Stork の設定(画面で変更) |

- 未対応(原因を調べて対応する):proxy001b の squid の DNS エラー(pwm-image.trendmicro.jp、ipv6.msftconnecttest.com)、dns001b・dns002b の named の query failed、ntp003 の rpi-poe、proxy001b・wap001b の raspberrypi-poe-pwm、Zabbix server の snmpd、raspberrypi-desktop の rundeckd.service
