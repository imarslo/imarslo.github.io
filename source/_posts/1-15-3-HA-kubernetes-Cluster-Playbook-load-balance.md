---
layout: post
title:  1.15.3 HA kubernetes Cluster Playbook (load balance)
date:   2020-09-10 22:40:45 +0800
categories: ["kubernetes", "centos"]
tags: ["kuberentes", "centos"]
excerpt_separator: <!--more-->
---

## Objective
<img src="{{site.url}}/assets/images/external-etcd-topology-lb.png" style="width: 666px;" />

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

## keepalived

### configuration

<div class="alert alert-success" role="alert">
<i class="fa fa-check-square-o"></i>
<b>Tip: </b> keepalived configuration need to be setup in all kubernetes masters
</div>

- with haproxy
  ```bash
  $ sudo bash -c 'cat > /etc/keepalived/keepalived.conf' << EOF
  ! Configuration File for keepalived

  global_defs {
    router_id LVS_DEVEL
  }

  vrrp_script check_haproxy {
    script "killall -0 haproxy"
    interval 3
    weight -2
    fall 10
    rise 2
  }

  vrrp_instance VI_1 {
    state MASTER
    interface ${interface}
    virtual_router_id 51
    priority 50
    advert_int 1
    authentication {
      auth_type PASS
      auth_pass 35f18af7190d51c9f7f78f37300a0cbd
    }

    virtual_ipaddress {
      ${virtualIP}
    }

    track_script {
      check_haproxy
    }
  }
  EOF
  ```

- without haproxy
  - `keepalived.conf`
    ```bash
    $ sudo bash -c 'cat > /etc/keepalived/keepalived.conf' << EOF
    ! Configuration File for keepalived
    global_defs {
      router_id LVS_DEVEL
    }

    vrrp_script check_apiserver {
      script "/etc/keepalived/check_apiserver.sh"
      interval 3
      weight -2
      fall 10
      rise 2
    }

    vrrp_instance VI_1 {
      state MASTER
      interface ${interface}
      virtual_router_id 51
      priority 50
      authentication {
        auth_type PASS
        auth_pass 4be37dc3b4c90194d1600c483e10ad1d
      }
      virtual_ipaddress {
        ${virtualIP}
      }
      track_script {
        check_apiserver
      }
    }
    EOF
    ```

  - `check_apiserver.sh`
    ```bash
    $ sudo bash -c 'cat > /etc/keepalived/check_apiserver.sh' << EOF
    #!/bin/sh
    errorExit() {
      echo "*** \$*" 1>&2
      exit 1
    }

    curl --silent                           \
         --max-time 2                       \
         --insecure https://localhost:6443/ \
         -o /dev/null                       \
         || errorExit 'Error GET https://localhost:6443/'

    if ip addr | grep -q ${virtualIP}; then
      curl --silent                                  \
           --max-time 2                              \
           --insecure https://${virtualIP}:6443/     \
           -o /dev/null                              \
           || errorExit "Error GET https://${virtualIP}:6443/"
    fi
    EOF
    ```

### enable keepalived services in all masters
- start keepalived serice and verify
  ```bash
  $ sudo systemctl enable keepalived.service
  $ sudo systemctl start keepalived.service
  ```
<br>

<div class="alert alert-success" role="alert">
<i class="fa fa-check-square-o"></i>
<b>Tip: </b> One of the master will be setup to virutal dual networking card and show 2 ip addresses.
<br>The one without Broadcast is the virutal IP.
</div>

- verify
  ```bash
  $ sudo systemctl is-enabled keepalived.service
  enabled
  $ sudo systemctl is-active keepalived.service
  active

  $ ip -4 a s ${interface}
  2: eno1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
      link/ether 00:50:50:85:96:64 brd ff:ff:ff:ff:ff:ff
      inet 192.168.100.202/24 brd 192.168.100.255 scope global noprefixroute eno1
         valid_lft forever preferred_lft forever
      inet 192.168.100.250/32 scope global eno1
         valid_lft forever preferred_lft forever
  ```

<details>
<summary> click for more details </summary>
<pre><code>$ for i in {1..3}; do
  -> echo '---------'
  -> ssh -q devops@master0${i} "/usr/sbin/ip -4 a s $(netstat -nr | grep -E 'UG|UGSc' | grep -E '^0.0.0|default' | grep -E '[0-9.]{7,15}' | awk -F' ' '{print $NF}')"
  -> done
---------
2: eno1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    inet 192.168.100.200/24 brd 192.168.100.255 scope global noprefixroute eno1
       valid_lft forever preferred_lft forever
---------
2: eno1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    inet 192.168.100.201/24 brd 192.168.100.255 scope global noprefixroute eno1
       valid_lft forever preferred_lft forever
---------
2: eno1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    inet 192.168.100.202/24 brd 192.168.100.255 scope global noprefixroute eno1
       valid_lft forever preferred_lft forever
    inet 192.168.100.250/32 scope global eno1       <<<< virtual ip in master node 03
       valid_lft forever preferred_lft forever
</code></pre>
<br>

<pre><code>$ for i in {1..3}; do
  -> ssh -q devops@master0${i} "sudo systemctl status keepalived"
  -> echo ''
  -> done
● keepalived.service - LVS and VRRP High Availability Monitor
   Loaded: loaded (/etc/systemd/system/keepalived.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2019-09-03 01:13:17 PDT; 18min ago
  Process: 26437 ExecStart=/usr/local/sbin/keepalived $KEEPALIVED_OPTIONS (code=exited, status=0/SUCCESS)
 Main PID: 26438 (keepalived)
    Tasks: 2
   Memory: 652.0K
   CGroup: /system.slice/keepalived.service
           ├─26438 /usr/local/sbin/keepalived -D
           └─26439 /usr/local/sbin/keepalived -D
Sep 03 01:15:35 master01 Keepalived_vrrp[26439]: (VI_1) ip address associated with VRID 51 not present in MASTER advert : 10.69.78.50
Sep 03 01:15:36 master01 Keepalived_vrrp[26439]: (VI_1) ip address associated with VRID 51 not present in MASTER advert : 10.69.78.50
Sep 03 01:15:37 master01 Keepalived_vrrp[26439]: (VI_1) ip address associated with VRID 51 not present in MASTER advert : 10.69.78.50
Sep 03 01:15:38 master01 Keepalived_vrrp[26439]: (VI_1) ip address associated with VRID 51 not present in MASTER advert : 10.69.78.50
Sep 03 01:15:39 master01 Keepalived_vrrp[26439]: (VI_1) ip address associated with VRID 51 not present in MASTER advert : 10.69.78.50
Sep 03 01:15:40 master01 Keepalived_vrrp[26439]: (VI_1) ip address associated with VRID 51 not present in MASTER advert : 10.69.78.50
Sep 03 01:15:41 master01 Keepalived_vrrp[26439]: (VI_1) ip address associated with VRID 51 not present in MASTER advert : 10.69.78.50
Sep 03 01:15:42 master01 Keepalived_vrrp[26439]: (VI_1) ip address associated with VRID 51 not present in MASTER advert : 10.69.78.50
Sep 03 01:15:43 master01 Keepalived_vrrp[26439]: (VI_1) ip address associated with VRID 51 not present in MASTER advert : 10.69.78.50
Sep 03 01:15:43 master01 Keepalived_vrrp[26439]: (VI_1) ip address associated with VRID 51 not present in MASTER advert : 10.69.78.50

● keepalived.service - LVS and VRRP High Availability Monitor
   Loaded: loaded (/etc/systemd/system/keepalived.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2019-09-03 01:17:24 PDT; 14min ago
  Process: 32672 ExecStart=/usr/local/sbin/keepalived $KEEPALIVED_OPTIONS (code=exited, status=0/SUCCESS)
 Main PID: 32673 (keepalived)
    Tasks: 2
   Memory: 652.0K
   CGroup: /system.slice/keepalived.service
           ├─32673 /usr/local/sbin/keepalived -D
           └─32674 /usr/local/sbin/keepalived -D
Sep 03 01:17:24 master02 Keepalived_vrrp[32674]: (Line 19) Truncating auth_pass to 8 characters
Sep 03 01:17:24 master02 Keepalived_vrrp[32674]: WARNING - script '/etc/keepalived/check_apiserver.sh' is not executable for uid:gid 0:0 - disabling.
Sep 03 01:17:24 master02 Keepalived_vrrp[32674]: SECURITY VIOLATION - scripts are being executed but script_security not enabled.
Sep 03 01:17:24 master02 Keepalived_vrrp[32674]: Assigned address 192.168.100.201 for interface eno1
Sep 03 01:17:24 master02 Keepalived_vrrp[32674]: Assigned address fe80::250:56ff:fe88:fd2 for interface eno1
Sep 03 01:17:24 master02 Keepalived_vrrp[32674]: Registering gratuitous ARP shared channel
Sep 03 01:17:24 master02 Keepalived_vrrp[32674]: (VI_1) removing VIPs.
Sep 03 01:17:24 master02 Keepalived_vrrp[32674]: (VI_1) Entering BACKUP STATE (init)
Sep 03 01:17:24 master02 Keepalived_vrrp[32674]: VRRP sockpool: [ifindex(2), family(IPv4), proto(112), unicast(0), fd(11,12)]
Sep 03 01:17:24 master02 systemd[1]: Started LVS and VRRP High Availability Monitor.

● keepalived.service - LVS and VRRP High Availability Monitor
   Loaded: loaded (/etc/systemd/system/keepalived.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2019-09-03 01:17:35 PDT; 14min ago
  Process: 16830 ExecStart=/usr/local/sbin/keepalived $KEEPALIVED_OPTIONS (code=exited, status=0/SUCCESS)
 Main PID: 16831 (keepalived)
    Tasks: 2
   Memory: 648.0K
   CGroup: /system.slice/keepalived.service
           ├─16831 /usr/local/sbin/keepalived -D
           └─16832 /usr/local/sbin/keepalived -D
Sep 03 01:17:35 master03 Keepalived_vrrp[16832]: (Line 19) Truncating auth_pass to 8 characters
Sep 03 01:17:35 master03 Keepalived_vrrp[16832]: WARNING - script '/etc/keepalived/check_apiserver.sh' is not executable for uid:gid 0:0 - disabling.
Sep 03 01:17:35 master03 Keepalived_vrrp[16832]: SECURITY VIOLATION - scripts are being executed but script_security not enabled.
Sep 03 01:17:35 master03 Keepalived_vrrp[16832]: Assigned address 192.168.100.202 for interface eno1
Sep 03 01:17:35 master03 Keepalived_vrrp[16832]: Assigned address fe80::250:56ff:fe88:9624 for interface eno1
Sep 03 01:17:35 master03 Keepalived_vrrp[16832]: Registering gratuitous ARP shared channel
Sep 03 01:17:35 master03 Keepalived_vrrp[16832]: (VI_1) removing VIPs.
Sep 03 01:17:35 master03 Keepalived_vrrp[16832]: (VI_1) Entering BACKUP STATE (init)
Sep 03 01:17:35 master03 Keepalived_vrrp[16832]: VRRP sockpool: [ifindex(2), family(IPv4), proto(112), unicast(0), fd(11,12)]
Sep 03 01:17:35 master03 systemd[1]: Started LVS and VRRP High Availability Monitor.
</code></pre>
</details>
<br>


## haproxy
### configuration
- haproxy configure
  ```bash
  $ sudo bash -c 'cat /etc/haproxy/haproxy.cfg' << EOF
  #---------------------------------------------------------------------
  # Example configuration for a possible web application.  See the
  # full configuration options online.
  #
  #   http://haproxy.1wt.eu/download/2.0/doc/configuration.txt
  #
  #---------------------------------------------------------------------

  #---------------------------------------------------------------------
  # Global settings
  #---------------------------------------------------------------------
  global
      log         127.0.0.1 local2

      chroot      /var/lib/haproxy
      pidfile     /var/run/haproxy.pid
      maxconn     4000
      user        haproxy
      group       haproxy
      daemon

      # turn on stats unix socket
      stats socket /var/lib/haproxy/stats

  #---------------------------------------------------------------------
  # common defaults that all the 'listen' and 'backend' sections will
  # use if not designated in their block
  #---------------------------------------------------------------------
  defaults
      mode                    http
      log                     global
      option                  httplog
      option                  dontlognull
      option http-server-close
      option forwardfor       except 127.0.0.0/8
      option                  redispatch
      retries                 3
      timeout http-request    10s
      timeout queue           1m
      timeout connect         10s
      timeout client          1m
      timeout server          1m
      timeout http-keep-alive 10s
      timeout check           10s
      maxconn                 3000

  #---------------------------------------------------------------------
  # kubernetes apiserver frontend which proxys to the backends
  #---------------------------------------------------------------------
  frontend kubernetes-apiserver
      mode                 tcp
      bind                 *:16443
      option               tcplog
      default_backend      kubernetes-apiserver

  #---------------------------------------------------------------------
  # round robin balancing between the various backends
  #---------------------------------------------------------------------
  backend kubernetes-apiserver
      mode        tcp
      balance     roundrobin
      option      tcplog
      option      tcp-check
      server      ${master01Name} ${master01IP}:6443 check
      server      ${master02Name} ${master02IP}:6443 check
      server      ${master03Name} ${master03IP}:6443 check

  #---------------------------------------------------------------------
  # collection haproxy statistics message
  #---------------------------------------------------------------------
  listen stats
      bind                 :8000
      stats auth           admin:devops
      maxconn              50
      stats refresh        10s
      stats realm          HAProxy\ Statistics
      stats uri            /healthy
  EOF
  ```

- Service
  ```bash
  $ sudo bash -c 'cat > /lib/systemd/system/haproxy.service' << EOF
  [Unit]
  Description=HAProxy Load Balancer
  After=network.target syslog.service
  Wants=syslog.service

  [Service]
  Environment="CONFIG=/etc/haproxy/haproxy.cfg" "PIDFILE=/run/haproxy.pid"
  EnvironmentFile=-/etc/default/haproxy
  ExecStartPre=/usr/sbin/haproxy -f $CONFIG -c -q
  ExecStart=/usr/sbin/haproxy -W -f $CONFIG -p $PIDFILE $EXTRAOPTS
  ExecReload=/usr/sbin/haproxy -f $CONFIG -c -q $EXTRAOPTS $RELOADOPTS
  ExecReload=/bin/kill -USR2 $MAINPID
  KillMode=mixed
  Restart=always
  Type=forking

  [Install]
  WantedBy=multi-user.target
  EOF
  ```
### Start Service
```bash
$ sudo systemctl enabled haproxy.service
$ sudo systemctl start haproxy.service
```

### verify
```bash
$ sudo systemctl is-enabled haproxy.service
enabled
$ sudo systemctl is-active haproxy.service
active
```

<img src="{{site.url}}/assets/images/haproxy-1.png" style="width: 999px;" />
