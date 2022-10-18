# elasticstack-practice

ElasticSearchに何かしらのログを集めてみるテストをしたときのメモ

## サーバーを用意する

```bash
vagrant up --provider=virtualbox

vagrant ssh

# とりあえずいろいろアプデ
sudo apt update && sudo apt upgrade
```

## Elasticsearchの導入と設定

基本的に公式ドキュメントに従って作業する。  
意識が低いのでチェックサムの確認とかを飛ばしている。  

Install Elasticsearch from archive on Linux or MacOS | Elasticsearch Guide [8.4] | Elastic : https://www.elastic.co/guide/en/elasticsearch/reference/8.4/targz.html


```bash
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.4.3-linux-x86_64.tar.gz

tar -xzf elasticsearch-8.4.3-linux-x86_64.tar.gz

cd elasticsearch-8.4.3/ 

# 他にも色々作業するのでtmuxを立ち上げておくと良い
# とりあえず起動

./bin/elasticsearch
```

初回起動で色々認証情報とかが出てくる。メモっておくが吉。  
（うっかり見逃しても後から再設定できる。）  

```
✅ Elasticsearch security features have been automatically configured!
✅ Authentication is enabled and cluster connections are encrypted.

ℹ  Password for the elastic user (reset with `bin/elasticsearch-reset-password -u elastic`):
  hogehogehoge

ℹ  HTTP CA certificate SHA-256 fingerprint:
  hogehogehogehoge

ℹ  Configure Kibana to use this cluster:
• Run Kibana and click the configuration link in the terminal when Kibana starts.
• Copy the following enrollment token and paste it into Kibana in your browser (valid for the next 30 minutes):
  hogehogehogehoge

ℹ  Configure other nodes to join this cluster:
• On this node:
  ⁃ Create an enrollment token with `bin/elasticsearch-create-enrollment-token -s node`.
  ⁃ Uncomment the transport.host setting at the end of config/elasticsearch.yml.
  ⁃ Restart Elasticsearch.
• On other nodes:
  ⁃ Start Elasticsearch with `bin/elasticsearch --enrollment-token <token>`, using the enrollment token that you generated.
```

tmuxで別のウィンドウを開く。

```bash
# 疎通確認

curl --cacert config/certs/http_ca.crt -u elastic https://localhost:9200

Enter host password for user 'elastic':

# 先程表示されたパスワードを入力する
```

うまくいくと以下のようなレスポンスがある。

```json
{
  "name" : "ubuntu2204.localdomain",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "c2uKCkawTL6QZV-dgYD9Ng",
  "version" : {
    "number" : "8.4.3",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "42f05b9372a9a4a470db3b52817899b99a76ee73",
    "build_date" : "2022-10-04T07:17:24.662462378Z",
    "build_snapshot" : false,
    "lucene_version" : "9.3.0",
    "minimum_wire_compatibility_version" : "7.17.0",
    "minimum_index_compatibility_version" : "7.0.0"
  },
  "tagline" : "You Know, for Search"
}
```

## Kibanaの導入と設定

こちらも同様に公式ドキュメントに沿って作業を行う。  

Install Kibana from archive on Linux or macOS | Kibana Guide [8.4] | Elastic : https://www.elastic.co/guide/en/kibana/8.4/targz.html

```bash
cd ~

curl -O https://artifacts.elastic.co/downloads/kibana/kibana-8.4.3-linux-x86_64.tar.gz

tar -xzf kibana-8.4.3-linux-x86_64.tar.gz

cd kibana-8.4.3/

# 設定ファイルの編集

vim config/kibana.yml

# hostを0.0.0.0にする

# 起動する
./bin/kibana
```

tmuxで別のウィンドウを開く。

```bash
cd ~/elasticsearch-8.4.3/

bin/elasticsearch-create-enrollment-token -s kibana

# hogehogehoge

# kibanaのwebの画面に入力する

cd ~/kibana-8.4.3

./bin/kibana-verification-code

# Your verification code is: 123 123

# kibanaのwebの画面に入力する
```

tmuxで別のウィンドウを開く。

## Filebeatの導入と設定

そろそろ集中力も切れてきた。  
Filebeatだけは/var/log/以下のファイルを参照しようとしたときに権限がなくて怒られたりするので、dpkgで入れたりすると楽だったりする。  
意識が低いですね。  

Filebeat quick start: installation and configuration | Filebeat Reference [8.4] | Elastic : https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-installation-configuration.html


```bash
cd ~

curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.4.3-amd64.deb

sudo dpkg -i filebeat-8.4.3-amd64.deb

sudo vim /etc/filebeat/filebeat.yml
```

設定ファイルで2点、変更を加える。

Elasticsearchの吐き出し先と

```yml
# ---------------------------- Elasticsearch Output ----------------------------
output.elasticsearch:
  # Array of hosts to connect to.
  hosts: ["localhost:9200"]
  username: "elastic"
  password: "bGFikao*2cn3ZEA84Ar+"
  ssl:
    enabled: true
    ca_trusted_fingerprint: "37a192ebcaea1bc9811eebb95c5eae47f62aa2ce0d34707b23e8f52ff01f5074"

  # Protocol - either `http` (default) or `https`.
  protocol: "https"
```

Kibanaのホスト

```yml
# =================================== Kibana ===================================

# Starting with Beats version 6.0.0, the dashboards are loaded via the Kibana API.
# This requires a Kibana endpoint configuration.
setup.kibana:

  # Kibana Host
  # Scheme and port can be left out and will be set to the default (http and 5601)
  # In case you specify and additional path, the scheme is required: http://localhost:5601/path
  # IPv6 addresses should always be defined as: https://[2001:db8::1]:5601
  host: "0.0.0.0:5601"

  # Kibana Space ID
  # ID of the Kibana Space into which the dashboards should be loaded. By default,
  # the Default Space will be used.
  #space.id:

# =============================== Elastic Cloud ================================
```

今回はsyslogを取ってみるテスト、ということで次の設定ファイルを書き換える。

```bash
sudo vim /etc/filebeat/modules.d/system.yml.disabled
```

enabledをtrueにするだけ。  
詳しいことはドキュメントを読んで下さい。  

```yml
# Module: system
# Docs: https://www.elastic.co/guide/en/beats/filebeat/8.4/filebeat-module-system.html

- module: system
  # Syslog
  syslog:
    enabled: true

    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    #var.paths:

  # Authorization logs
  auth:
    enabled: true

    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    #var.paths:
```

```bash
# モジュールを有効にする
sudo mv /etc/filebeat/modules.d/system.yml.disabled /etc/filebeat/modules.d/system.yml

# セットアップ
sudo filebeat setup -e

# 起動
sudo systemctl status filebeat.service
```

Kibanaでデータが入っていることが確認できれば完成。
