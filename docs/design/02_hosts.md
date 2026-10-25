# 監視対象一覧

- 定義の正本: roles/zabbix_hosts/defaults/main.yml(zabbix_host_defs、zabbix_snmp_host_defs)
- 登録: zabbix_hosts.yml(Zabbix API 経由)
- 監視対象: 22台(サーバー 19台、ネットワーク機器 3台)

## 1. サーバー共通

### 1.1 OS 別の設定

| OS | OS 別グループ | ホストにリンクするテンプレート |
|---|---|---|
| Raspberry Pi OS | OS/RaspberryPiOS | Linux by Zabbix agent、Base-Linux |
| Ubuntu | OS/Ubuntu | Linux by Zabbix agent、Base-Linux |
| AlmaLinux | OS/AlmaLinux | Linux by Zabbix agent、Base-Linux |
| Windows | OS/Windows | Windows by Zabbix agent、Base-Windows |

### 1.2 死活監視(ICMP)

- Zabbix サーバーから ICMP を送る。テンプレートは標準の ICMP Ping
- backup001 のみ ICMP-backup001(192.168.250.1 への ICMP。wap001b の ICMP 障害に依存)

## 2. サーバー(19台)

インターフェースはエージェント(ポート 10050)。テンプレート列は、1.1 と 1.2 のテンプレートに加えてリンクするもの。

| Zabbix 登録名 | インターフェース | 用途 | OS | 役割別グループ | テンプレート(時刻同期・役割別) | ホストマクロ |
|---|---|---|---|---|---|---|
| Zabbix server(raspberrypi-zabbix) | 127.0.0.1:10050 | Zabbix サーバー | Raspberry Pi OS | Zabbix servers、Role/Zabbix | Zabbix server health、Time-chrony、Role-Zabbix | - |
| raspberrypi-desktop | 192.168.200.4:10050 | Zabbix DB | Ubuntu | Role/DB | Time-timesyncd、Role-DB | - |
| nas001b | 192.168.200.7:10050 | NAS/NFS サーバー | Raspberry Pi OS | Role/NAS | Time-timesyncd、Role-NAS | - |
| ntp001 | 192.168.200.9:10050 | NTP サーバー#1 | Raspberry Pi OS | Role/NTP | Time-chrony、Role-NTP | - |
| logbackup001 | 192.168.200.10:10050 | システムログ収集サーバー | AlmaLinux | Role/Log | Time-chrony、Role-Log | {$TIME.CHRONY.UNIT}=chronyd.service |
| ntp002 | 192.168.200.12:10050 | NTP サーバー#2 | Raspberry Pi OS | Role/NTP | Time-chrony、Role-NTP | - |
| ntp003 | 192.168.200.13:10050 | NTP サーバー#3 | Raspberry Pi OS | Role/NTP | Time-chrony、Role-NTP | - |
| ntp004 | 192.168.200.14:10050 | NTP サーバー#4 | Raspberry Pi OS | Role/NTP | Time-chrony、Role-NTP | - |
| proxyvm001b | 192.168.200.15:10050 | プロキシサーバー(Windows10 + VMware Player) | Windows | Role/Proxy | なし(サービス監視は Windows by Zabbix agent の自動検出) | {$SERVICE.NAME.NOT_MATCHES}(標準の除外一覧に GoogleUpdater.+、LMS、BtLockerService を追加) |
| cache001b | 192.168.200.16:10050 | DNS キャッシュサーバー | Raspberry Pi OS | Role/Cache | Time-timesyncd、Role-Cache | - |
| proxy001b | 192.168.200.17:10050 | プロキシサーバー | Raspberry Pi OS | Role/Proxy | Time-timesyncd、Role-Proxy-Linux | - |
| dns001b | 192.168.200.18:10050 | DNS サーバー#1 | Raspberry Pi OS | Role/DNS | Time-timesyncd、Role-DNS | - |
| dns002b | 192.168.200.19:10050 | DNS サーバー#2 | Raspberry Pi OS | Role/DNS | Time-timesyncd、Role-DNS | - |
| gps001b | 192.168.200.20:10050 | GPS モジュール接続サーバー | Raspberry Pi OS | Role/GPS | Time-chrony、Role-GPS | - |
| dhcp001b | 192.168.200.21:10050 | DHCP サーバー | Raspberry Pi OS | Role/DHCP | Time-timesyncd、Role-DHCP | - |
| wap001b | 192.168.200.22:10050 | 無線 LAN アクセスポイント | Raspberry Pi OS | Role/WAP | Time-chrony、Role-WAP | - |
| backup001 | 192.168.200.22:10060 | バックアップサーバー | Raspberry Pi OS | Role/Backup | Time-timesyncd | - |
| web001b | 192.168.200.24:10050 | Web サーバー#1(Apache) | Raspberry Pi OS | Role/Web | Time-timesyncd、Role-Web-Apache | - |
| web002b | 192.168.200.25:10050 | Web サーバー#2(nginx) | Raspberry Pi OS | Role/Web | Time-timesyncd、Role-Web-Nginx | - |

- Zabbix server: 既存ホスト「Zabbix server」を使用し、標準グループ Zabbix servers と標準テンプレート Zabbix server health を残している
- backup001: wap001b の nftables で 192.168.250.1:10050 へ転送(zabbix_nat.yml)
- 登録順: backup001 以外 → ICMP-backup001 テンプレート → backup001

## 3. ネットワーク機器(3台)

| Zabbix 登録名 | インターフェース | 用途 | グループ | テンプレート |
|---|---|---|---|---|
| NVR510 | SNMP 192.168.200.1:161 | ルーター | Network/Yamaha | Network Generic Device by SNMP、Net-Yamaha-RT |
| NVR500 | SNMP 192.168.100.5:161 | ルーター | Network/Yamaha | Network Generic Device by SNMP、Net-Yamaha-RT |
| WLX212 | SNMP 192.168.100.9:161 | 無線 LAN アクセスポイント | Network/Yamaha | Network Generic Device by SNMP |

- SNMPv2c。コミュニティーはホストマクロ {$SNMP_COMMUNITY} で設定し、値は humidai002 の ~/.zabbix_snmp_community(権限 600)から読み込む
- 死活監視(ICMP)は Network Generic Device by SNMP に含まれるため、ICMP Ping はリンクしない

## 4. 監視対象外

| ホスト | IP | 理由 |
|---|---|---|
| humidai001b | 192.168.200.23 | 端末機 |
| humidai002 | 192.168.200.26 | 端末機(Ansible コントロールノード) |
| SWX2210-8G | 192.168.100.2 | SNMP 監視の対象から除外 |
| SWX2210P-10G | 192.168.100.10 | SNMP 監視の対象から除外 |
