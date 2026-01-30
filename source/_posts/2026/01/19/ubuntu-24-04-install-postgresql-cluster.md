---
title: Ubuntu 24.04 Install PostgreSQL Cluster
tags:
  - Database
  - PostgreSQL
date: 2026-01-19 18:20:34
---

{% note info %}
## 虛擬機
{% endnote %}

#### <span style="color: #0088FF;">HAProxy</span>
<table style="table-layout: fixed; width: 400px;">
    <tr align="center">
        <th style="width: 100px;" align="center">HostName</th>
        <th style="width: 100px;" align="center">IP Address</th>
        <th style="width: 150px;" align="center">安裝項目</th>
    </tr>
    <tr align="left">
        <td align="center">pgha-01</td>
        <td align="center">172.30.1.211</td>
        <td align="left" rowspan="3">
            <ul>
                <li>HAProxy</li>
                <li>Keepalived</li>
            </ul>
        </td>
    </tr>
    <tr align="center">
        <td align="center">pgha-02</td>
        <td align="center">172.30.1.212</td>
    </tr>
    <tr align="center">
        <td align="center">pgha-03</td>
        <td align="center">172.30.1.213</td>
    </tr>
</table>

#### <span style="color: #0088FF;">PostgreSQL</span>

<table style="table-layout: fixed; width: 600px;">
    <tr align="center">
        <th style="width: 100px;" align="center">HostName</th>
        <th style="width: 100px;" align="center">IP Address</th>
        <th style="width: 150px;" align="center">安裝項目</th>
        <th style="width: 150px;" align="center">Virtual IP</th>
    </tr>
    <tr align="center">
        <td align="center">pgsql-01</td>
        <td align="center">172.30.1.221</td>
        <td align="left" rowspan="3">
            <ul>
                <li>etcd</li>
                <li>Patroni</li>
                <li>PostgresSQL</li>
            </ul>
        </td>
        <td align="center" rowspan="3">
            Virtual IP: 172.30.1.220
        </td>
    </tr>
    <tr align="center">
        <td align="center">pgsql-02</td>
        <td align="center">172.30.1.222</td>
    </tr>
    <tr align="left">
        <td align="center">pgsql-03</td>
        <td align="center">172.30.1.223</td>
    </tr>
</table>

{% note info %}
## PostgreSQL
{% endnote %}

#### <span style="color: #0088FF;">安裝</span>

```bash pgsql 各節點
# 安裝 postgresql、postgresql-client
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh # 留意是否需要設定 proxy
sudo apt install -y postgresql-18 postgresql-client-18
```

#### <span style="color: #0088FF;">停用服務</span>

```bash pgsql 各節點
sudo systemctl stop postgresql
sudo systemctl disable postgresql

# 或
sudo systemctl disable --now postgresql
```

#### <span style="color: #0088FF;">憑證</span>

<i class="fa-solid fa-triangle-exclamation"></i> {% label danger @以下在使用者端進行 %} <i class="fa-solid fa-triangle-exclamation"></i>

```bash
openssl genrsa -out server.key 2048 # private key
openssl req -new -key server.key -out server.req # csr
openssl req -x509 -key server.key -in server.req -out server.crt -days 7300 # generate cert, valid for 20 years
chmod 600 server.key

scp server.crt server.key server.req pguser@172.30.1.221:/tmp
scp server.crt server.key server.req pguser@172.30.1.222:/tmp
scp server.crt server.key server.req pguser@172.30.1.223:/tmp
```

<i class="fa-solid fa-triangle-exclamation"></i> {% label danger @以下在 pgsql 各節點進行 %} <i class="fa-solid fa-triangle-exclamation"></i>

```bash
sudo mkdir -p /var/lib/postgresql/data
sudo mkdir -p /var/lib/postgresql/ssl

cd /tmp
sudo mv server.crt server.key server.req /var/lib/postgresql/ssl

sudo chmod 600 /var/lib/postgresql/ssl/server.key
sudo chmod 644 /var/lib/postgresql/ssl/server.crt
sudo chmod 600 /var/lib/postgresql/ssl/server.req
sudo chown postgres:postgres /var/lib/postgresql/data
sudo chown postgres:postgres /var/lib/postgresql/ssl/server.*

sudo apt update
sudo apt install -y acl
```

{% tabs Tabs0, 1 %}
<!-- tab pgsql-01@fa-solid fa-file-code -->
```conf
sudo setfacl -m u:postgres:r /etc/etcd/ssl/ca.crt
sudo setfacl -m u:postgres:r /etc/etcd/ssl/pgsql-01.crt
sudo setfacl -m u:postgres:r /etc/etcd/ssl/pgsql-01.key
```
<!-- endtab -->

<!-- tab pgsql-02@fa-solid fa-file-code -->
```conf
sudo setfacl -m u:postgres:r /etc/etcd/ssl/ca.crt
sudo setfacl -m u:postgres:r /etc/etcd/ssl/pgsql-02.crt
sudo setfacl -m u:postgres:r /etc/etcd/ssl/pgsql-02.key
```
<!-- endtab -->

<!-- tab pgsql-03@fa-solid fa-file-code -->
```conf
sudo setfacl -m u:postgres:r /etc/etcd/ssl/ca.crt
sudo setfacl -m u:postgres:r /etc/etcd/ssl/pgsql-03.crt
sudo setfacl -m u:postgres:r /etc/etcd/ssl/pgsql-03.key
```
<!-- endtab -->
{% endtabs %}

{% note info %}
## etcd
{% endnote %}

#### <span style="color: #0088FF;">安裝</span>

{% codeblock "pgsql 各節點" lang:bash <https://github.com/etcd-io/etcd/releases> "下載 etcd 最新版本" %}
sudo apt update
sudo apt-get install -y wget curl

wget <https://github.com/etcd-io/etcd/releases/download/v3.6.7/etcd-v3.6.7-linux-amd64.tar.gz>
tar xvf etcd-v3.6.7-linux-amd64.tar.gz
sudo mv etcd-3.6.7-linux-amd64/ /usr/local/bin/etcd/

# 建立 etcd 服務的使用者
sudo useradd --system --home /var/lib/etcd --shell /bin/false etcd
{% endcodeblock %}

```bash pgsql 各節點
# 檢查 etcd 版本
etcd --version
etcdctl version
```

<!-- {% asset_img etcd-version.png 300 "etcd version'etcd version'" %} -->

<img src="{% asset_path etcd-version.png %}" alt="etcd version" title="etct version" width="300" height="auto">

#### <span style="color: #0088FF;">建立憑證</span>

<i class="fa-solid fa-triangle-exclamation"></i> {% label danger @以下在使用者端進行 %} <i class="fa-solid fa-triangle-exclamation"></i>

```bash
# 安裝 openssl
sudo apt install openssl

# 確認 openssl 版本
openssl version

# 建立存放憑證的路徑
mkdir certs
cd certs

# 產生 Certificate Authority
openssl genrsa -out ca.key 2048
openssl req -x509 -new -nodes -key ca.key -subj "/CN=etcd-ca" -days 7300 -out ca.crt
```

產生 pgsql 各節點的憑證 (留意主機 IP 與主機名稱)：
{% tabs Tabs1, 1 %}
<!-- tab pgsql-01@fa-solid fa-file-code -->
{% gist e394fbeea2854b3f9d1e7587415cdf93 %}
<!-- endtab -->

<!-- tab pgsql-02@fa-solid fa-file-code -->
{% gist 80cba7ecd01915af82c3ca3a9abf168d %}
<!-- endtab -->

<!-- tab pgsql-03@fa-solid fa-file-code -->
{% gist 1638e3fdd42661b949cb4fda0275cbf7 %}
<!-- endtab -->
{% endtabs %}

```bash 將憑證複製至各 pgsql 節點
scp ca.crt pgsql-01.crt pgsql-01.key pguser@172.30.1.221:/tmp/
scp ca.crt pgsql-02.crt pgsql-02.key pguser@172.30.1.222:/tmp/
scp ca.crt pgsql-03.crt pgsql-03.key pguser@172.30.1.223:/tmp/
```

#### <span style="color: #0088FF;">設定</span>

<i class="fa-solid fa-triangle-exclamation"></i> {% label danger @以下在 pgsql 各節點進行 %} <i class="fa-solid fa-triangle-exclamation"></i>

{% tabs Tabs2, 1 %}
<!-- tab pgsql-01@fa-solid fa-file-code -->
```bash
sudo mkdir -p /etc/etcd
sudo mkdir -p /etc/etcd/ssl

sudo mv /tmp/pgsql-01.crt /etc/etcd/ssl/
sudo mv /tmp/pgsql-01.key /etc/etcd/ssl/
sudo mv /tmp/ca.crt /etc/etcd/ssl/

sudo chown -R etcd:etcd /etc/etcd/
sudo chmod 600 /etc/etcd/ssl/pgsql-01.key
sudo chmod 644 /etc/etcd/ssl/pgsql-01.crt /etc/etcd/ssl/ca.crt

# 編輯 /etc/etcd/etcd.env
sudo nano /etc/etcd/etcd.env
```

{% gist 6b9b97597f4a8ac3b22868f3bd78dbed %}
<!-- endtab -->

<!-- tab pgsql-02@fa-solid fa-file-code -->
```bash
sudo mkdir -p /etc/etcd
sudo mkdir -p /etc/etcd/ssl

sudo mv /tmp/pgsql-02.crt /etc/etcd/ssl/
sudo mv /tmp/pgsql-02.key /etc/etcd/ssl/
sudo mv /tmp/ca.crt /etc/etcd/ssl/

sudo chown -R etcd:etcd /etc/etcd/
sudo chmod 600 /etc/etcd/ssl/pgsql-02.key
sudo chmod 644 /etc/etcd/ssl/pgsql-02.crt /etc/etcd/ssl/ca.crt

# 編輯 /etc/etcd/etcd.env
sudo nano /etc/etcd/etcd.env
```

{% gist 61ca18d25cb42cefabeba20c10551733 %}
<!-- endtab -->

<!-- tab pgsql-03@fa-solid fa-file-code -->
```bash
sudo mkdir -p /etc/etcd
sudo mkdir -p /etc/etcd/ssl

sudo mv /tmp/pgsql-03.crt /etc/etcd/ssl/
sudo mv /tmp/pgsql-03.key /etc/etcd/ssl/
sudo mv /tmp/ca.crt /etc/etcd/ssl/

sudo chown -R etcd:etcd /etc/etcd/
sudo chmod 600 /etc/etcd/ssl/pgsql-03.key
sudo chmod 644 /etc/etcd/ssl/pgsql-03.crt /etc/etcd/ssl/ca.crt

# 編輯 /etc/etcd/etcd.env
sudo nano /etc/etcd/etcd.env
```

{% gist 2a2611d84881e558a01861748b55c284 %}
<!-- endtab -->
{% endtabs %}

#### <span style="color: #0088FF;">啟動服務</span>

```bash 新增 etcd.service
sudo nano /etc/systemd/system/etcd.service
```

{% gist fd5294827414603fbfc28d7bb872d126 %}

```bash
# 建立 ETCD_DATA_DIR 變數路徑
sudo mkdir -p /var/lib/etcd 
sudo chown -R etcd:etcd /var/lib/etcd

# 啟用服務
sudo systemctl daemon-reload
sudo systemctl enable etcd

# 啟動服務並檢查狀態
sudo systemctl start etcd
sudo systemctl status etcd
```

#### <span style="color: #0088FF;">驗證</span>

```bash
sudo etcdctl \
--endpoints=https://172.30.1.221:2379,https://172.30.1.222:2379,https://172.30.1.223:2379 \
--cacert=/etc/etcd/ssl/ca.crt \
--cert=/etc/etcd/ssl/$(hostname).crt \
--key=/etc/etcd/ssl/$(hostname).key \
endpoint status --write-out=table
```

{% asset_img etcd.png %}

{% note info %}
## Patroni
{% endnote %}

#### <span style="color: #0088FF;">安裝</span>

```bash
sudo apt install -y patroni
```

#### <span style="color: #0088FF;">設定</span>

```bash
sudo mkdir -p /etc/patroni/
sudo nano /etc/patroni/config.yml
```

{% tabs Tabs3, 1 %}
<!-- tab pgsql-01@fa-solid fa-file-code -->
{% gist 7ed36778daad57acc29f85255eff24f8 %}
<!-- endtab -->

<!-- tab pgsql-02 @fa-solid fa-file-code-->
{% gist 24c47304e01fc6cac14604d217a0ea9e %}
<!-- endtab -->

<!-- tab pgsql-03@fa-solid fa-file-code -->
{% gist 31d09d35e81eef0af970cb146276fd7c %}
<!-- endtab -->
{% endtabs %}

#### <span style="color: #0088FF;">憑證</span>

```bash
sudo sh -c 'cat /var/lib/postgresql/ssl/server.crt /var/lib/postgresql/ssl/server.key > /var/lib/postgresql/ssl/server.pem'
sudo chown postgres:postgres /var/lib/postgresql/ssl/server.pem
sudo chmod 600 /var/lib/postgresql/ssl/server.pem

sudo openssl x509 -in /var/lib/postgresql/ssl/server.pem -text -noout
```

#### <span style="color: #0088FF;">啟動</span>

```bash
# 重啟 Patroni 服務
sudo systemctl restart patroni

# 檢查 log
journalctl -u patroni -f
```

#### <span style="color: #0088FF;">修改 etcd 設定</span>

```bash 修改&nbsp;/etc/etcd/etcd.env
sudo nano /etc/etcd/etcd.env
```

```bash
# 修改前
ETCD_INITIAL_CLUSTER_STATE="new"

# 修改後
ETCD_INITIAL_CLUSTER_STATE="existing"
```

```bash 重新啟動 etcd
sudo systemctl daemon-reload
sudo systemctl restart etcd
```

#### <span style="color: #0088FF;">驗證</span>

```bash
curl -k https://172.30.1.221:8008/primary
curl -k https://172.30.1.222:8008/primary
curl -k https://172.30.1.223:8008/primary
```

```bash
sudo patronictl -c /etc/patroni/config.yml list
```

<!-- {% asset_img patroni.png %} -->

<img src="{% asset_path patroni.png %}" alt="patroni" title="patroni" width="800" height="auto">

#### <span style="color: #0088FF;">修改 Patroni 設定</span>

```bash 顯示設定檔
sudo patronictl -c /etc/patroni/config.yml show-config
```

```bash 編輯設定檔
sudo patronictl -c /etc/patroni/config.yml edit-config
```

{% note info %}
## HAProxy
{% endnote %}

#### <span style="color: #0088FF;">安裝</span>

```bash
sudo apt -y install haproxy
```

#### <span style="color: #0088FF;">設定</span>

```bash
sudo nano /etc/haproxy/haproxy.cfg
```

{% gist 3ea9bdc82bfc2e2b558d083bdf168c29 %}

#### <span style="color: #0088FF;">啟動</span>

```bash
sudo systemctl reload haproxy
sudo systemctl enable --now haproxy
```

#### <span style="color: #0088FF;">檢查</span>

```bash
sudo tail -f /var/log/syslog | grep haproxy
```

{% note info %}
## Keepalived
{% endnote %}

#### <span style="color: #0088FF;">安裝</span>

```bash
sudo apt update
sudo apt install keepalived -y
```

#### <span style="color: #0088FF;">設定</span>

```bash
sudo nano /etc/keepalived/keepalived.conf
```

{% tabs Tabs5, 1 %}
<!-- tab pgsql-01@fa-solid fa-file-code -->
{% gist 0443af1ab0a0e0c2eaf4c5f287fd5a2b %}
<!-- endtab -->

<!-- tab pgsql-02 @fa-solid fa-file-code-->
{% gist 298f85fff801ecce88cb673508e2963d %}
<!-- endtab -->

<!-- tab pgsql-03@fa-solid fa-file-code -->
{% gist 40a49f822a4e6e998a9d3a757a2442c7 %}
<!-- endtab -->
{% endtabs %}

```bash 建立&nbsp;/etc/keepalived/check_haproxy.sh
sudo nano /etc/keepalived/check_haproxy.sh
```

{% gist 0ba3c9251e2cb650c9fa6f65862e3744 %}

```bash
sudo useradd -r -s /bin/false keepalived_script

# 權限
sudo chmod +x /etc/keepalived/check_haproxy.sh
sudo chown keepalived_script:keepalived_script /etc/keepalived/check_haproxy.sh
sudo chmod 700 /etc/keepalived/check_haproxy.sh
```

#### <span style="color: #0088FF;">啟動</span>

```bash
sudo systemctl restart keepalived
```

#### <span style="color: #0088FF;">驗證</span>

```bash
sudo journalctl -u keepalived -f
```
