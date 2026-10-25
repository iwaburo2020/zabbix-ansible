# 全体構成

## 1. 構成要素

| 役割 | ホスト名 | IP | 機器・OS | ソフトウェア |
|---|---|---|---|---|
| Zabbix サーバー・フロントエンド | raspberrypi-zabbix | 192.168.200.3 | Raspberry Pi 4B 8GB、Raspberry Pi OS Trixie arm64(USB SSD 起動) | Zabbix 7.0.32、Apache、PHP 8.4 |
| Zabbix DB | raspberrypi-desktop | 192.168.200.4 | ThinkPad X240、Ubuntu 26.04 amd64 | PostgreSQL 18 |
| Ansible コントロールノード | humidai002 | 192.168.200.26 | Raspberry Pi 500、Raspberry Pi OS Trixie | ansible 12.0.0(ansible-core 2.19.4)、同梱の community.zabbix 4.1.0・ansible.netcommon 8.1.0 |

- フロントエンド: http://192.168.200.3/zabbix
- バージョン方針: Zabbix 7.0 LTS で構築し、8.0 LTS のリリース後に移行する

## 2. ネットワーク

| VLAN | セグメント | 用途 |
|---|---|---|
| VLAN1 | 192.168.100.0/24 | メインネットワーク(外部接続可) |
| VLAN200 | 192.168.200.0/24 | 保守用ネットワーク(RDP、SSH、NFS、NTP など) |

- VLAN は NVR510 で管理(タグVLAN)
- Zabbix サーバー・DB は VLAN200 のみに接続
- サーバー(エージェント導入ホスト)は VLAN200 のアドレスで監視する

### 2.1 NVR510 の VLAN 間フィルタ(Zabbix 用に追加したもの)

- 追加したフィルタ: 500013、500014(lan1/1 out)、500023、500024(lan1/1 in)
- 内容: Zabbix サーバー(192.168.200.3)と NVR500(192.168.100.5)・WLX212(192.168.100.9)の間の SNMP(UDP 161)と ICMP を許可
- 適用後の設定:
  - `ip lan1/1 secure filter in 500020 500023 500024 500021 500022`
  - `ip lan1/1 secure filter out 500010 500013 500014 500011 500012`
- 追加前の config バックアップ: config1(2026/10/24 23:10:25)

### 2.2 backup001 への接続

- backup001 の実 IP は 192.168.250.1(wap001b にガジェットモードで接続)
- エージェント(Passive): Zabbix サーバーから 192.168.200.22:10060 に接続し、wap001b の nftables で 192.168.250.1:10050 へ転送する(zabbix_nat.yml)
- 死活監視(ICMP): Zabbix サーバーから 192.168.250.1 へ送る。raspberrypi-zabbix の NM プロファイル eth0 に 192.168.250.0/30 via 192.168.200.22 の経路を追加(zabbix_route.yml)

### 2.3 IPv6

- 各ホストは IPv6 を OS で無効化しているため、zabbix_server ロールで `Fping6Location=`(空)を設定している

## 3. エージェント方針

| 項目 | 方針 |
|---|---|
| 種別 | Zabbix agent 2 |
| チェック方式 | Passive に統一。ログ監視を行うため、全ホストで Active を併用(ServerActive=192.168.200.3) |
| 通信の暗号化 | なし(内部ネットワークのため PSK は使わない) |
| Hostname | Ansible inventory のホスト名(Zabbix の登録名と同じ) |
| Zabbix サーバー自身 | パッケージ標準設定のまま(Server=127.0.0.1、ServerActive=127.0.0.1、Hostname=Zabbix server)。Zabbix には既存ホスト「Zabbix server」(127.0.0.1:10050)として登録 |
| Windows(proxyvm001b) | OpenSSH Server 経由で Ansible から導入 |
| 対象外 | 端末機(humidai001b、humidai002) |

## 4. ネットワーク機器(SNMP)

| 項目 | 方針 |
|---|---|
| 監視対象 | NVR510(192.168.200.1)、NVR500(192.168.100.5)、WLX212(192.168.100.9) |
| 監視対象外 | SWX2210-8G、SWX2210P-10G |
| バージョン | SNMPv2c |
| コミュニティー | 読み出し専用のみ(3機器同じ値)。Zabbix ではホストマクロ {$SNMP_COMMUNITY} で設定し、値はリポジトリに含めない |
| アクセス許可 | 192.168.200.3 のみ |
| トラップ | 使わない |

- WLX212 はクラスター管理機能を使用しており、SNMP の設定は仮想コントローラー(192.168.100.8)で行う

## 5. Zabbix API

- 接続設定: vars/zabbix_api.yml(httpapi、`ansible_httpapi_use_proxy: false`)
- API トークン: ユーザー Admin、名前 zabbix-ansible、有効期限なし。値は humidai002 の `~/.zabbix_api_token`(権限 600)に保存し、リポジトリに含めない
