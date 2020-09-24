---
layout: post
title:  1.15.3 HA kubernetes Cluster Playbook (basic environment)
date:   2020-08-25 16:47:24 +0800
categories: ["kubernetes", "centos"]
tags: ["kuberentes", "centos"]
excerpt_separator: <!--more-->
---

## Objective

<img src="{{site.url}}/assets/images/external-etcd-topology.png" style="width: 666px;" />

<!--more-->

> reference:
> - [Kubernetes HA cluster with external etcd](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/#external-etcd-topology)
> - [Generate self-signed certificates](https://coreos.com/os/docs/latest/generate-self-signed-certificates.html)


### Server Matrix

#### Environment List

| Hostname   | IP Address      | etcd              | cfssl & cfssljson | keepalived | haproxy  |
|------------|-----------------|-------------------|-------------------|------------|----------|
| master01   | 192.168.100.200 | master01 (etcd-0) | &#x2714;          | &#x2714;   | &#x2714; |
| master02   | 192.168.100.201 | master02 (etcd-1) | &#x2714;          | &#x2714;   | &#x2714; |
| master03   | 192.168.100.202 | master03 (etcd-2) | &#x2714;          | &#x2714;   | &#x2714; |
| node01     | 192.168.100.203 | &#x2717;          | &#x2717;          | &#x2717;   | &#x2717; |
| node02     | 192.168.100.204 | &#x2717;          | &#x2717;          | &#x2717;   | &#x2717; |
|            |                 |                   |                   |            |          |
| virtual IP | 192.168.100.250 |                   |                   |            |          |

#### `/etc/hosts`

Add for all servers

```bash
192.168.100.200     master01
192.168.100.201     master02
192.168.100.202     master03
192.168.100.203     worker01
192.168.100.204     worker02

192.168.100.250     jenkins.marslo.com
192.168.100.250     prometheus.marslo.com
192.168.100.250     grafana.marslo.com
192.168.100.250     dashboard.marslo.com
```

#### variables

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

## Tools Setup
### cfssl & cfssljson
<div class="alert alert-success" role="alert">
<i class="fa fa-check-square-o"></i>
<b>Tip: </b> cfssl and cfssljson need to be setup in all masters!
</div>

```bash
$ sudo bash -c "curl -o /usr/local/bin/cfssl ${cfsslDownloadUrl}/cfssl_linux-amd64"
$ sudo bash -c "curl -o /usr/local/bin/cfssljson ${cfsslDownloadUrl}/cfssljson_linux-amd64"
$ sudo chmod +x /usr/local/bin/cfssl*
```

### etcd
<div class="alert alert-success" role="alert">
<i class="fa fa-check-square-o"></i>
<b>Tip: </b> etcd need to be setup in all masters!
</div>

```bash
$ curl -sSL ${etcdDownloadUrl}/${etcdVer}/etcd-${etcdVer}-linux-amd64.tar.gz \
    | sudo tar -xzv --strip-components=1 -C /usr/local/bin/
```

<details>
<summary> click for more details </summary>

<pre><code>$ for i in {1..3}; do
  ssh devops@master0${i} curl -sSL ${etcd_download_url}/${etcd_version}/etcd-${etcd_version}-linux-amd64.tar.gz | sudo tar -xzv --strip-components=1 -C /usr/local/bin/
  ssh devops@master0${i} 'etcd --version'
done

Result:
etcd Version: 3.3.15
Git SHA: 94745a4ee
Go Version: go1.12.9
Go OS/Arch: linux/amd64
</code></pre>
</details>
<br>

### keepalived

- Installation
    ```bash
    $ mkdir -p ~/temp
    $ sudo mkdir -p /etc/keepalived/

    $ curl -fsSL ${keepaliveDownloadUrl}/keepalived-${keepaliveVer}.tar.gz \
       | tar xzf - -C ~/temp

    $ pushd .
    $ cd ~/temp/keepalived-${keepaliveVer}
    $ ./configure && make
    $ sudo make install
    $ sudo cp keepalived/keepalived.service /etc/systemd/system/
    $ popd
    $ rm -rf ~/temp
    ```
### Haproxy 2.0.6
- Install haproxy from source code
    ```bash
   $ curl -fsSL http://www.haproxy.org/download/$(echo ${haproxyVer%\.*})/src/haproxy-${haproxyVer}.tar.gz \
           | tar xzf - -C ~

    $ pushd .
    $ cd ~/haproxy-${haproxyVer}
    $ make TARGET=linux-glibc \
           USE_LINUX_TPROXY=1 \
           USE_ZLIB=1 \
           USE_REGPARM=1 \
           USE_PCRE=1 \
           USE_PCRE_JIT=1 \
           USE_OPENSSL=1 \
           SSL_INC=/usr/include \
           SSL_LIB=/usr/lib \
           ADDLIB=-ldl \
           USE_SYSTEMD=1
    $ sudo make install
    $ sudo cp haproxy /usr/sbin/
    $ sudo cp examples/haproxy.init /etc/init.d/haproxy && sudo chmod +x $_
    $ popd
    $ rm -rf ~/haproxy-${haproxyVer}
    ```
- Result
<img src="{{site.url}}/assets/images/haproxy-1.png" style="width: 999px;" />

### helm
- Installation
    ```bash
    $ curl -fsSL \
        https://get.helm.sh/helm-v2.14.3-linux-amd64.tar.gz \
        | sudo tar -xzv --strip-components=1 -C /usr/local/bin/

    $ while read -r _i; do
        sudo chmod +x "/usr/local/bin/${_i}"
    done < <(echo helm tiller)
    ```
- Configration
    ```bash
    $ helm init
    $ helm init --client-only

    $ kubectl create serviceaccount -n kube-system tiller
    $ kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
    $ kubectl patch deploy -n kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'

    $ helm repo add jetstack https://charts.jetstack.io
    ```

### docker
- Presetup
    ```bash
    # clean environment
    $ sudo yum remove -y docker                  \
                         docker-client           \
                         docker-client-latest    \
                         docker-common           \
                         docker-latest           \
                         docker-latest-logrotate \
                         docker-logrotate        \
                         docker-selinux          \
                         docker-engine-selinux   \
                         docker-engine

    # nice to have
    $ sudo yum -y groupinstall 'Development Tools'

    $ sudo yum install -y yum-utils                   \
                        device-mapper-persistent-data \
                        lvm2                          \
                        bash-completion*

    $ sudo yum-config-manager \
         --add-repo           \
         https://download.docker.com/linux/centos/docker-ce.repo
    $ sudo yum-config-manager --disable docker-ce-edge
    $ sudo yum-config-manager --disable docker-ce-test
    $ sudo yum makecache
    ```

- Installaiton
    ```bash
    $ dockerVer=$(sudo yum list docker-ce --showduplicates \
                    | sort -r \
                    | grep 18\.09 \
                    | awk -F' ' '{print $2}' \
                    | awk -F':' '{print $NF}' \
                 )
    $ sudo yum install -y                      \
             docker-ce-${dockerVer}.x86_64     \
             docker-ce-cli-${dockerVer}.x86_64 \
             containerd.io
    ```
- Configuraiton
    ```bash
    $ sudo systemctl enable --now docker
    $ sudo systemctl status docker
    $ sudo chown -a -G docker $(whomai)
    ```
