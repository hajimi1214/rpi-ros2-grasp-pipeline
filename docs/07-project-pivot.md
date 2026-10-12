# 07 · 主项目转向：ROS2 连接诊断 CLI

> 决策日期：2026-10-12
> 状态：**方向已定，命名待定，未开工**（等 Ubuntu 装好）
> 这份文档的目的是——以后任何一次新对话读到它，都不用重新论证一遍。

---

## 一、为什么换

原来的 `rpi-ros2-grasp-pipeline` 属于「证明我会」的作品，不是「别人要用」的工具。
机械臂抓取仿真是教程里最常见的题材，同质化严重。

**自测问题：你 star 过别人的抓取 demo 吗？** 没有。这就是答案。

高星规律（多来源一致）：解决痛苦 > 堆功能；越小越具体越容易传播；5 分钟上手是硬门槛。

---

## 二、候选 B 已排除（边缘推理性能基准）

查过了，**红海，且我的条件劣势明显**：

| 已有项目 | 说明 |
|---|---|
| aallan/benchmarking-ml-on-the-edge | 知名博主，Pi 3/4/5 + Coral + NCS 全测过，有文章引流 |
| hig-dev/edgebench | 硕士论文项目，Pi5 / BeagleY-AI / Hailo-8 / ESP32-S3，MQTT 管理多设备 |
| joeltadeu/edge-ai-benchmark | TFLite/ORT/PyTorch Mobile 三方对比，方法学严格 |
| littlebee/scatbot-edge-ai-shootout | 目标检测专项，含 CPU 抢占场景 |

**排除的硬理由**：这类项目的可信度来自"多设备横向对比"，我只有一台 Pi 5。
单机数据没说服力，这是资源决定的，不是努力能补的。

---

## 三、方向 A 的竞品再调研（比上一轮更精确）

| 工具 | 形态 | 管什么 | 缺口 |
|---|---|---|---|
| **connection_inspector** | C++ GUI（Dear ImGui） | QoS 不兼容 / type mismatch / dead link | **已进 ROS Index 官方索引，支持 Jazzy。QoS 这块被占住了。弱点是依赖 X11，SSH 用不了** |
| **ros2diag**（pitosalas） | **CLI** | TF 帧 / 参数搜索 / 节点间延迟（`r2latency`） | 是 CLI，但**不管网络发现、不管 QoS 匹配** |
| **ros2doctor**（官方） | CLI | 平台/网络/环境全面检查，`doctor hello` 测跨主机 | **给报告，不给修复动作**，要自己解读 |
| rclshark / ros2graph_explorer / domain_id_scanner | 混合 | IP 发现 / Web 图 / 扫 domain | 各有侧重，都不解决"为什么连不上" |

### 交叉缺口（我要占的位置）

```
                    管网络发现层    管 QoS/类型匹配    无头 CLI 可用    给修复命令
connection_inspector      ✗              ✓               ✗ (GUI)        部分
ros2diag                  ✗              ✗               ✓              ✗
ros2doctor                ✓              ✗               ✓              ✗
─────────────────────────────────────────────────────────────────────────────
本工具                    ✓              ✓               ✓              ✓
```

**一句话定位**：SSH 里跑一条命令，告诉你哪一层断了、为什么、给你一条能直接复制的修复命令。

---

## 四、痛点真实性验证（多来源交叉，已查证）

不是我臆想的痛点。以下来源在讲**同一套排查流程**，且每个来源都列出了「现象 → 诊断命令 → 修复命令」：

- CSDN 多篇：「话题存在但收不到数据的五个原因」速查表
- 工业实践帖：MTU 不匹配导致组播被交换机分片丢弃（最隐蔽的坑）
- 鱼香 ROS 相关问答：DDS 发现机制阻断占多机通信失效的 90%+
- RoboCloud 面试题库 q786：**「两台 ROS2 机器在同一网络但互相看不见，你怎么 debug？」**

最后一条最关键——**这套排查流程本身就是面试题**。做出工具 = 把面试答案变成了作品，讲的时候是双重证明。

### 经典症状（README 开篇就要写这个）

> `ros2 topic list` 两边都能看到 topic，但 `ros2 topic echo` 什么都没有。
> 没有报错，没有异常，就是收不到。

原因：DDS 在发现阶段只认 topic 名，真正建数据链路时才做 QoS 协商，协商失败**静默丢弃**。

---

## 五、产品规格：分层诊断

从下往上查，任一层 FAIL 就停在那里报修复命令，不往下查。

| 层 | 查什么 | 典型修复 |
|---|---|---|
| **L0 本机环境** | `ROS_DOMAIN_ID` / `ROS_LOCALHOST_ONLY` / `ROS_AUTOMATIC_DISCOVERY_RANGE`（Jazzy+）/ `RMW_IMPLEMENTATION` / distro 是否两端一致 | `export ROS_DOMAIN_ID=0` 写进 `.bashrc` |
| **L1 链路** | 同网段判断、ping 双向、网卡是否有 `MULTICAST` 标志、多网卡时 DDS 是否绑错接口、Wi-Fi AP 隔离 | 用 `FASTRTPS_DEFAULT_PROFILES_FILE` 钉死接口白名单 |
| **L2 发现** | `ros2 multicast send/receive`、防火墙（ufw/iptables）是否拦 UDP 7400–7500、`/etc/hosts` 主机名解析、`ip link` MTU 是否一致 | `sudo ufw allow 7400:7500/udp`；`ip link set dev eth0 mtu 1500` |
| **L3 匹配** | topic 存在但无数据 → `ros2 topic info -v` 比对两端 QoS/类型 | `ros2 topic echo /x --qos-reliability best_effort` 验证 |
| **L4 性能** | `topic hz` / `bw` / `delay` 是否达标，丢包定位 | 调 depth / 降频 / 拆线程 |

**每层输出三行**：`PASS/WARN/FAIL` + 一句人话解释 + 一条可复制的修复命令。

### 杀手功能（v2，MVP 之后）

`r2net peer <user@host>`：通过 SSH 在两端同时跑诊断，回传并**自动对比差异**。
现有的排查方式要开两个终端来回敲命令切机器，这个功能是消灭那件事的。已有工具都没有。

---

## 六、MVP 范围（严格限定，先做小）

**v0.1 只做一件事**：一条命令 `r2net check`，输出 L0 + L1 + L2 的分层报告 + 修复命令。

不做（v0.1）：不做 QoS GUI、不做 Web 界面、不做自动修复（只**给**命令不代执行）、不做 `--peer` SSH 模式。

### 两条关键设计约束

1. **不做成 colcon 包，做纯 Python 单文件。**
   做成 ROS2 包就要 `colcon build` + `source install/setup.bash`，直接毁掉「5 分钟上手」。
   只用 `ros2` CLI 的输出做解析 + 标准库，装法是一行 curl 或 `pip install`。
2. **不自动改任何东西。** 只输出修复命令让用户自己复制执行。
   自动改防火墙/网络配置的工具没人敢在机器人上跑，而且出错责任说不清。

---

## 七、命名（待定，我倾向第 2 个）

| 候选 | 理由 |
|---|---|
| `r2net` | **推荐**。跟已有 `ros2diag` 的命令前缀 `r2*` 一致（`r2tflist`/`r2latency`），用户肌肉记忆通用；短 |
| `ros2-netdoc` | 语义清楚，但长，且跟 `ros2doctor` 容易混 |
| `ros2link` | 好读，但语义偏"链接"容易误解 |

命令形如：`r2net check` / `r2net why /topic` / `r2net peer pi@192.168.1.x`

---

## 八、验收标准（做完没做完，看这个）

- [ ] 在一个**故意配错**的环境里（比如两端 domain id 不同），工具能指出是 L0 的哪一项，并给出正确命令
- [ ] 在真实的 Pi5 + PC 环境里，全绿时输出不超过一屏
- [ ] 陌生人照 README 装好到跑出第一条报告 **≤ 5 分钟**
- [ ] README 第一屏是「症状 → 命令 → 输出截图」对比，不是"This is a tool for..."
- [ ] 有 ≥1 个**非本人**的 star 或 issue（真实需求的唯一硬证据）

---

## 九、两个仓库的关系（重要，别理解成放弃原仓库）

| 仓库 | 定位 | 投入 |
|---|---|---|
| **新：`r2net`** | 主项目，冲 star + 简历第一项目 | 主力 |
| **原：rpi-ros2-grasp-pipeline** | 降级为**素材来源 + 简历第二项目** | 只做 W1（组网） |

降级不是放弃，理由有两条：

1. **做工具需要真实痛点，而痛点来自真去组网。** 原仓库的 W1（PC + Pi5 双向组网）就是这个工具的素材来源——不亲自踩一遍，做出来的东西是臆想的。
2. 原仓库在简历上仍有独立价值：它是**操作类**（抓取），证明你懂 MoveIt2 和执行链路，跟工具是互补的两个证据。

所以顺序是：**先做 W1 组网 → 踩坑过程沉淀成脚本 → 独立成新仓库。** 已有投入不浪费。

---

## 十、预期管理（必须说清楚，别到时候失望）

**ROS2 是 niche 领域，star 天花板大概在几百，不是几千。**
但在这个圈子里，几百 star 已经是"发到群里大家都见过"的级别。

要拉开差距靠这三件事，不靠功能堆砌：

1. **中文 + 英文双语 README**。中文社区（鱼香、古月居）是主战场，英文是 ROS Discourse / r/ROS。
2. **发布渠道已备好**，见 `docs/04-info-sources.md`：鱼香社区、古月居、ROS Discourse、r/ROS、GitHub Issues。
3. **README 第一屏的截图对比**决定传播率——"以前你要敲 8 条命令，现在敲 1 条"。

⚠️ 别买星、别刷星。假星在这个圈子里一眼看穿，且面试时讲到这个项目会心虚。

---

## 十一、待拍板（我可以先按默认往下做，你随时推翻）

1. 命名：默认 `r2net`
2. 是否现在就把新仓库建出来（默认：等 W1 组网踩完坑再建，避免空仓库）
3. v0.1 是否包含 L3（QoS 匹配）。默认**不包含**——先做网络层，QoS 已被 connection_inspector 占住，晚点做差异化
