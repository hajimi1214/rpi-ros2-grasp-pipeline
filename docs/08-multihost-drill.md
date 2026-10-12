# 08 · 多机组网实操 + 故障注入清单

> 用途有两条，第二条更重要：
> 1. Ubuntu 装好后第一天照这个跑，不用再到处查
> 2. **这张表就是 `r2net` 的测试用例集**——每注入一个故障，就多一条诊断规则

来源：CSDN 多篇排查帖 + 工业实践帖 + 鱼香问答 + RoboCloud 面题库 q786，多来源交叉确认过。

---

## 零、开场：两台机器的信息先填齐

开两个终端之前，先把这张表填了。90% 的"连不上"都是这里某一行不一致。

| 项 | PC（台式机） | Pi 5 | 检查命令 |
|---|---|---|---|
| 主机名 | | | `hostname` |
| IP | | | `hostname -I` |
| ROS 版本 | Jazzy | Jazzy | `ros2 --version`（或 `printenv ROS_DISTRO`） |
| ROS_DOMAIN_ID | | | `echo $ROS_DOMAIN_ID`（空 = 0） |
| RMW 实现 | | | `echo $RMW_IMPLEMENTATION`（空 = Fast DDS） |
| discovery range | | | `echo $ROS_AUTOMATIC_DISCOVERY_RANGE`（Jazzy+） |
| LOCALHOST_ONLY | | | `echo $ROS_LOCALHOST_ONLY` |
| 防火墙 | | | `sudo ufw status` |

**两台机器的发行版和 RMW 必须一致。** 混用 Fast DDS / Cyclone DDS 会出现"发现成功但传不了数据"。

---

## 一、正常流程：先证明"能通"

别一上来就查故障。**先建立一条已知能通的基线**，不然出问题没法判断是网络还是代码。

| 步 | PC 上 | Pi 上 | 预期 |
|---|---|---|---|
| 1 | `ping <pi-ip>` | `ping <pc-ip>` | 双向通。不通就别往下走了 |
| 2 | `ros2 multicast receive` | `ros2 multicast send` | PC 收到 Hello World |
| 3 | `ros2 run demo_nodes_cpp talker` | — | — |
| 4 | — | `ros2 run demo_nodes_cpp listener` | Pi 收到 Hello World |
| 5 | — | `ros2 node list` | **能看到 `/talker`**，否则发现层就断了 |
| 6 | `ros2 topic list` | `ros2 topic list` | 两边都看到 `/chatter` |

第 5 步是关键分水岭：`node list` 看到对方 = 发现层通；`node list` 看不到 = L0/L1/L2 有问题，跟代码无关。

---

## 二、故障注入清单（10 种）

**每种都先手动注入、记下现象，再让工具去查。** 这是做出真工具的唯一路子——没踩过的坑写不出诊断规则。

### L0 · 本机环境

| # | 故障 | 怎么注入 | 现象 | 修复 |
|---|---|---|---|---|
| 1 | domain id 不一致 | 一端 `export ROS_DOMAIN_ID=5` | `node list` 只看到本机节点，ping 正常 | 两端统一，写进 `~/.bashrc` |
| 2 | 仅限本机 | `export ROS_LOCALHOST_ONLY=1`（Humble）<br>`export ROS_AUTOMATIC_DISCOVERY_RANGE=LOCALHOST`（Jazzy+） | 同上，最容易在 `.bashrc` 里遗留 | `unset` 掉，检查 `.bashrc` |
| 3 | RMW 混用 | 一端 `export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp` | 可能发现成功但无数据，最诡异 | 两端统一 |

### L1 · 链路

| # | 故障 | 怎么注入 | 现象 | 修复 |
|---|---|---|---|---|
| 4 | 多网卡绑错接口 | 机器同时有 Wi-Fi + 以太网 + Docker 网桥 | DDS 广播了错的地址，对方收不到 | 用 `FASTRTPS_DEFAULT_PROFILES_FILE` 钉接口白名单 |
| 5 | Wi-Fi AP 隔离 | 连公共/校园 Wi-Fi | ping 都不通 | 换独立路由器或热点 |

### L2 · 发现

| # | 故障 | 怎么注入 | 现象 | 修复 |
|---|---|---|---|---|
| 6 | 防火墙拦 UDP | `sudo ufw enable` | ping 通，multicast 收不到 | `sudo ufw allow 7400:7500/udp` |
| 7 | 多播被网络丢弃 | 企业 Wi-Fi / 虚拟机 NAT 模式 | ping 通，multicast 不通 | 虚拟机改桥接；或用静态 peer 走单播 |
| 8 | 主机名解析失败 | `/etc/hosts` 没有对端映射 | `node list` 报 node not found，`ping <hostname>` 失败 | `sudo vim /etc/hosts` 加 `IP hostname` |
| 9 | MTU 不匹配 | 一端 `ip link set dev eth0 mtu 9000` | 大消息分片被交换机丢，小消息正常 | 统一 `mtu 1500` |

### L3 · 匹配（v0.1 不做，先记下来）

| # | 故障 | 怎么注入 | 现象 | 修复 |
|---|---|---|---|---|
| 10 | QoS 不兼容 | 发布端 `best_effort`，订阅端默认 `reliable` | **`topic list` 看得到，`echo` 什么都没有，无报错** | `ros2 topic echo /x --qos-reliability best_effort` 验证后改代码 |

第 10 条是**最经典的静默失败**。DDS 在发现阶段只认 topic 名，建数据链路时才做 QoS 协商，协商失败直接丢弃且不抛异常。

---

## 三、踩坑时要记录什么

每撞一次，往 `docs/02-weekly-log.md` 写一条，固定三行：

```
现象：<原样粘贴终端输出，不要转述>
根因：<哪一层、哪个变量>
解法：<能直接复制的那条命令>
```

**不要写"调了一下就好了"**——这种记录三个月后等于没写，面试时也讲不出来。

---

## 四、这些记录怎么变成工具

注入表里的 10 种故障，每种对应一条诊断规则，形如：

```
IF ping 通 AND 多播不通 AND ufw 状态是 active
THEN 判定 L2 防火墙，输出：sudo ufw allow 7400:7500/udp
```

**规则来自真实终端输出，不是来自文档。** 所以必须先手动踩一遍，看到真实报错长什么样——
官方文档写的是设计意图，终端里打出来的才是实际结果。

---

## 五、明天装完 Ubuntu 的第一件事

不是装 ROS2，是先做这张表的**第零步**：把两台机器的信息表填齐。
填不齐说明网络还没配好，后面全是白搭。
