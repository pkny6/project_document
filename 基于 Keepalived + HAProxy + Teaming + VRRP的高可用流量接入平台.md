# 基于 Keepalived + HAProxy + Teaming + VRRP的高可用流量接入平台

---

## 一、系统架构

```mermaid
graph TB
    subgraph Internet
        Client[客户端]
    end

    subgraph "负载均衡层 (VIP: 192.168.227.100)"
        LB1[HAProxy+Keepalived+Teaming<br/>主机: lb1<br/>IP: 192.168.227.11]
        LB2[HAProxy+Keepalived+Teaming<br/>主机: lb2<br/>IP: 192.168.227.12]
        VIP((虚拟IP<br/>192.168.227.100))
    end

    subgraph "Web应用层"
        WEB1[Apache/Nginx<br/>主机: web1<br/>IP: 192.168.227.13]
        WEB2[Apache/Nginx<br/>主机: web2<br/>IP: 192.168.227.14]
    end

    subgraph "物理链路聚合 (每台LB)"
        NIC_LB1_1[ens33] --- Team_LB1[team0<br/>]
        NIC_LB1_2[ens36] --- Team_LB1
        NIC_LB2_1[ens33] --- Team_LB2[team0]
        NIC_LB2_2[ens36] --- Team_LB2
    end

    Client --> VIP
    VIP -.-> LB1
    VIP -.-> LB2
    LB1 --> WEB1
    LB1 --> WEB2
    LB2 --> WEB1
    LB2 --> WEB2
```

## 二、节点规划

| 主机名      | IP             | 用途      | 备注    |
| -------- | -------------- | ------- | ----- |
| lb1.com  | 192.168.227.11 | 主负载均衡   | 双网卡聚合 |
| lb2.com  | 192.168.227.12 | 备负载均衡   | 双网卡聚合 |
| web1.com | 192.168.227.13 | web服务器1 | 单网卡   |
| web2.com | 192.168.227.14 | web服务器2 | 单网卡   |

## 三、teaming实现网卡聚合

> 配置网卡聚合并启动

```bash
nmcli con add ifname team0 con-name team0 type team \
> config '{"runner":{"name":"roundrobin"}'
nmcli con modify team0 ipv4.method manual ipv4.addresses 192.168.227.11 \
> ipv4.gateway 192.168.227.2 ipv4.dns 114.114.114.114
nmcli con add con-name team0-port1 ifname ens33 type team-slave \
> master team0
nmcli con add con-name team0-port2 ifname ens36 type team-slave \
> master team0
nmcli con up team0-port1
nmcli con up team0-port2
nmcli con up team0
teamdctl team0 state
```

## 四、Haproxy实现负载均衡

> haproxy.conf

```bash
global
    daemon
    maxconn 50000

defaults
    mode http
    timeout connect 5s
    timeout client 50s
    timeout server 50s

frontend http-in
    bind *:80
    default_backend webservers

backend webservers
    balance roundrobin
    option httpchk GET /
    server web1 192.168.227.13:80 check
    server web2 192.168.227.14:80 check
```

## 五、Keepalived实现高可用

> Master的keepalived.conf

```bash
global_defs {
    router_id LVS_MASTER
}

vrrp_script chk_haproxy {
    script "/etc/keepalived/check_haproxy.sh"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state MASTER
    interface team0          
    virtual_router_id 51
    priority 150
    advert_int 1
    virtual_ipaddress {
        192.168.227.100/24 dev team0
    }
    track_script {
        chk_haproxy
    }
}
```

> Backup的keepalived.conf

```bash
global_defs {
    router_id LVS_BACKUP
}

vrrp_script chk_haproxy {
    script "/etc/keepalived/check_haproxy.sh"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface team0           
    virtual_router_id 51
    priority 100
    advert_int 1
    virtual_ipaddress {
        192.168.227.100/24 dev team0
    }
    track_script {
        chk_haproxy
    }
}
```

> 编辑启动脚本

```bash
sed -i '/KillMode/d' /usr/lib/systemd/system/keepalived.service
systemctl daemon-reload
```

> keepalived测试haproxy是否宕机的脚本

```bash
#!/bin/bash
if ! pidof haproxy > /dev/null; then
    systemctl start haproxy
    sleep 1
    if ! pidof haproxy > /dev/null; then
        systemctl stop keepalived
    fi
fi
```
