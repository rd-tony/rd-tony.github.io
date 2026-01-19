---
title: Ubuntu 24.04 Install PostgreSQL Cluster
tags:
  - Database
  - PostgreSQL
date: 2026-01-19 18:20:34
---

{% note info %}

### 虛擬機

{% endnote %}

<table>
    <tr align="center">
        <th>HostName</th>
        <th>IP Address</th>
        <th>安裝項目</th>
        <th>Virtual IP</th>
    </tr>
    <tr align="center">
        <td>pgha-01</td>
        <td>172.30.1.211</td>
        <td rowspan="3" align="left">
            <ul>
                <li>HAProxy</li>
                <li>Keepalived</li>
            </ul>
        </td>
        <td rowspan="3" align="center">Virtual IP: 172.30.1.220</td>
    </tr>
    <tr align="center">
        <td>pgha-02</td>
        <td>172.30.1.212</td>
    </tr>
    <tr align="center">
        <td>pgha-03</td>
        <td>172.30.1.213</td>
    </tr>
</table>

<table>
    <tr align="center">
        <th>HostName</th>
        <th>IP Address</th>
        <th>安裝項目</th>
        <th>Virtual IP</th>
    </tr>
    <tr align="center">
        <td>pgsql-01</td>
        <td>172.30.1.221</td>
        <td rowspan="3" align="left">
            <ul>
                <li>etcd</li>
                <li>Patroni</li>
                <li>PostgresSQL</li>
            </ul>
        </td>
        <td rowspan="3" align="center">
            Virtual IP: 172.30.1.221
            </td>
    </tr>
    <tr align="center">
        <td>pgsql-02</td>
        <td>172.30.1.222</td>
    </tr>
    <tr align="center">
        <td>pgsql-03</td>
        <td>172.30.1.223</td>
    </tr>
</table>

{% note info %}

### PostgresSQL

{% endnote %}

```bash pgsql 各主機安裝 postgresql、postgresql-client
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh # 留意是否需要設定 proxy
sudo apt install -y postgresql-18 postgresql-client-18
```

```bash pgsql 各主機停用 postgresql
sudo systemctl stop postgresql
sudo systemctl disable postgresql

# 或
sudo systemctl disable --now postgresql
```

{% note info %}

### etcd

{% endnote %}

#### <i class="fa-solid fa-gear"></i> 安裝 etcd

```bash pgsql 各主機安裝 etcd
wget https://github.com/etcd-io/etcd/releases/download/v3.6.7/etcd-v3.6.7-linux-amd64.tar.gz
tar xvf etcd-v3.6.7-linux-amd64.tar.gz
sudo mv etcd-3.6.7-linux-amd64/ /usr/local/bin/etcd/

# 檢查 etcd 版本
etcd --version
etcdctl version

# 建立 etcd 服務的使用者
sudo useradd --system --home /var/lib/etcd --shell /bin/false etcd
```

#### <i class="fa-solid fa-gear"></i> 憑證 (Certificates)

在使用者本機建立憑證

```bash
sudo apt install openssl
openssl version
mkdir certs
cd certs
```

```bash 產生 Certificate Authority
openssl genrsa -out ca.key 2048
openssl req -x509 -new -nodes -key ca.key -subj "/CN=etcd-ca" -days 7300 -out ca.crt
```

產生各主機的 Certificate (留意主機 IP 與主機名稱)：
{% tabs Tabs1, 1 %}
<!-- tab pgsql-01@fa-solid fa-file-code -->
```bash
# 產生私鑰
openssl genrsa -out pgsql-01.key 2048

# 產生暫存檔
cat > temp.cnf <<EOF
[ req ]
distinguished_name = req_distinguished_name
req_extensions = v3_req
[ req_distinguished_name ]
[ v3_req ]
subjectAltName = @alt_names
[ alt_names ]
IP.1 = 172.30.1.221
IP.2 = 127.0.0.1
EOF

# 建立 Certificate Signing Request
openssl req -new -key pgsql-01.key -out pgsql-01.csr \
  -subj "/C=TW/ST=YourState/L=YourCity/O=YourOrganization/OU=YourUnit/CN=pgsql-01" \
  -config temp.cnf

# 簽署憑證
openssl x509 -req -in pgsql-01.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out pgsql-01.crt -days 7300 -sha256 -extensions v3_req -extfile temp.cnf

# 驗證並確認 Subject Name Alternative
openssl x509 -in pgsql-01.crt -text -noout | grep -A1 "Subject Alternative Name"

# 刪除暫存檔
rm temp.cnf
```
<!-- endtab -->
<!-- tab pgsql-02@fa-solid fa-file-code -->
```bash
# 產生私鑰
openssl genrsa -out pgsql-02.key 2048

# 產生暫存檔
cat > temp.cnf <<EOF
[ req ]
distinguished_name = req_distinguished_name
req_extensions = v3_req
[ req_distinguished_name ]
[ v3_req ]
subjectAltName = @alt_names
[ alt_names ]
IP.1 = 172.30.1.222
IP.2 = 127.0.0.1
EOF

# 建立 Certificate Signing Request
openssl req -new -key pgsql-02.key -out pgsql-02.csr \
  -subj "/C=TW/ST=YourState/L=YourCity/O=YourOrganization/OU=YourUnit/CN=pgsql-02" \
  -config temp.cnf

# 簽署憑證
openssl x509 -req -in pgsql-02.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out pgsql-02.crt -days 7300 -sha256 -extensions v3_req -extfile temp.cnf

# 驗證並確認 Subject Name Alternative
openssl x509 -in pgsql-02.crt -text -noout | grep -A1 "Subject Alternative Name"

# 刪除暫存檔
rm temp.cnf
```
<!-- endtab -->
<!-- tab pgsql-03@fa-solid fa-file-code -->
```bash
# 產生私鑰
openssl genrsa -out pgsql-03.key 2048

# 產生暫存檔
cat > temp.cnf <<EOF
[ req ]
distinguished_name = req_distinguished_name
req_extensions = v3_req
[ req_distinguished_name ]
[ v3_req ]
subjectAltName = @alt_names
[ alt_names ]
IP.1 = 172.30.1.223
IP.2 = 127.0.0.1
EOF

# 建立 Certificate Signing Request
openssl req -new -key pgsql-03.key -out pgsql-03.csr \
  -subj "/C=TW/ST=YourState/L=YourCity/O=YourOrganization/OU=YourUnit/CN=pgsql-03" \
  -config temp.cnf

# 簽署憑證
openssl x509 -req -in pgsql-03.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out pgsql-03.crt -days 7300 -sha256 -extensions v3_req -extfile temp.cnf

# 驗證並確認 Subject Name Alternative
openssl x509 -in pgsql-03.crt -text -noout | grep -A1 "Subject Alternative Name"

# 刪除暫存檔
rm temp.cnf
```
<!-- endtab -->
{% endtabs %}

```bash 將憑證複製至各 pgsql 主機
scp ca.crt pgsql-01.crt pgsql-01.key pguser@172.30.1.221:/tmp/
scp ca.crt pgsql-02.crt pgsql-02.key pguser@172.30.1.222:/tmp/
scp ca.crt pgsql-03.crt pgsql-03.key pguser@172.30.1.223:/tmp/
```

在 pgsql 各主機上執行 (留意主機 IP 與主機名稱)：
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
```

#### <i class="fa-solid fa-gear"></i> 設定

```bash
sudo nano /etc/etcd/etcd.env
```

```conf 依下列內容設定
ETCD_NAME="pgsql-01"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_INITIAL_CLUSTER="pgsql-01=https://172.30.1.221:2380,pgsql-02=https://172.30.1.222:2380,pgsql-03=https://172.30.1.223:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://172.30.1.221:2380"
ETCD_LISTEN_PEER_URLS="https://0.0.0.0:2380"
ETCD_LISTEN_CLIENT_URLS="https://0.0.0.0:2379"
ETCD_ADVERTISE_CLIENT_URLS="https://172.30.1.221:2379"
ETCD_CLIENT_CERT_AUTH="true"
ETCD_TRUSTED_CA_FILE="/etc/etcd/ssl/ca.crt"
ETCD_CERT_FILE="/etc/etcd/ssl/pgsql-01.crt"
ETCD_KEY_FILE="/etc/etcd/ssl/pgsql-01.key"
ETCD_PEER_CLIENT_CERT_AUTH="true"
ETCD_PEER_TRUSTED_CA_FILE="/etc/etcd/ssl/ca.crt"
ETCD_PEER_CERT_FILE="/etc/etcd/ssl/pgsql-01.crt"
ETCD_PEER_KEY_FILE="/etc/etcd/ssl/pgsql-01.key"
```
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
```

```bash
sudo nano /etc/etcd/etcd.env
```

```conf 依下列內容設定
ETCD_NAME="pgsql-02"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_INITIAL_CLUSTER="pgsql-01=https://172.30.1.221:2380,pgsql-02=https://172.30.1.222:2380,pgsql-03=https://172.30.1.223:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://172.30.1.222:2380"
ETCD_LISTEN_PEER_URLS="https://0.0.0.0:2380"
ETCD_LISTEN_CLIENT_URLS="https://0.0.0.0:2379"
ETCD_ADVERTISE_CLIENT_URLS="https://172.30.1.222:2379"
ETCD_CLIENT_CERT_AUTH="true"
ETCD_TRUSTED_CA_FILE="/etc/etcd/ssl/ca.crt"
ETCD_CERT_FILE="/etc/etcd/ssl/pgsql-02.crt"
ETCD_KEY_FILE="/etc/etcd/ssl/pgsql-02.key"
ETCD_PEER_CLIENT_CERT_AUTH="true"
ETCD_PEER_TRUSTED_CA_FILE="/etc/etcd/ssl/ca.crt"
ETCD_PEER_CERT_FILE="/etc/etcd/ssl/pgsql-02.crt"
ETCD_PEER_KEY_FILE="/etc/etcd/ssl/pgsql-02.key"
```
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
```

```bash
sudo nano /etc/etcd/etcd.env
```

```conf 依下列內容設定
ETCD_NAME="pgsql-03"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_INITIAL_CLUSTER="pgsql-01=https://172.30.1.221:2380,pgsql-02=https://172.30.1.222:2380,pgsql-03=https://172.30.1.223:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://172.30.1.223:2380"
ETCD_LISTEN_PEER_URLS="https://0.0.0.0:2380"
ETCD_LISTEN_CLIENT_URLS="https://0.0.0.0:2379"
ETCD_ADVERTISE_CLIENT_URLS="https://172.30.1.223:2379"
ETCD_CLIENT_CERT_AUTH="true"
ETCD_TRUSTED_CA_FILE="/etc/etcd/ssl/ca.crt"
ETCD_CERT_FILE="/etc/etcd/ssl/pgsql-03.crt"
ETCD_KEY_FILE="/etc/etcd/ssl/pgsql-03.key"
ETCD_PEER_CLIENT_CERT_AUTH="true"
ETCD_PEER_TRUSTED_CA_FILE="/etc/etcd/ssl/ca.crt"
ETCD_PEER_CERT_FILE="/etc/etcd/ssl/pgsql-03.crt"
ETCD_PEER_KEY_FILE="/etc/etcd/ssl/pgsql-03.key"
```
<!-- endtab -->
{% endtabs %}

#### <i class="fa-solid fa-gear"></i> Create etcd service

```bash
sudo nano /etc/systemd/system/etcd.service
```

```bash
[Unit]
Description=etcd key-value store
Documentation=https://github.com/etcd-io/etcd
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd
EnvironmentFile=/etc/etcd/etcd.env
ExecStart=/usr/local/bin/etcd
Restart=always
RestartSec=10s
LimitNOFILE=40000
User=etcd
Group=etcd

[Install]
WantedBy=multi-user.target
```

```bash
# create a directory for ETCD_DATA_DIR
sudo mkdir -p /var/lib/etcd 
sudo chown -R etcd:etcd /var/lib/etcd

# Reload daemon and enable the service
sudo systemctl daemon-reload
sudo systemctl enable etcd

# Start etcd and check status
sudo systemctl start etcd
sudo systemctl status etcd
```

{% note info %}

### Patroni

{% endnote %}

#### <i class="fa-solid fa-gear"></i> Installing Patroni

```bash
sudo apt install -y patroni
sudo mkdir -p /etc/patroni/
```

#### <i class="fa-solid fa-gear"></i> Configuring Patroni

```bash
sudo nano /etc/patroni/config.yml
```

#### <i class="fa-solid fa-gear"></i> PostgreSQL Certificates

```bash
openssl genrsa -out server.key 2048 # private key
openssl req -new -key server.key -out server.req # csr
openssl req -x509 -key server.key -in server.req -out server.crt -days 7300 # generate cert, valid for 20 years
chmod 600 server.key

scp server.crt server.key server.req pguser@172.30.1.221:/tmp
scp server.crt server.key server.req pguser@172.30.1.222:/tmp
scp server.crt server.key server.req pguser@172.30.1.223:/tmp
```

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

{% tabs Tabs3, 1 %}
<!-- tab pgsql-01@fa-solid fa-file-code -->
```bash
sudo setfacl -m u:postgres:r /etc/etcd/ssl/ca.crt
sudo setfacl -m u:postgres:r /etc/etcd/ssl/pgsql-01.crt
sudo setfacl -m u:postgres:r /etc/etcd/ssl/pgsql-01.key
```
<!-- endtab -->
<!-- tab pgsql-02 @fa-solid fa-file-code-->
```bash
sudo setfacl -m u:postgres:r /etc/etcd/ssl/ca.crt
sudo setfacl -m u:postgres:r /etc/etcd/ssl/pgsql-02.crt
sudo setfacl -m u:postgres:r /etc/etcd/ssl/pgsql-02.key
```
<!-- endtab -->
<!-- tab pgsql-03@fa-solid fa-file-code -->
```bash
sudo setfacl -m u:postgres:r /etc/etcd/ssl/ca.crt
sudo setfacl -m u:postgres:r /etc/etcd/ssl/pgsql-03.crt
sudo setfacl -m u:postgres:r /etc/etcd/ssl/pgsql-03.key
```
<!-- endtab -->
{% endtabs %}

```bash
sudo sh -c 'cat /var/lib/postgresql/ssl/server.crt /var/lib/postgresql/ssl/server.key > /var/lib/postgresql/ssl/server.pem'
sudo chown postgres:postgres /var/lib/postgresql/ssl/server.pem
sudo chmod 600 /var/lib/postgresql/ssl/server.pem

sudo openssl x509 -in /var/lib/postgresql/ssl/server.pem -text -noout
```

#### <i class="fa-solid fa-gear"></i> Starting Patroni and HA Cluster

```bash
sudo systemctl restart patroni
journalctl -u patroni -f
```

#### <i class="fa-solid fa-gear"></i> Reconfiguring our etcd Cluster

```bash
sudo nano /etc/etcd/etcd.env
```

```bash
# change
ETCD_INITIAL_CLUSTER_STATE="new"

# to
ETCD_INITIAL_CLUSTER_STATE="existing"
```

#### <i class="fa-solid fa-gear"></i> Verifying Our Postgres Cluster

```bash
curl -k https://172.30.1.221:8008/primary
curl -k https://172.30.1.222:8008/primary
curl -k https://172.30.1.223:8008/primary
```

```bash show config
sudo patronictl -c /etc/patroni/config.yml show-config
```

```bash edit config
sudo patronictl -c /etc/patroni/config.yml edit-config
```

{% note info %}

### HAProxy

{% endnote %}

{% note info %}

### Keepalived

{% endnote %}

參考連結：

* [PostgresSQL Clustering the hard way][1]
* [Building a Highly Available PostgreSQL Cluster with Patroni, etcd, and PgBouncer][2]

[1]: https://technotim.com/posts/postgresql-high-availability/#configuring-patroni "PostgresSQL Clustering the hard way"

[2]: https://medium.com/@yahya.muhaned/building-a-highly-available-postgresql-cluster-with-patroni-etcd-and-pgbouncer-f8e363342bf4 "Building a Highly Available PostgreSQL Cluster with Patroni, etcd, and PgBouncer"
