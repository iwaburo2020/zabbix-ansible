# 監視対象の追加手順

- 定義の変更・playbook の実行は、すべて 01_ansible_workflow.md の固定手順(差分確認 → --syntax-check → commit → 実行 → 再実行で changed=0 → push)で行う
- 実行端末: humidai002(ユーザー pi)、作業ディレクトリ: ~/zabbix-ansible

## 1. サーバー(エージェントで監視)

前提:
- 対象ホストの初期構築が完了していること
- humidai002 から SSH 鍵認証で接続できること
- VLAN200 のアドレスを持っていること

| 順 | 作業 | 対象ファイル・コマンド | 内容 |
|---|---|---|---|
| 1 | ホスト変数の作成 | host_vars/<ホスト名>.yml | 既存ファイルと同じ形式で作成する(1行目 `---`、2行目にコメントで用途・機器・OS、3行目 `ansible_host: <VLAN200 の IP>`) |
| 2 | inventory への追加 | inventory.yml | OS に応じたグループの hosts に追加する。Raspberry Pi OS: zabbix_agent_raspios、AlmaLinux: zabbix_agent_almalinux、Windows: zabbix_agent_windows、Ubuntu: zabbix_agent(raspberrypi-desktop と同じ位置) |
| 3 | sudo 設定の統一(Raspberry Pi OS で必要な場合のみ) | `ansible-playbook sudoers_pi.yml -l <ホスト名>` | pi の sudo 設定をほかのホストに合わせる |
| 4 | エージェント導入 | `ansible-playbook <導入用 playbook> -l <ホスト名>` | Raspberry Pi OS・Ubuntu: zabbix_agent.yml、AlmaLinux: zabbix_agent_almalinux.yml、Windows: zabbix_agent_windows.yml |
| 5 | 役割の追加(新しい役割の場合のみ) | roles/zabbix_groups/defaults/main.yml、roles/zabbix_templates/defaults/main.yml | ホストグループ Role/<役割> と独自テンプレート Role-<役割> を追加し、zabbix_groups.yml → zabbix_templates.yml の順に実行する |
| 6 | ホスト定義の追加 | roles/zabbix_hosts/defaults/main.yml の zabbix_host_defs | キーは inventory のホスト名。os、groups(役割別)、templates(時刻同期・役割別)を指定し、必要な場合のみ ip、port、icmp_template、macros を指定する |
| 7 | Zabbix への登録 | `ansible-playbook zabbix_hosts.yml` | グループ所属・テンプレートのリンク・インターフェースを登録する |
| 8 | 登録結果の確認 | 03_api_check.md | エージェントのインターフェースが利用可能(available=1)になっていること |
| 9 | 設計書の更新 | docs/design/02_hosts.md(役割を追加した場合は 03_groups_templates.md も) | 一覧に追加したホストを記載する |

- 手順6の os は zabbix_os_defs のキー(RaspberryPiOS、Ubuntu、AlmaLinux、Windows)。OS 別グループ・OS 別テンプレート・ICMP Ping は os から自動で付与される
- 時刻同期のテンプレートは、chrony を使うホストは Time-chrony、それ以外は Time-timesyncd
- NFS ブートのホストを追加する場合は、zabbix ユーザーがログを読めるかを確認し、必要に応じて zabbix_nfs ロールの定義を見直して zabbix_nfs.yml を実行する

## 2. ネットワーク機器(SNMP で監視)

| 順 | 作業 | 対象 | 内容 |
|---|---|---|---|
| 1 | 機器の SNMP 設定 | 対象機器 | SNMPv2c、読み出し専用コミュニティー(値は humidai002 の ~/.zabbix_snmp_community)、アクセス許可は 192.168.200.3 のみ、トラップなし。手順は各機器のマニュアルを確認してから行う |
| 2 | VLAN 間フィルタの追加(VLAN1 の機器の場合のみ) | NVR510 | 事前に config をバックアップしてから、lan1/1 secure filter に Zabbix サーバー(192.168.200.3)との SNMP(UDP 161)・ICMP の許可を追加する。NVR510 の CLI は一般ユーザーでログインし、administrator で管理ユーザーに切り替えてから設定する |
| 3 | ホスト定義の追加 | roles/zabbix_hosts/defaults/main.yml の zabbix_snmp_host_defs | キーは Zabbix 登録名。ip(SNMP のインターフェース、ポート 161)と templates(標準テンプレート以外にリンクするもの)を指定する |
| 4 | Zabbix への登録 | `ansible-playbook zabbix_hosts.yml` | グループ Network/Yamaha、標準テンプレート Network Generic Device by SNMP で登録される |
| 5 | 登録結果の確認 | 03_api_check.md | SNMP のインターフェースが利用可能(available=1)になっていること |
| 6 | 設計書の更新 | docs/design/01_overview.md、docs/design/02_hosts.md | 監視対象と VLAN 間フィルタを記載する |

## 3. ダッシュボード

- Global view の「サーバー・NW機器の状態」は全ホストを表示するため、登録したホストは自動で表示される
- 表示行数は 25(roles/zabbix_dashboard/defaults/main.yml の show_lines)。登録ホストが 25台を超える場合は show_lines を変更して zabbix_dashboard.yml を実行する
