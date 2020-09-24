---
layout: post
title:  1.15.3 HA kubernetes Cluster Playbook (etcd cluster)
date:   2020-08-30 21:38:45 +0800
categories: ["kubernetes", "centos"]
tags: ["kuberentes", "centos"]
excerpt_separator: <!--more-->
---

## Objective
<img src="{{site.url}}/assets/images/external-etcd-topology-etcd-cluster.png" style="width: 666px;" />

<!--more-->

### variables
<div class="alert alert-success" role="alert">
<i class="fa fa-check-square-o"></i>
<b>Tip: </b>execute the variables in all console (masters) at the very begining, make sure all servers are using the exact same value (and avoid manual input)
</div>

```bash
## change if necessary
# hostname
master01Name='master01'
master02Name='master02'
master03Name='master03'

# ipaddress
master01IP='192.168.100.200'
master01IP='192.168.100.201'
master01IP='192.168.100.202'
virtualIP='192.168.100.250'

leadIP="${master01IP}"
leadName="${master01Name}"

k8sVer='v1.15.3'
cfsslDownloadUrl='https://pkg.cfssl.org/R1.2'

etcdVer='v3.3.15'
etcdDownloadUrl='https://github.com/etcd-io/etcd/releases/download'
etcdSSLPath='/etc/etcd/ssl'
etcdInitialCluster="${master01Name}=https://${master01IP}:2380,${master02Name}=https://${master02IP}:2380,${master03Name}=https://${master03IP}:2380"

keepaliveVer='2.0.18'
haproxyVer='2.0.6'
helmVer='v2.14.3'

interface=$(netstat -nr | grep -E 'UG|UGSc' | grep -E '^0.0.0|default' | grep -E '[0-9.]{7,15}' | awk -F' ' '{print $NF}')
ipAddr=$(ip a s "${interface}" | sed -rn 's|\W*inet[^6]\W*([0-9\.]{7,15}).*$|\1|p')
peerName=$(hostname)
```
<br>

## Certificates
<div class="alert alert-success" role="alert">
<i class="fa fa-check-square-o"></i>
<b>Tip: </b> setup certificate in one of master
</div>

```bash
$ sudo mkdir -p ${etcdSSLPath}
# cd ${etcdSSLPath}
```

- Certificate Signing Request
  ```bash
  master01 $ sudo bash -c 'cat > ${etcdSSLPath}/ca-config.json' << EOF
  {
      "signing": {
          "default": {
              "expiry": "43800h"
          },
          "profiles": {
              "server": {
                  "expiry": "43800h",
                  "usages": [
                      "signing",
                      "key encipherment",
                      "server auth",
                      "client auth"
                  ]
              },
              "client": {
                  "expiry": "43800h",
                  "usages": [
                      "signing",
                      "key encipherment",
                      "client auth"
                  ]
              },
              "peer": {
                  "expiry": "43800h",
                  "usages": [
                      "signing",
                      "key encipherment",
                      "server auth",
                      "client auth"
                  ]
              }
          }
      }
  }
  EOF
  ```

- CA
  ```bash
  master01 $ sudo bash -c 'cat > ${etcdSSLPath}/ca-csr.json' << EOF
  {
      "CN": "etcd",
      "key": {
          "algo": "rsa",
          "size": 2048
      }
  }
  EOF
  ```

- Client
  ```bash
  master01 $ sudo bash -c 'cat > ${etcdSSLPath}/client.json' << EOF
  {
      "CN": "client",
      "key": {
          "algo": "ecdsa",
          "size": 256
      }
  }
  EOF
  ```

- generate the ca and client cert files
  ```bash
  $ cd ${etcdSSLPath}

  # ca
  $ sudo /usr/local/bin/cfssl gencert \
         -initca ca-csr.json \
         | sudo /usr/local/bin/cfssljson -bare ca -

  # client
  $ sudo /usr/local/bin/cfssl gencert \
         -ca=ca.pem \
         -ca-key=ca-key.pem \
         -config=ca-config.json \
         -profile=client client.json \
         | sudo /usr/local/bin/cfssljson -bare client
  ```

<details>
<summary> click for more details </summary>
<pre><code>master01 $ sudo /usr/local/bin/cfssl gencert \
           -initca ca-csr.json \
           | sudo /usr/local/bin/cfssljson -bare ca -
2019/09/30 01:49:48 [INFO] generating a new CA key and certificate from CSR
2019/09/30 01:49:48 [INFO] generate received request
2019/09/30 01:49:48 [INFO] received CSR
2019/09/30 01:49:48 [INFO] generating key: rsa-2048
2019/09/30 01:49:49 [INFO] encoded CSR
2019/09/30 01:49:49 [INFO] signed certificate with serial number 173058676426587113118523721133402005590635049898
master01 $ ls
ca-config.json  ca.csr  ca-csr.json  ca-key.pem  ca.pem

master01 $ sudo /usr/local/bin/cfssl gencert \
       -ca=ca.pem \
       -ca-key=ca-key.pem \
       -config=ca-config.json \
       -profile=client client.json \
       | sudo /usr/local/bin/cfssljson -bare client
2019/09/30 01:51:23 [INFO] generate received request
2019/09/30 01:51:23 [INFO] received CSR
2019/09/30 01:51:23 [INFO] generating key: ecdsa-256
2019/09/30 01:51:23 [INFO] encoded CSR
2019/09/30 01:51:23 [INFO] signed certificate with serial number 294078367149194824536836589897529440907602394934
2019/09/30 01:51:23 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
master01 $ ls
ca-config.json  ca.csr  ca-csr.json  ca-key.pem  ca.pem  client.csr  client.json  client-key.pem  client.pem
</code></pre>
</details>
<br>


<div class="alert alert-success" role="alert">
<i class="fa fa-check-square-o"></i>
<b>Tip: </b> copy certs (ca & client) to the other two kubernetes masters
</div>
- copy certs to another two masters
  ```bash
  $ for i in {2..3}; do
    ssh master0${i} 'sudo mkdir -p ${etcdSSLPath}'
    rsync -avzrlpgoDP \
          --rsync-path='sudo rsync' \
          ${etcdSSLPath}/*.pem \
          master0${i}:${etcdSSLPath}/
    rsync -avzrlpgoDP \
          --rsync-path='sudo rsync' \
          ${etcdSSLPath}/ca-config.json \
          master0${i}:${etcdSSLPath}/
  done
  ```
<br>

## Configuration
<div class="alert alert-success" role="alert">
<i class="fa fa-check-square-o"></i>
<b>Tip: </b> etcd cluster need to be setup in all masters
</div>

- `etcd.service`
  ```bash
  $ sudo bash -c 'cat >/etc/systemd/system/etcd.service' << EOF
  [Install]
  WantedBy=multi-user.target

  [Unit]
  Description=Etcd Server
  Documentation=https://github.com/marslo/mytools
  Conflicts=etcd.service
  Conflicts=etcd2.service

  [Service]
  Type=notify
  WorkingDirectory=/var/lib/etcd/
  Restart=always
  RestartSec=5s
  EnvironmentFile=-/etc/etcd/etcd.conf
  ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /usr/local/bin/etcd"
  Restart=on-failure
  RestartSec=5
  LimitNOFILE=65536

  [Install]
  WantedBy=multi-user.target
  EOF
  ```

- `etcd.conf`
  ```bash
  $ sudo bash -c 'cat > /etc/etcd/etcd.conf' << EOF
  ETCD_NAME=${peerName}
  ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
  #ETCD_WAL_DIR=""
  #ETCD_SNAPSHOT_COUNT="10000"
  #ETCD_HEARTBEAT_INTERVAL="100"
  #ETCD_ELECTION_TIMEOUT="1000"
  ETCD_LISTEN_PEER_URLS="https://0.0.0.0:2380"
  ETCD_LISTEN_CLIENT_URLS="https://0.0.0.0:2379"
  #ETCD_MAX_SNAPSHOTS="5"
  #ETCD_MAX_WALS="5"
  #ETCD_CORS=""

  #[cluster]
  ETCD_INITIAL_ADVERTISE_PEER_URLS="https://${ipAddr}:2380"
  # if you use different ETCD_NAME (e.g. test), set ETCD_INITIAL_CLUSTER value for this name, i.e. "test=http://
  ..."
  ETCD_INITIAL_CLUSTER="${etcdInitialCluster}"
  ETCD_INITIAL_CLUSTER_STATE="new"
  ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
  ETCD_ADVERTISE_CLIENT_URLS="https://${ipAddr}:2379"
  #ETCD_DISCOVERY=""
  #ETCD_DISCOVERY_SRV=""
  #ETCD_DISCOVERY_FALLBACK="proxy"
  #ETCD_DISCOVERY_PROXY=""
  #ETCD_STRICT_RECONFIG_CHECK="false"
  #ETCD_AUTO_COMPACTION_RETENTION="0"

  #[proxy]
  #ETCD_PROXY="off"
  #ETCD_PROXY_FAILURE_WAIT="5000"
  #ETCD_PROXY_REFRESH_INTERVAL="30000"
  #ETCD_PROXY_DIAL_TIMEOUT="1000"
  #ETCD_PROXY_WRITE_TIMEOUT="5000"
  #ETCD_PROXY_READ_TIMEOUT="0"

  #[security]
  ETCD_CERT_FILE="${etcdSSLPath}/server.pem"
  ETCD_KEY_FILE="${etcdSSLPath}/server-key.pem"
  ETCD_CLIENT_CERT_AUTH="true"
  ETCD_TRUSTED_CA_FILE="${etcdSSLPath}/ca.pem"
  ETCD_AUTO_TLS="true"
  ETCD_PEER_CERT_FILE="${etcdSSLPath}/peer.pem"
  ETCD_PEER_KEY_FILE="${etcdSSLPath}/peer-key.pem"
  #ETCD_PEER_CLIENT_CERT_AUTH="false"
  ETCD_PEER_TRUSTED_CA_FILE="${etcdSSLPath}/ca.pem"
  ETCD_PEER_AUTO_TLS="true"

  #[logging]
  #ETCD_DEBUG="false"
  # examples for -log-package-levels etcdserver=WARNING,security=DEBUG
  #ETCD_LOG_PACKAGE_LEVELS=""
  #[profiling]
  #ETCD_ENABLE_PPROF="false"
  #ETCD_METRICS="basic"
  EOF
  ```

- enable service
  ```bash
  $ sudo systemctl daemon-reload
  $ sudo systemctl enable --now etcd
  $ sudo systemctl start etcd.service
  ```

- verify
  ```bash
  master0{1..3} $ sudo systemctl status etcd
  master01 $ sudo /usr/local/bin/etcdctl --ca-file /etc/etcd/ssl/ca.pem --cert-file /etc/etcd/ssl/client.pem --key-file /etc/etcd/ssl/client-key.pem --endpoints https://192.168.100.200:2379,https://192.168.100.201:2379,https://192.168.100.202:2379 cluster-health
  member ae76391b129**** is healthy: got healthy result from https://192.168.100.200:2379
  member cda996b3ea5a*** is healthy: got healthy result from https://192.168.100.201:2379
  member e295a3c1654e*** is healthy: got healthy result from https://192.168.100.202:2379
  cluster is healthy
  ```

<details>
<summary>tips</summary>
<pre><code>$ alias etcdctl="sudo /usr/local/bin/etcdctl --ca-file /etc/etcd/ssl/ca.pem --cert-file /etc/etcd/ssl/client.pem --key-file /etc/etcd/ssl/client-key.pem --endpoints https://192.168.100.200:2379,https://192.168.100.201:2379,https://192.168.100.202:2379"
$ etcdctl cluster-health
</code></pre>
</details>
