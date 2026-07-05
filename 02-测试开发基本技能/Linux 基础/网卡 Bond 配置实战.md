---
tags:
  - 实战
  - Linux
  - 网络
---
# 网卡 Bond 配置实战

## 一、Bond 是什么

Bond 是将多个物理网卡绑定成一个逻辑网卡的技术，使用同一个 IP 工作。

主要作用：
- **提高网络传输速度**：多块网卡并行传输，提升带宽。
- **高可用冗余**：其中一块网卡故障时，业务仍可正常提供网络服务。

---

## 二、常用 Bond 模式

网卡绑定 mode 共有七种（0~6），常用的有三种：

| 模式 | 名称 | 特点 | 适用场景 |
|---|---|---|---|
| mode=0 | 平衡负载模式 | 两块网卡同时工作，自动备援 | 需交换机支持链路聚合 |
| mode=1 | 自动备援模式 | 仅主网卡工作，故障后切换 | 简单高可用 |
| mode=6 | 平衡负载模式 | 两块网卡同时工作，自动备援 | 无需交换机支持 |

---

## 三、Bond 准备工作

首先要确定服务器上的网卡规划用途，以及哪些网卡已插网线。一般是有两块网卡对应两根网线，分别连接不同的交换机。

使用 `ethtool` 查看网卡状态：

```sh
[root@master01 network-scripts]# ethtool p4p2
Settings for p4p2:
        Supported ports: [ FIBRE ]
        Supported link modes:   1000baseKX/Full
                                10000baseKR/Full
                                25000baseCR/Full
                                25000baseKR/Full
                                25000baseSR/Full
        Supported pause frame use: Symmetric
        Supports auto-negotiation: Yes
        Supported FEC modes: None BaseR
        Advertised link modes:  1000baseKX/Full
                                10000baseKR/Full
                                25000baseCR/Full
                                25000baseKR/Full
                                25000baseSR/Full
        Advertised pause frame use: Symmetric
        Advertised auto-negotiation: Yes
        Advertised FEC modes: None
        Speed: 10000Mb/s
        Duplex: Full
        Port: FIBRE
        PHYAD: 0
        Transceiver: internal
        Auto-negotiation: on
        Supports Wake-on: d
        Wake-on: d
        Current message level: 0x00000004 (4)
                               link
        Link detected: yes
```

> `Link detected: yes` 表示有网线插入；
> 如果 `Link detected: no`，尝试用 `ifup ethxxx`，如果依然为 no，才能说明此网卡确实没有网线插入。

---

## 四、网卡 Bond 配置

### 4.1 方法一：命令行配置

```sh
# 创建一个名为 storagepub 的网卡绑定接口，类型为 bond，模式为 802.3ad
ip link add storagepub type bond mode 802.3ad xmit_hash_policy layer3+4

# 关闭网卡
ip link set ens4np1 down

# 将网卡 ens4np1 加入到名为 storagepub 的绑定接口，成为其从属（slave）设备
ip link set ens4np1 master storagepub

# 启用网卡
ip link set ens4np1 up

# 启用绑定接口 storagepub，使其可用
ip link set storagepub up

# 查看绑定接口 storagepub 的详细信息，包括模式、成员网卡、负载均衡策略等
cat /proc/net/bonding/storagepub
```

### 4.2 方法二：修改 Bond 网卡配置文件

**主设备配置 `ifcfg-bond-storagepub`：**

```sh
[root@master01 network-scripts]# cat ifcfg-bond-storagepub
DEVICE=storagepub
BONDING_OPTS="mode=4 miimon=100 xmit_hash_policy=1"
TYPE=Bond
BONDING_MASTER=yes      # 表示该设备是绑定主设备（master）
BOOTPROTO=static        # 使用静态 IP 配置（不依赖 DHCP）
PEERDNS=no
IPV4_FAILURE_FATAL=no
IPV6INIT=no             # 不启用 IPv6 配置
NAME=bond-storagepub
ONBOOT=yes
IPADDR=172.22.88.177
NETMASK=255.255.255.0
```

**从设备配置 `ifcfg-p4p2`：**

```sh
[root@master01 network-scripts]# cat ifcfg-p4p2
TYPE=Ethernet
BOOTPROTO=static
DEVICE=p4p2
ONBOOT=yes
MASTER=storagepub       # 指定绑定接口的主设备为 storagepub
SLAVE=yes               # 指定该网卡是绑定接口的从设备（slave）
```

---

## 五、重启网络并验证

```sh
[root@master01 ~]# systemctl restart network

[root@master01 ~]# ip a | grep storagepub
11: p4p2: <BROADCAST,MULTICAST,SLAVE,UP,LOWER_UP> mtu 1500 qdisc mq master storagepub state UP group default qlen 1000
19: storagepub: <BROADCAST,MULTICAST,MASTER,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    inet 172.22.88.177/24 brd 172.22.88.255 scope global noprefixroute storagepub
```

验证要点：
- 绑定接口已启用并获取到 IP。
- 从属网卡状态为 `SLAVE,UP`，并指向正确的 master。

---

## 六、测试开发视角

- **高可用测试**：拔掉其中一根网线，验证业务是否中断、流量是否自动切换。
- **带宽测试**：使用 `iperf` 或 `netperf` 对比单网卡与 Bond 后的吞吐量。
- **模式匹配**：mode=0/mode=6 需要在交换机侧做对应配置，mode 不匹配可能导致网络环路或丢包。
- **故障注入**：在混沌工程或容灾测试中，Bond 是常见的网络冗余手段。
