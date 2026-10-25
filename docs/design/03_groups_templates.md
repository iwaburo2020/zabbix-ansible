# ホストグループ・テンプレート設計

- ホストグループの定義: roles/zabbix_groups/defaults/main.yml(zabbix_groups.yml で作成)
- 独自テンプレートの定義: roles/zabbix_templates/defaults/main.yml(zabbix_templates.yml で登録)

## 1. ホストグループ

役割別と OS 別の2軸で作成し、1台のホストを両方のグループに所属させる。Zabbix 標準のグループは変更しない。

| 区分 | グループ |
|---|---|
| 役割別 | Role/Zabbix、Role/DB、Role/NAS、Role/NTP、Role/GPS、Role/Web、Role/DNS、Role/Cache、Role/DHCP、Role/Proxy、Role/WAP、Role/Backup、Role/Log |
| OS 別 | OS/RaspberryPiOS、OS/Ubuntu、OS/Windows、OS/AlmaLinux |
| ネットワーク機器 | Network/Yamaha |

- テンプレートグループ: Templates/Custom(独自テンプレートの登録先)
- Zabbix server は標準グループ Zabbix servers にも所属させている

## 2. テンプレートの方針

- Zabbix 標準テンプレートは変更しない
- 標準テンプレート(Linux by Zabbix agent、Windows by Zabbix agent、ICMP Ping、Network Generic Device by SNMP、Zabbix server health)はホストに直接リンクする
- 独自テンプレート(Base-*、Time-*、Role-*、Net-*)には標準テンプレートをリンクしない
  - 理由: Zabbix 7.0 の configuration.importcompare は、ほかのテンプレートをリンクしたテンプレートを毎回「変更あり」と判定するため、community.zabbix の zabbix_template が毎回 changed を返す
- ホスト登録(zabbix_host)の link_templates には、そのホストにリンクするテンプレートをすべて列挙する(列挙しなかったテンプレートはリンク解除される)
- 閾値などの値はテンプレートマクロで共通化し、例外はホスト側マクロで上書きする
- 独自テンプレートの UUID は、Zabbix 7.0 のインポート形式(UUIDv4)に合わせ、テンプレート名・アイテムキーなどから毎回同じ値を生成する

## 3. 独自テンプレート一覧

監視方式の意味:
- ログ(Active): ログファイル・イベントログを Active チェックで収集(条件は 04_monitoring.md)
- ユニット(Passive): systemd ユニットの ActiveState が active 以外で障害
- UDP 待受(Passive): 指定ポートの待受が無ければ障害

| テンプレート | 監視方式 | 監視対象 | テンプレートマクロ |
|---|---|---|---|
| Base-Linux | ログ(Active) | /var/log/syslog、/var/log/messages | - |
| Base-Windows | ログ(Active) | イベントログ System、Application | - |
| Time-chrony | ユニット(Passive) | {$TIME.CHRONY.UNIT} | {$TIME.CHRONY.UNIT}=chrony.service |
| Time-timesyncd | ユニット(Passive) | systemd-timesyncd.service | - |
| Role-Zabbix | ユニット(Passive) | zabbix-server.service、apache2.service | - |
| Role-DB | ユニット(Passive) | postgresql@18-main.service | - |
| Role-NAS | ユニット(Passive) | smbd.service、nmbd.service、nfs-mountd.service、nfs-server.service | - |
| Role-NTP | UDP 待受(Passive) | 123 | - |
| Role-GPS | ユニット(Passive) | gpsd.service | - |
| Role-Web-Apache | ユニット(Passive) | apache2.service | - |
| Role-Web-Nginx | ユニット(Passive) | nginx.service | - |
| Role-DNS | ユニット(Passive) | named.service | - |
| Role-Cache | ユニット(Passive) | unbound.service | - |
| Role-DHCP | ユニット(Passive) | kea-dhcp4-server.service、kea-ctrl-agent.service、isc-stork-agent.service、isc-stork-server.service、postgresql@17-main.service | - |
| Role-Proxy-Linux | ユニット(Passive) | squid.service | - |
| Role-WAP | ユニット(Passive) | hostapd.service | - |
| Role-Log | ユニット(Passive) | rsyslog.service | - |
| Net-Yamaha-RT | SNMP | CPU 使用率(yrhCpuUtil1min、1.3.6.1.4.1.1182.2.1.6.0)、メモリ使用率(yrhMemoryUtil、1.3.6.1.4.1.1182.2.1.4.0) | - |

- Net-Yamaha-RT はヤマハ プライベート MIB(YAMAHA-RT-HARDWARE)を使用。対象は NVR510・NVR500
- logbackup001(AlmaLinux)はホスト側マクロ {$TIME.CHRONY.UNIT}=chronyd.service で上書きする

## 4. ICMP-backup001

- backup001 の死活監視用。Zabbix サーバーから 192.168.250.1 へ ICMP を送る
- アイテム名は標準の ICMP Ping と同じ「ICMP ping」
- トリガーは wap001b の ICMP 障害に依存させ、wap001b 停止時に backup001 の障害を重ねて出さない
- zabbix_hosts.yml の中で作成する(backup001 以外の登録 → ICMP-backup001 の作成 → backup001 の登録)

## 5. 時刻同期テンプレートの割り当て

| テンプレート | 対象ホスト |
|---|---|
| Time-chrony | Zabbix server、ntp001〜ntp004、gps001b、wap001b、logbackup001(chronyd.service) |
| Time-timesyncd | 上記以外の Linux ホスト |
