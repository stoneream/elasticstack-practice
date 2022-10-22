# 1-with-fluentd

## やること

- ElasticSerchを立てる
- Fluentdでsyslogとauth.logをElasticSearchに投入する

## サーバーを用意する

今回もvagrantを使う。

```
vagrant up

ssh -p 2223 vagrant@127.0.0.1
```

## 必要なソフトウェアのインストール

以降、vagrant上での作業

```bash
# とりあえずいろいろアプデ
sudo apt-get update -y && sudo apt-get upgrade -y

# fluentdを入れる
curl -fsSL https://toolbelt.treasuredata.com/sh/install-ubuntu-jammy-td-agent4.sh | sh

# fluent-dのエージェントが立ち上がっているが確認する
sudo systemctl status td-agent.service

# ● td-agent.service - td-agent: Fluentd based data collector for Treasure Data
#      Loaded: loaded (/lib/systemd/system/td-agent.service; enabled; vendor preset: enabled)
#      Active: active (running) since Sat 2022-10-22 15:32:53 UTC; 1min 49s ago
#        Docs: https://docs.treasuredata.com/display/public/PD/About+Treasure+Data%27s+Server-Side+Agent
#     Process: 697 ExecStart=/opt/td-agent/bin/fluentd --log $TD_AGENT_LOG_FILE --daemon /var/run/td-agent/td-agent.pid $TD_AGENT_OPTIONS (code=exited, status=0/SUCCESS)
#    Main PID: 810 (fluentd)
#       Tasks: 9 (limit: 4575)
#      Memory: 120.3M
#         CPU: 1.175s
#      CGroup: /system.slice/td-agent.service
#              ├─810 /opt/td-agent/bin/ruby /opt/td-agent/bin/fluentd --log /var/log/td-agent/td-agent.log --daemon /var/run/td-agent/td-agent.pid
#              └─813 /opt/td-agent/bin/ruby -Eascii-8bit:ascii-8bit /opt/td-agent/bin/fluentd --log /var/log/td-agent/td-agent.log --daemon /var/run/td-agent/td-agent.pid --under-supervi>
# Oct 22 15:32:51 ubuntu2204.localdomain systemd[1]: Starting td-agent: Fluentd based data collector for Treasure Data...
# Oct 22 15:32:53 ubuntu2204.localdomain systemd[1]: Started td-agent: Fluentd based data collector for Treasure Data.

# ElasticSearchを取ってくる
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.4.3-linux-x86_64.tar.gz

# 解答
tar -xzfv elasticsearch-8.4.3-linux-x86_64.tar.gz

tmux
```

- Install by DEB Package (Debian/Ubuntu) - Fluentd : https://docs.fluentd.org/installation/install-by-deb

## ElasticSearchを立ち上げる

```bash
cd elasticsearch-8.4.3

# 立ち上げる
bin/elasticsearch

# パスワードとか確認してメモっておく

# ✅ Elasticsearch security features have been automatically configured!
# ✅ Authentication is enabled and cluster connections are encrypted.
# 
# ℹ  Password for the elastic user (reset with `bin/elasticsearch-reset-password -u elastic`):
#   gHR8_w+kZsf3hzgV8kxG
# 
# ℹ  HTTP CA certificate SHA-256 fingerprint:
#   15d3f0259894359da1a9c25ce5f438ff2c0df18db39007e7a7e1dc6ea132669b
# 
# ℹ  Configure Kibana to use this cluster:
# • Run Kibana and click the configuration link in the terminal when Kibana starts.
# • Copy the following enrollment token and paste it into Kibana in your browser (valid for the next 30 minutes):
#   eyJ2ZXIiOiI4LjQuMyIsImFkciI6WyIxMC4wLjIuMTU6OTIwMCJdLCJmZ3IiOiIxNWQzZjAyNTk4OTQzNTlkYTFhOWMyNWNlNWY0MzhmZjJjMGRmMThkYjM5MDA3ZTdhN2UxZGM2ZWExMzI2NjliIiwia2V5IjoiTFBaYkFJUUJJY190WjZoU2F4SFM6Vld2aDI2bVVTcHVnNWNDU2Zad0loUSJ9
# 
# ℹ  Configure other nodes to join this cluster:
# • On this node:
#   ⁃ Create an enrollment token with `bin/elasticsearch-create-enrollment-token -s node`.
#   ⁃ Uncomment the transport.host setting at the end of config/elasticsearch.yml.
#   ⁃ Restart Elasticsearch.
# • On other nodes:
#   ⁃ Start Elasticsearch with `bin/elasticsearch --enrollment-token <token>`, using the enrollment token that you generated.
```

tmuxで別のウィンドウを開く

```bash
cd elasticsearch-8.4.3

# 疎通確認
curl --cacert config/certs/http_ca.crt -u elastic https://localhost:9200

# Enter host password for user 'elastic': (パスワードを入力)
# {
#   "name" : "ubuntu2204.localdomain",
#   "cluster_name" : "elasticsearch",
#   "cluster_uuid" : "nYqvmRFSSFSm2nNNZIukhQ",
#   "version" : {
#     "number" : "8.4.3",
#     "build_flavor" : "default",
#     "build_type" : "tar",
#     "build_hash" : "42f05b9372a9a4a470db3b52817899b99a76ee73",
#     "build_date" : "2022-10-04T07:17:24.662462378Z",
#     "build_snapshot" : false,
#     "lucene_version" : "9.3.0",
#     "minimum_wire_compatibility_version" : "7.17.0",
#     "minimum_index_compatibility_version" : "7.0.0"
#   },
#   "tagline" : "You Know, for Search"
# }
```

## fluentdの設定

```bash
# 設定ファイルの編集
sudo vim /etc/td-agent/td-agent.conf
```

```conf
<source>
  @type syslog
  port 5140
  bind 0.0.0.0
  tag system
</source>

<match system>
  @type elasticsearch

  scheme https
  host localhost
  port 9200
  ssl_verify false

  logstash_format true

  user elastic
  password gHR8_w+kZsf3hzgV8kxG

  # 接続エラーでリトライ中のときにkillしても死ななくなるのでfalseにする
  # デバッグ中は色々試したいのでこのようにした
  # reconnect_on_error false
  # reload_on_failure false
  # reload_connections false
</match>
```

```bash
# 変更を適用
sudo systemctl restart td-agent.service
```

- elasticsearch - Fluentd : https://docs.fluentd.org/output/elasticsearch
- Config: Transport Section - Fluentd : https://docs.fluentd.org/configuration/transport-section

## ElasticSearchにデータが入っているか確認する

```bash
curl --cacert ~/elasticsearch-8.4.3/config/certs/http_ca.crt -u elastic:gHR8_w+kZsf3hzgV8kxG https://localhost:9200/_aliases?pretty

# 何らかのログが入っているっぽい
# {
#   ".security-7" : {
#     "aliases" : {
#       ".security" : {
#         "is_hidden" : true
#       }
#     }
#   },
#   "logstash-2022.10.22" : {
#     "aliases" : { }
#   }
# }

logout
```

## サーバーを破棄する

動作確認できたのでとりあえず消す

```
vagrant destroy
```
