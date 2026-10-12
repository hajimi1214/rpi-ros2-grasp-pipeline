# 09 · 装机前的推进路径（不依赖 Linux 也能做的事）

> 写于 2026-10-12，装机当天。前提：台式机 Ubuntu 还没装好，但不能干等。

---

## 一、今天就能跑 ROS2：GitHub Codespaces

**不用装系统，浏览器里就有 ROS2。** 免费个人账户每月 120 核小时（2 核机器 = 60 小时），够用。

**步骤**（约 10 分钟）：
1. 打开 https://github.com/waseemhnyc/ROS2-Github-Codespace
2. 点 Code → Codespaces 标签 → Create codespace on main
3. 等容器起来（首次约 5 分钟），终端里直接有 `ros2`
4. 跑一遍：
   ```
   ros2 run demo_nodes_cpp talker
   ros2 run demo_nodes_cpp listener      # 另一个终端
   ros2 node list / ros2 topic list / ros2 topic echo /chatter
   ```

**⚠️ 它是 Humble 不是 Jazzy。** 只用于练手和采集命令输出格式，
不要把 Codespaces 当主力开发环境——版本不一致会让你后面困惑。

**它的真正价值不是练手，是下面这件事 ↓**

---

## 二、最重要的一件事：采集命令的真实输出格式

`r2net` 的本质是「解析 ros2 命令输出 → 匹配规则 → 给修复建议」。
**没有真实输出样本，解析器就是凭空写的，一定错。**

所以在 Codespaces 里把下面每条命令的输出**原样保存**下来，存进 `fixtures/`：

| 命令 | 用途 |
|---|---|
| `ros2 doctor --report` | L0/L2 的主要输入，输出很长，重点看 network 段 |
| `ros2 topic info /chatter -v` | L3 匹配的输入，看 QoS 字段长什么样 |
| `ros2 multicast send` / `receive` | L2 发现层判定 |
| `ros2 node list` / `topic list` | 基准 |
| `ros2 topic hz / bw / delay` | L4 |
| `ip addr`（看 MULTICAST 标志） | L1 |
| `hostname -I` / `printenv ROS_*` | L0 |

**采集时注意两种状态都要存**：正常能通的一份 + 故意配错的一份（比如两端 domain id 不同）。
对比两份输出，诊断规则就出来了。

---

## 三、关键认知：v0.1 大部分不需要两台机器

原本以为要等 Pi5 和 PC 都装好才能开工，查下来不是：

| 层 | 需要两台机器吗 | 单机怎么造故障 |
|---|---|---|
| **L0 环境** | ❌ 不需要 | 两个终端设不同的 `ROS_DOMAIN_ID`，互相看不见 |
| **L1 链路** | ❌ 不需要 | 虚拟网卡 `veth pair` 造 MTU 不一致；多网卡绑错可用 Docker 网桥模拟 |
| **L2 发现** | 部分 | `ufw` 开起来拦 UDP；`ros2 multicast` 单机内也能测 |
| **L3 匹配** | ❌ 不需要 | 单机两个节点用不同 QoS 发布/订阅 |
| **真正需要两台的** | — | 只有「跨主机发现」这一条 |

**结论：台式机一装好 Ubuntu 就能开工 r2net 骨架，不用等 Pi5。**

---

## 四、Pi5 是第二台机器，且它不依赖台式机装机

Pi5 装 Ubuntu 24.04 只需要 microSD 卡 + 读卡器，**不动台式机的分区**，
所以它能和台式机装机**并行推进**，不互相阻塞。

待确认（问用户）：
- Pi5 现在跑的是什么系统？有没有 microSD 卡和读卡器？
- 如果 Pi5 已有 Raspberry Pi OS（Debian 12），ROS2 Jazzy 官方只支持 Ubuntu 24.04，需要重装

---

## 五、工具名 `r2net` 可用性（已查）

GitHub 上 `r2net` 是一个**用户名**（Rudy，无公开仓库，0 贡献），不是仓库名。
所以 `github.com/hajimi1214/r2net` **可以正常创建，不冲突**。

唯一副作用：别人搜 `r2net` 时可能先看到那个用户。可接受。

---

## 六、今天的推进顺序（装机之外）

| 优先级 | 做什么 | 耗时 |
|---|---|---|
| 1 | 开 Codespaces，跑通 talker/listener | 15 min |
| 2 | 采集第二张表的命令输出，存进 fixtures | 30 min |
| 3 | 确认 Pi5 现状（系统 + SD 卡） | 问一句 |
| 4 | 装机完成后：按 08 文档跑组网基线 | — |

第 2 项是今天唯一真正推进 `r2net` 的事——其余都是铺垫。
