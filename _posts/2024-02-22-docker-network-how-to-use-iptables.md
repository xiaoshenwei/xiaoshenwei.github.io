---
toc: true
title: "云上慎用docker -p 80:80"
categories:
  - devops
tags:
  - linux
  - docker
  - network
---

# 背景

  云服务器上(绑定了公网IP), 使用docker部署服务时, 使用了docker -p 80:80, 部署完成后, 接口日志发现一些奇奇怪怪的请求路径，突然意识到服务被恶意扫描了。我没有操作iptables公开这个端口啊。这是怎么回事呢？

# Iptables 入门

![https://raw.githubusercontent.com/xiaoshenwei/xiaoshenwei.github.io/master/assets/images/5cf36899c47eb56912.png](https://raw.githubusercontent.com/xiaoshenwei/xiaoshenwei.github.io/master/assets/images/5cf36899c47eb56912.png)

1. 数据包通过pre_routing 关卡时，进行一次路由选择，当目的地址为本机时，数据进入input, 非本地的目标地址进入forward 关卡(本机内核支持ip_forward)，所以目标地址转换DNAT 通常发生在这个 关卡。
2. 数据包经过路由后，目的是本地的数据, 会经过input 关卡。所以过滤数据包通常发生在这个关卡
3. 数据包经过路由后，目的非本地的数据, 会进过forward关卡，所以网络防火墙通常在此关卡配置
4. OUTPUT 由本地用户空间应用进程产生的数据包过此关卡，所以 OUTPUT 包过滤在此关卡进行
5. POST_ROUTING 刚刚通过 FORWARD 和 OUTPUT 关卡的数据包要通过一次路由选择由哪个接口送往网络中，经过路由之后的数据包要通过 POST_ROUTING 此关卡，源地址转换SNAT通常在此关卡进行

## 四表五链

iptables 默认有五条链（chain），分别对应上面提到的五个关卡：

- PRE_ROUTING
- INPUT
- FORWARD
- OUTPUT
- POST_ROUTING

为什么叫做“链”呢？ 

我们知道，iptables/netfilter 防火墙对经过的数据包进行“规则”匹配，然后执行相应的“处理”。当报文经过某一个关卡时，这个关卡上的“规则”不止一条，很多条规则会按照顺序逐条匹配，将在此关卡的所有规则组织称“链”就很适合，对于经过相应关卡的网络数据包按照顺序逐条匹配“规则”。

![https://raw.githubusercontent.com/xiaoshenwei/xiaoshenwei.github.io/master/assets/images/5cf4c0e47307857800.png](https://raw.githubusercontent.com/xiaoshenwei/xiaoshenwei.github.io/master/assets/images/5cf4c0e47307857800.png)

另外一个问题是，每一条“链”上的一串规则里面有些功能是相似的

比如，A 类规则都是对 IP 或者端口进行过滤，B 类规则都是修改报文，我们考虑能否将这些功能相似的规则放到一起，这样管理 iptables 规则会更方便。

iptables 把具有相同功能的规则集合叫做“表”，并且定一个四种表：

- `filter 表`：用来对数据包进行过滤，具体的规则要求决定如何处理一个数据包。

  对应的内核模块为：`iptable_filter`，其表内包括三个链：`input`、`forward`、`output`;

- `nat 表`：nat 全称：network address translation 网络地址转换，主要用来修改数据包的 IP 地址、端口号信息。

  对应的内核模块为：`iptable_nat`，其表内包括四个链：`prerouting`、`postrouting`、`input`、 `output`;

- `mangle 表`：主要用来修改数据包的服务类型，生存周期，为数据包设置标记，实现流量整形、策略路由等。

  对应的内核模块为：`iptable_mangle`，其表内包括五个链：`prerouting`、`postrouting`、`input`、`output`、`forward`;

- `raw 表`：主要用来决定是否对数据包进行状态跟踪。

  对应的内核模块为：`iptable_raw`，其表内包括两个链：`output`、`prerouting`;

**表的处理优先级：`raw`>`mangle`>`nat`>`filter`。**

|              | raw  | mangle | nat  | filter |
| ------------ | ---- | ------ | ---- | ------ |
| pre_routing  | ✅    | ✅      | ✅    |        |
| input        |      | ✅      | ✅    | ✅      |
| forward      |      | ✅      |      | ✅      |
| output       | ✅    | ✅      | ✅    | ✅      |
| post_routing |      | ✅      | ✅    |        |



```shell
iptables -t raw -nL
iptables -t mangle -nL
iptables -t nat -nL
iptables -t filter -nL
```

网络管理员还可以使用 iptables 创建自定义的“链”，将针对某个应用层序所设置的规则放到这个自定义“链”中，但是自定义的“链”不能直接使用，只能被某个默认的“链”当作处理动作 action 去调用。可以这样说，自定义链是“短”链，这些“短”链并不能直接使用，而是需要和 iptables上 的内置链一起配合，被内置链“引用”。

# 安装docker 后iptables是什么样子的？

```shell
root@d1:~# iptables -t raw -nL
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
```

```shell
root@d1:~# iptables -t mangle -nL
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination

Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
```

```shell
root@d1:~# iptables -t nat -nL
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination
DOCKER     all  --  0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
DOCKER     all  --  0.0.0.0/0           !127.0.0.0/8          ADDRTYPE match dst-type LOCAL

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
MASQUERADE  all  --  172.17.0.0/16        0.0.0.0/0

Chain DOCKER (2 references)
target     prot opt source               destination
RETURN     all  --  0.0.0.0/0            0.0.0.0/0

Chain L (0 references)
target     prot opt source               destination
```

```shell
root@d1:~# iptables -t filter -nL
Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination
DOCKER-USER  all  --  0.0.0.0/0            0.0.0.0/0
DOCKER-ISOLATION-STAGE-1  all  --  0.0.0.0/0            0.0.0.0/0
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
DOCKER     all  --  0.0.0.0/0            0.0.0.0/0
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination

Chain DOCKER (1 references)
target     prot opt source               destination

Chain DOCKER-ISOLATION-STAGE-1 (1 references)
target     prot opt source               destination
DOCKER-ISOLATION-STAGE-2  all  --  0.0.0.0/0            0.0.0.0/0
RETURN     all  --  0.0.0.0/0            0.0.0.0/0

Chain DOCKER-ISOLATION-STAGE-2 (1 references)
target     prot opt source               destination
DROP       all  --  0.0.0.0/0            0.0.0.0/0
RETURN     all  --  0.0.0.0/0            0.0.0.0/0

Chain DOCKER-USER (1 references)
target     prot opt source               destination
RETURN     all  --  0.0.0.0/0            0.0.0.0/0
```



# 启动一个docker 使用 bridge 网络

```shell
root@d2:~# docker run -d  harbor-sf.dxy.net/k8s/nginx:stable
```

```shell
root@d2:~# iptables -t nat -nL
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination
DOCKER     all  --  0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
DOCKER     all  --  0.0.0.0/0           !127.0.0.0/8          ADDRTYPE match dst-type LOCAL

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
MASQUERADE  all  --  172.17.0.0/16        0.0.0.0/0

Chain DOCKER (2 references)
target     prot opt source               destination
RETURN     all  --  0.0.0.0/0            0.0.0.0/0
```

```shell
root@d2:~# iptables -t filter -nL
Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination
DOCKER-USER  all  --  0.0.0.0/0            0.0.0.0/0
DOCKER-ISOLATION-STAGE-1  all  --  0.0.0.0/0            0.0.0.0/0
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
DOCKER     all  --  0.0.0.0/0            0.0.0.0/0
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination

Chain DOCKER (1 references)
target     prot opt source               destination

Chain DOCKER-ISOLATION-STAGE-1 (1 references)
target     prot opt source               destination
DOCKER-ISOLATION-STAGE-2  all  --  0.0.0.0/0            0.0.0.0/0
RETURN     all  --  0.0.0.0/0            0.0.0.0/0

Chain DOCKER-ISOLATION-STAGE-2 (1 references)
target     prot opt source               destination
DROP       all  --  0.0.0.0/0            0.0.0.0/0
RETURN     all  --  0.0.0.0/0            0.0.0.0/0

Chain DOCKER-USER (1 references)
target     prot opt source               destination
RETURN     all  --  0.0.0.0/0            0.0.0.0/0
```

# 启动一个docker -p 80:80

```shell
root@d3:~# docker run -d -p 80:80  harbor-sf.dxy.net/k8s/nginx:stable
```

```shell
root@d3:~# iptables -t nat -nL
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination
DOCKER     all  --  0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
DOCKER     all  --  0.0.0.0/0           !127.0.0.0/8          ADDRTYPE match dst-type LOCAL

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
MASQUERADE  all  --  172.17.0.0/16        0.0.0.0/0
MASQUERADE  tcp  --  172.17.0.2           172.17.0.2           tcp dpt:80

Chain DOCKER (2 references)
target     prot opt source               destination
RETURN     all  --  0.0.0.0/0            0.0.0.0/0
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:80 to:172.17.0.2:80
```

```shell
root@d3:~# iptables -t filter -nL
Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination
DOCKER-USER  all  --  0.0.0.0/0            0.0.0.0/0
DOCKER-ISOLATION-STAGE-1  all  --  0.0.0.0/0            0.0.0.0/0
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
DOCKER     all  --  0.0.0.0/0            0.0.0.0/0
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination

Chain DOCKER (1 references)
target     prot opt source               destination
ACCEPT     tcp  --  0.0.0.0/0            172.17.0.2           tcp dpt:80

Chain DOCKER-ISOLATION-STAGE-1 (1 references)
target     prot opt source               destination
DOCKER-ISOLATION-STAGE-2  all  --  0.0.0.0/0            0.0.0.0/0
RETURN     all  --  0.0.0.0/0            0.0.0.0/0

Chain DOCKER-ISOLATION-STAGE-2 (1 references)
target     prot opt source               destination
DROP       all  --  0.0.0.0/0            0.0.0.0/0
RETURN     all  --  0.0.0.0/0            0.0.0.0/0

Chain DOCKER-USER (1 references)
target     prot opt source               destination
RETURN     all  --  0.0.0.0/0            0.0.0.0/0
```

