# Zabbix API の確認方法

playbook の実行結果や監視の状態を、Zabbix API で確認する。

- 実行端末: humidai002(ユーザー pi)
- API の URL: http://192.168.200.3/zabbix/api_jsonrpc.php
- API トークン: ~/.zabbix_api_token(権限 600)。値を画面やファイルに出力しない

## 1. 共通の書式

```
curl --noproxy '*' -s -H "Authorization: Bearer $(cat ~/.zabbix_api_token)" -H 'Content-Type: application/json-rpc' -d '<リクエスト>' http://192.168.200.3/zabbix/api_jsonrpc.php | python3 -m json.tool
```

- 1行で実行する
- `--noproxy '*'`: humidai002 に設定されているプロキシを使わずに接続する
- `-H "Authorization: Bearer $(cat ~/.zabbix_api_token)"`: トークンファイルの内容を認証ヘッダーに入れる
- `python3 -m json.tool`: 応答の JSON を見やすく整形する
- 応答に "error" がある場合は、その message と data を確認する

## 2. 確認の例

以下の各コマンドは、いずれも1行で実行する。

### 2.1 API のバージョン(認証不要)

```
curl --noproxy '*' -s -H 'Content-Type: application/json-rpc' -d '{"jsonrpc":"2.0","method":"apiinfo.version","params":{},"id":1}' http://192.168.200.3/zabbix/api_jsonrpc.php | python3 -m json.tool
```

- apiinfo.version は認証なしで呼び出す API のため、Authorization ヘッダーを付けない

### 2.2 ホストとインターフェースの状態

```
curl --noproxy '*' -s -H "Authorization: Bearer $(cat ~/.zabbix_api_token)" -H 'Content-Type: application/json-rpc' -d '{"jsonrpc":"2.0","method":"host.get","params":{"output":["host","status"],"selectInterfaces":["ip","port","type","available"],"sortfield":"host"},"id":1}' http://192.168.200.3/zabbix/api_jsonrpc.php | python3 -m json.tool
```

- status: 0 = 監視有効、1 = 監視無効
- type: 1 = エージェント、2 = SNMP
- available: 0 = 不明、1 = 利用可能、2 = 利用不可

### 2.3 障害が発生中のトリガー

```
curl --noproxy '*' -s -H "Authorization: Bearer $(cat ~/.zabbix_api_token)" -H 'Content-Type: application/json-rpc' -d '{"jsonrpc":"2.0","method":"trigger.get","params":{"output":["description","priority","lastchange"],"filter":{"value":1},"monitored":true,"expandDescription":true,"selectHosts":["host"]},"id":1}' http://192.168.200.3/zabbix/api_jsonrpc.php | python3 -m json.tool
```

- priority: 0 = 未分類、1 = 情報、2 = 警告、3 = 軽度の障害、4 = 重度の障害、5 = 致命的な障害

### 2.4 取得不可のアイテム

```
curl --noproxy '*' -s -H "Authorization: Bearer $(cat ~/.zabbix_api_token)" -H 'Content-Type: application/json-rpc' -d '{"jsonrpc":"2.0","method":"item.get","params":{"output":["name","key_","error"],"filter":{"state":1},"monitored":true,"selectHosts":["host"]},"id":1}' http://192.168.200.3/zabbix/api_jsonrpc.php | python3 -m json.tool
```

### 2.5 独自テンプレート

```
curl --noproxy '*' -s -H "Authorization: Bearer $(cat ~/.zabbix_api_token)" -H 'Content-Type: application/json-rpc' -d '{"jsonrpc":"2.0","method":"template.get","params":{"output":["host"],"search":{"host":["Base-","Time-","Role-","Net-","ICMP-"]},"searchByAny":true,"startSearch":true,"sortfield":"host"},"id":1}' http://192.168.200.3/zabbix/api_jsonrpc.php | python3 -m json.tool
```

### 2.6 メンテナンス期間

```
curl --noproxy '*' -s -H "Authorization: Bearer $(cat ~/.zabbix_api_token)" -H 'Content-Type: application/json-rpc' -d '{"jsonrpc":"2.0","method":"maintenance.get","params":{"output":["name","maintenance_type","active_since","active_till"],"selectHosts":["host"],"selectTimeperiods":"extend"},"id":1}' http://192.168.200.3/zabbix/api_jsonrpc.php | python3 -m json.tool
```

- active_since、active_till は UNIX 時刻(秒)

### 2.7 ダッシュボード(Global view)のウィジェット

```
curl --noproxy '*' -s -H "Authorization: Bearer $(cat ~/.zabbix_api_token)" -H 'Content-Type: application/json-rpc' -d '{"jsonrpc":"2.0","method":"dashboard.get","params":{"output":["name"],"selectPages":"extend","filter":{"name":"Global view"}},"id":1}' http://192.168.200.3/zabbix/api_jsonrpc.php | python3 -m json.tool
```

## 3. 参考

- Zabbix 7.0 API リファレンス: https://www.zabbix.com/documentation/7.0/en/manual/api/reference
