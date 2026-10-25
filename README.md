# zabbix-ansible

Zabbix 7.0 LTS の監視基盤(サーバー・DB・エージェント・監視設定)を Ansible で構築・管理するリポジトリ。

- コントロールノード: humidai002(`~/zabbix-ansible` を正本として実行)
- Zabbix サーバー: raspberrypi-zabbix(192.168.200.3)、フロントエンド http://192.168.200.3/zabbix
- DB サーバー: raspberrypi-desktop(192.168.200.4、PostgreSQL 18)
- バージョン: Zabbix 7.0.32(8.0 LTS リリース後に移行予定)
- 初期構築用リポジトリ(RaspberryPi_script の raspi-setup)とは分けて管理する

## ディレクトリ構成

| パス | 内容 |
|---|---|
| ansible.cfg | Ansible の設定 |
| inventory.yml | 対象ホストとグループの一覧 |
| group_vars/ | グループ単位の変数 |
| host_vars/ | ホスト単位の変数 |
| vars/zabbix_api.yml | Zabbix API の接続設定(httpapi) |
| roles/ | 各 playbook が使うロール |
| *.yml(ルート直下) | playbook(下表) |
| docs/design/ | 監視設計書 |
| docs/runbook/ | 運用手順書 |

## playbook 一覧

| playbook | 内容 | 対象 | 前提・実行時の入力 |
|---|---|---|---|
| zabbix_db.yml | Zabbix 用データベース構築 | zabbix_db(raspberrypi-desktop) | Zabbix DB ユーザーのパスワードを設定(確認のため2回入力) |
| zabbix_server.yml | Zabbix サーバー構築 | zabbix_server(raspberrypi-zabbix) | zabbix_db.yml 実行済み。Zabbix DB ユーザーのパスワードを入力 |
| zabbix_agent.yml | エージェント2 導入(Raspberry Pi OS / Ubuntu)。Zabbix サーバーは zabbix ユーザーの adm グループ追加のみ | zabbix_agent、zabbix_server | `-l <ホスト名>` で指定。sudo パスワードが同じホストだけをまとめて指定 |
| zabbix_agent_almalinux.yml | エージェント2 導入(AlmaLinux) | zabbix_agent_almalinux | `-l <ホスト名>` で指定 |
| zabbix_agent_windows.yml | エージェント2 導入(Windows) | zabbix_agent_windows | `-l <ホスト名>` で指定 |
| sudoers_pi.yml | pi の sudo 設定をほかのホストに合わせる(Raspberry Pi OS) | zabbix_agent_raspios、zabbix_server | `-l <ホスト名>` で指定 |
| zabbix_nfs.yml | NFS ブートホストの zabbix ユーザーがログを読めるようにする | nas001b | - |
| zabbix_nat.yml | backup001 の Agent(Passive)へ接続するための転送(nftables) | wap001b | - |
| zabbix_route.yml | backup001 の死活監視(ICMP 192.168.250.1)のため、wap001b 経由の経路を追加 | zabbix_server | - |
| zabbix_groups.yml | ホストグループ・テンプレートグループの作成 | Zabbix API | - |
| zabbix_templates.yml | 独自テンプレート(Base-・Time-・Role-・Net-)の登録 | Zabbix API | - |
| zabbix_hosts.yml | ホスト登録(グループ所属・テンプレートのリンク・インターフェース) | Zabbix API | zabbix_groups.yml、zabbix_templates.yml 実行済み |
| zabbix_maintenance.yml | メンテナンス期間の登録 | Zabbix API | zabbix_hosts.yml 実行済み |
| zabbix_dashboard.yml | 標準ダッシュボード「Global view」のウィジェット編集 | localhost(humidai002 から Zabbix API) | zabbix_hosts.yml 実行済み |

## 実行方法

- humidai002 の `~/zabbix-ansible` で `ansible-playbook <playbook名>` を実行する
- BECOME password: raspberrypi-desktop(ubuntu)を対象に含む場合は ubuntu の sudo パスワード、それ以外は Enter のみ
- 定義を変更するときの手順は docs/runbook/01_ansible_workflow.md を参照

## リポジトリに含めない情報

| 情報 | 保存場所 |
|---|---|
| Zabbix API トークン | humidai002 の `~/.zabbix_api_token`(権限 600) |
| SNMP コミュニティー名 | humidai002 の `~/.zabbix_snmp_community`(権限 600) |
| Zabbix DB ユーザーのパスワード | リポジトリには保存しない(playbook 実行時に入力) |

## ドキュメント

| ファイル | 内容 |
|---|---|
| docs/design/01_overview.md | 全体構成 |
| docs/design/02_hosts.md | 監視対象一覧 |
| docs/design/03_groups_templates.md | ホストグループ・テンプレート設計 |
| docs/design/04_monitoring.md | 監視内容(ログ監視・メンテナンス期間・通知・ダッシュボード) |
| docs/runbook/01_ansible_workflow.md | 定義変更の固定手順 |
| docs/runbook/02_add_host.md | 監視対象の追加手順 |
| docs/runbook/03_api_check.md | Zabbix API の確認方法 |
