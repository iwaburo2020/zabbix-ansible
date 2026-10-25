# 定義変更の固定手順

Zabbix の設定・監視定義を変更するときは、必ず次の手順で行う。Zabbix のフロントエンドで直接変更しない(次回の playbook 実行で定義の内容に戻るため)。

- 実行端末: humidai002(ユーザー pi)
- 作業ディレクトリ: ~/zabbix-ansible(正本)

## 1. 手順

| 順 | 作業 | コマンド | 確認すること |
|---|---|---|---|
| 1 | 現在地と状態の確認 | `pwd` → `git status --short; git log --oneline -1` | /home/pi/zabbix-ansible にいること。未コミットが無く、最新コミットに (HEAD -> main, origin/main) と表示されること |
| 2 | 定義の修正 | 対象ファイル(roles/<ロール>/defaults/main.yml など)を編集 | - |
| 3 | 差分の確認 | `git diff --stat` → `git diff` | 意図したファイル・行だけが変わっていること |
| 4 | 構文の確認 | `ansible-playbook <playbook> --syntax-check` | エラーが出ないこと |
| 5 | commit | `git add <ファイル>` → `git commit -m "<ロール名>: <変更内容>"` | - |
| 6 | 実行 | `ansible-playbook <playbook>` | failed=0。変更した箇所が changed になること |
| 7 | 再実行 | `ansible-playbook <playbook>`(同じコマンド) | changed=0(冪等であること) |
| 8 | 未 push のコミットの確認 | `git log --oneline origin/main..main` | 手順5のコミットだけが表示されること |
| 9 | push | `git push` | - |
| 10 | 結果の確認 | Zabbix API で確認(03_api_check.md) | 定義どおりに反映されていること |

- 手順7で changed=0 にならない場合は push しない。原因を直して手順2からやり直す
- git add はファイルを指定して行う(`git add .` は使わない)

## 2. 実行時の入力

### 2.1 BECOME password

| 対象ホスト | 入力する内容 |
|---|---|
| raspberrypi-desktop(ユーザー ubuntu)を含む場合 | ubuntu の sudo パスワード |
| 上記以外(Raspberry Pi OS、AlmaLinux、Windows、Zabbix API を使う playbook) | 何も入力せず Enter |

- zabbix_agent.yml は、sudo パスワードが同じホストだけを `-l` でまとめて指定する

### 2.2 playbook が求めるパスワード

| playbook | プロンプト | 入力する内容 |
|---|---|---|
| zabbix_db.yml | Zabbix DBユーザーのパスワード(シングルクォート不可) | 設定する Zabbix DB ユーザーのパスワード(確認のため2回) |
| zabbix_server.yml | Zabbix DBユーザーのパスワード(zabbix_db.yml で設定したもの) | zabbix_db.yml で設定したパスワード |

## 3. 実行順(全体を構築し直す場合)

| 順 | playbook | 前提 |
|---|---|---|
| 1 | zabbix_db.yml | - |
| 2 | zabbix_server.yml | zabbix_db.yml 実行済み |
| 3 | sudoers_pi.yml | pi の sudo 設定がほかのホストと異なるホストがある場合のみ |
| 4 | zabbix_agent.yml、zabbix_agent_almalinux.yml、zabbix_agent_windows.yml | - |
| 5 | zabbix_nfs.yml、zabbix_nat.yml、zabbix_route.yml | - |
| 6 | zabbix_groups.yml | Zabbix サーバー構築済み |
| 7 | zabbix_templates.yml | zabbix_groups.yml 実行済み(テンプレートグループ Templates/Custom) |
| 8 | zabbix_hosts.yml | zabbix_groups.yml、zabbix_templates.yml 実行済み |
| 9 | zabbix_maintenance.yml | zabbix_hosts.yml 実行済み |
| 10 | zabbix_dashboard.yml | zabbix_hosts.yml 実行済み |

## 4. 定義を書くときの注意

- OS 標準の設定ファイルは変更・削除しない(Zabbix のダッシュボード等のアプリ側の設定はこの対象外)
- ホストごとに実行を分けず、ほかのサーバーの設定に合わせる
- ansible-core 2.19 では、loop するタスクの name に item を使うと警告が出るため使わない(表示は loop_control の label で行う)
- zabbix_host の link_templates には、そのホストにリンクするテンプレートをすべて列挙する(列挙しなかったテンプレートはリンク解除される)
- 独自テンプレートに標準テンプレートをリンクしない(理由は docs/design/03_groups_templates.md)
- API トークン・SNMP コミュニティー名・パスワードの値はリポジトリに書かない(保存場所は README.md)
