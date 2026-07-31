## 第一部分：总体架构

### 1. 部署目标

在 Proxmox VE 服务器（192.168.100.83，PVE 9.2.2）上部署三个虚拟机：

| VM | 用途 | 系统 |
|----|------|------|
| VM 101 | NAS 文件存储（**珍贵老照片**） | 飞牛 OS（fnOS） |
| VM 102 | Minecraft 游戏服务器 | Debian 13 |
| VM 103 | 轻量级 Web 应用测试服务器（网站测试 / 聊天机器人 / agentic 应用） | Debian 13 |

### 2. 宿主机硬件

| 项目 | 配置 |
|------|------|
| 主板 | 华擎 B450M Pro4-F（4× SATA3、M2_1 NVMe、M2_2 SATA M.2、支持 IOMMU） |
| CPU | AMD Ryzen 5 3400G（4核8线程，Vega 11 核显可直通） |
| 内存 | 16GB DDR4 3200MHz + 8GB swap |
| 系统盘 | 256GB 致钛 ZHITAI PC005（NVMe，M2_1，PVE 所在） |
| 数据盘 | 512GB Samsung 860 EVO（SATA SSD，**未使用**，规划给 NAS） |
| 备份盘 | 2TB WD Elements 2.5寸移动机械硬盘（USB，每周本地备份） |

### 3. 架构总览

```
              家庭路由器 192.168.100.1（网关 + DHCP）
                          │
       ┌──────────────────┼───────────────────┐
       │                  │                   │
  PVE 宿主机         你的 MacBook          Debian 13 设备
  192.168.100.83     192.168.100.96        （备用备份点）
       │ (vmbr0 桥接，全部同网段)
  ┌────┼────────┬────────────────────────┐
  │            │                        │
VM 101       VM 102                   VM 103
飞牛 NAS     Minecraft                Web 测试服务器
.101         .102                     .103
2核/4G       4核/6G                   2核/2G
860 EVO      Paper+Java21             Docker Compose
直通+Btrfs   25565 端口                NPM 反代 + 多容器
```

**设计原则**：
- **三台独立**：各自独立的系统、IP、服务，互不干扰
- **NAS 数据安全第一**：单盘运行（接受单点风险），数据盘用 Btrfs + 3-2-1 备份兜底（见 VM 101 章节）
- **测试环境隔离**：VM 103 是沙盒，测试项目用完即删，不影响 NAS 和游戏服

### 4. PVE 虚拟化原理（简述）

PVE（Proxmox VE）不是简单的「虚拟机软件」，而是一个完整的虚拟化平台。理解下面七点，指南中的每个技术决策就都有了依据。

**① KVM + QEMU：PVE 的核心引擎**

| 组件 | 角色 |
|------|------|
| **KVM**（Kernel-based Virtual Machine） | Linux 内核内置的虚拟化模块，直接利用 CPU 的硬件虚拟化指令（AMD 的 SVM/V，即 BIOS 里的 SVM Mode），让虚拟机 CPU 指令**近乎原生速度**执行 |
| **QEMU** | 负责模拟虚拟机的「设备」：网卡、磁盘控制器、显卡、主板等，让虚拟机以为自己在真实硬件上运行 |
| PVE = 管理壳 | 在 KVM/QEMU 之上提供 Web 界面、存储管理、快照、备份、网络配置等编排能力 |

> 这也是为什么 BIOS 必须开 **SVM Mode**：不开它，KVM 只能纯软件模拟（慢 10 倍以上）；开了，虚拟机 CPU 性能几乎等于物理机。

**② 资源怎么分：CPU / 内存**

- **CPU（vCPU）**：一个「核」= 一个线程。`--cores 4` 表示给 VM 4 个虚拟核，由宿主的 8 线程分时调度。**虚拟机空闲时不吃核**，所以 8 线程可以全部分出去（超分），同时跑满才可能互相抢。
- **CPU 类型 `host`**：让虚拟机直接使用宿主 CPU 的全部指令集（AVX、SSE 等）。对 Minecraft 尤其重要——游戏服务端大量用 SIMD 指令，用默认的兼容型号会明显变慢。
- **内存**：PVE 默认开内存气球（ballooning），可动态回收空闲内存。但游戏/NAS 有突发内存需求，回收后再分配有延迟会造成卡顿，所以指南里三台都 `--balloon 0` 关掉，内存固定。

**③ 磁盘怎么接：虚拟磁盘 vs 直通**

| 方式 | 原理 | 特点 |
|------|------|------|
| **虚拟磁盘**（local-lvm:32） | 数据存在 NVMe 上的一个 LVM 卷/镜像文件里，由 PVE 管理 | 支持快照、备份、克隆；但有一层虚拟化开销 |
| **整盘直通**（860 EVO） | PVE 把整块物理盘作为块设备直接透传给 VM，VM 自己格式化、自己管 SMART | 性能与直连一致；PVE 无法对它快照，数据由 VM 自己负责 |

> 磁盘直通走的是 virtio-scsi 控制器（QEMU 模拟）透传块设备，**不涉及 PCIe 设备直通，因此不需要 IOMMU**。指南里直通 860 EVO 不依赖 BIOS 的 IOMMU 开关，就是这个原因（IOMMU 只对 GPU/NVMe 这类 PCIe 设备直通必需）。

**④ 网络怎么接：桥接（vmbr0）**

PVE 在物理网卡上创建一个 Linux 网桥 `vmbr0`，虚拟机网卡（virtio-net）直接挂到桥上：

```
物理网卡 (enp/eth0) ── vmbr0 网桥 ──┬─ VM101 虚拟网卡 (virtio)
                                     ├─ VM102 虚拟网卡
                                     ├─ VM103 虚拟网卡
                                     └─ PVE 宿主机自身
```

- 桥接 = **二层交换**，虚拟机与宿主机、路由器在同一个广播域（同一个网段），每台 VM 就像插在同一台交换机上的独立电脑——所以每台 VM 有自己的 IP，可直接被内网访问
- 网卡类型选 **virtio**（半虚拟化）：虚拟机和宿主内核协作收发数据，比模拟 e1000 网卡快得多
- 对比 NAT 模式（虚拟化软件默认）：虚拟机藏在宿主后面、共用宿主 IP，外部访问必须靠转发，且每台 VM 不在局域网可见——这正是本指南选桥接的原因

**⑤ 存储后端：LVM-thin / 目录 / ZFS（磁盘放哪里）**

PVE 安装时默认建两种存储：`local`（目录型，放 ISO、备份、容器模板）和 `local-lvm`（LVM-thin）。虚拟磁盘可以放在不同后端，各有取舍：

| 存储类型 | 原理 | 快照 | 备份 | 适用 |
|----------|------|------|------|------|
| **LVM-thin**（本指南 `local-lvm`） | 基于 LVM2 的精简置备卷池。创建 VM 盘时**不预占全部空间**，只按实际写入量增长（thin provisioning）；数据存为块设备，直接读写 NVMe，无文件系统层开销 | ✅ 秒级、低开销 | ✅ | 系统盘（性能好，快照成本低） |
| **目录型**（`local`） | 虚拟盘是一个普通文件（qcow2/raw），存在文件系统上 | qcow2 支持，raw 不支持 | ✅ | ISO/备份文件；灵活性高 |
| ZFS | 自带校验和 + 快照 + 压缩，数据安全性最强 | ✅ 秒级 | ✅ | 数据盘（吃内存，本机 16G 不推荐给 VM 用） |

**LVM-thin 的几个要点**：
- **精简置备**：给 VM 分配 32G，实际可能只用了 8G，剩余空间其他 VM 和快照共用。但**不是无限超分**——物理池（NVMe 剩余空间）满了 VM 会报错，所以指南里三个 32G 系统盘 + PVE 系统 30G 在 256G NVMe 上留足余量
- **块设备直读**：LVM-thin 卷没有文件系统层，写入路径短，比 qcow2 文件略快
- 查看：`pvesm status` 显示 `local-lvm` 的类型就是 `lvmthin`

**⑥ 快照 / 备份机制（数据保护怎么工作）**

| 机制 | 原理 | 特点 |
|------|------|------|
| **快照（Snapshot）** | 冻结 VM 当前状态：内存 + 磁盘。LVM-thin 用 **COW（写时复制）**——快照瞬间完成，之后写新数据时旧数据块保留给快照，所以快照几乎不占空间、秒级完成 | 适合**短期回滚**（升级前拍一个，出问题立刻还原）；快照不是备份，删了快照旧数据就回收了 |
| **备份（vzdump）** | 把整个 VM 打包成归档文件（含配置），存到备份存储（local 或 NAS），可恢复到任意时间点 | 适合**长期留档**；支持压缩、加密、定期任务 |
| **直通盘例外** | 860 EVO 是透传物理盘，PVE **看不到文件系统**，无法对它做快照/备份 | NAS 数据的安全靠飞牛内部：Btrfs 快照 + 3-2-1 备份（见 VM 101 章节） |

**实操对应关系**：
- VM 101/102/103 的系统盘（在 local-lvm 上）→ PVE 里可随时快照 + 定期备份
- VM 101 的 860 EVO（直通）→ 只能靠飞牛内部的 Btrfs 快照和备份任务
- 升级系统/装软件前：先拍快照（Web 界面 VM → 快照 → 拍摄），出问题一键回滚；确认稳定后再删除旧快照释放空间

**⑦ 一次创建流程背后的原理**（对应附录脚本）：

```
qm create 101 ...            → 在 /etc/pve/qemu-server/101.conf 生成 VM 配置
  --ide2 local:iso/xxx.iso   → ISO 作为光驱设备挂给 VM，供安装启动
  --net0 virtio,bridge=vmbr0 → 建一块 virtio 网卡挂到 vmbr0
  --scsihw virtio-scsi-pci   → 建一个 virtio-SCSI 控制器（磁盘都挂它下面）
qm set 101 --scsi0 local-lvm:32   → 在 NVMe 存储上创建 32G 虚拟盘
qm set 101 --scsi1 /dev/disk/...  → 把物理盘作为第二个 scsi 设备透传
qm start 101                 → KVM 启动进程，CPU 用 SVM 硬件虚拟化执行 VM 指令
```

### 5. 资源分配规划（基于 4核8线程 / 16G 内存）

| VM | CPU 核心 | 内存 | 系统盘（NVMe 上） | 数据盘 | 说明 |
|----|----------|------|-------------------|--------|------|
| VM 101 飞牛 NAS | 2 | 4G | 32G | **512G 860 EVO 整盘直通** | 数据盘用 Btrfs（校验和防静默损坏） |
| VM 102 Minecraft | 4 | 6G | 32G | — | 游戏服务器吃单核性能+内存 |
| VM 103 Web 测试服务器 | 2 | 2G（可调至 4G） | 32G | — | Docker 跑多个轻量应用，按需加内存 |

- **内存**：基础分配 12G/16G，剩余 4G 留给 PVE 宿主 + 8G swap 兜底，不超分
- **CPU**：8 线程全排满（2+4+2），虚拟机空闲不吃核，可正常超分
- **磁盘**：三个系统盘共 96G 在 256G NVMe 上（PVE 系统约占 30G，富余充足）；NAS 数据走 860 EVO 整盘直通

### 6. 部署前准备

**① BIOS 设置（华擎 B450M Pro4-F）** — 开机按 `Del` 进 UEFI：

```
Advanced → AMD CBS → NBIO Common Options → IOMMU → Enabled
Advanced → AMD CBS → CPU Common Options → SVM Mode → Enabled（虚拟化）
Advanced → PCI Configuration → Above 4G Decoding → Enabled（预留核显直通用）
```

> IOMMU/SVM 开启后，未来可做核显直通（见补充说明 Q4）；即使不做，开启也无副作用。

**② 下载所需镜像**

| 镜像 | 用途 | 下载地址 |
|------|------|----------|
| 飞牛 OS ISO | VM 101 | https://www.fnos.com.cn/（首页「下载」，ISO 链接带时效签名，需即时下载） |
| Debian 13 netinst ISO | VM 102/103 | https://www.debian.org/distrib/netinst（amd64） |

上传到 PVE：Web 界面 `数据中心 → 存储(local) → ISO 镜像 → 上传`，或：

```bash
scp fnos-*.iso root@192.168.100.83:/var/lib/vz/template/iso/
scp debian-13*.iso root@192.168.100.83:/var/lib/vz/template/iso/
```

**③ 确认数据盘设备名**（SSH 到 PVE 执行）

```bash
ls -l /dev/disk/by-id/ | grep -i samsung
# 输出形如：ata-Samsung_SSD_860_EVO_512GB_S3Z1NB0M123456 -> ../../sdX
```

记录 `ata-...` 全名，磁盘直通用。

---

## 第二部分：网络配置

### 1. 家庭内网

- 网段：`192.168.100.0/24`
- 路由器（网关 + DHCP）：`192.168.100.1`
- 所有虚拟机桥接 `vmbr0`，与 PVE 宿主机、Mac、Debian 设备同网段

**关键概念**：桥接模式下，每台 VM 相当于内网中**新增的一台独立设备**，有自己的 IP。访问对应虚拟机 = 在浏览器/客户端输入该 VM 的 `IP:端口`。

### 2. IP 规划表

| 设备 | IP | 端口 |
|------|-----|------|
| 路由器（网关+DHCP） | 192.168.100.1 | — |
| PVE 宿主机 | 192.168.100.83 | 8006 (Web) |
| VM 101 飞牛 NAS | 192.168.100.101 | 5666 (Web) / 445 (SMB) |
| VM 102 Minecraft | 192.168.100.102 | 25565 |
| VM 103 Web 测试服务器 | 192.168.100.103 | 80/443 (反代入口) / 81 (NPM管理) / 8080+ (各应用) / 6099 (NapCat) |

### 3. 访问方式速查

| 目标 | 访问方式 |
|------|----------|
| PVE 管理界面 | `https://192.168.100.83:8006` |
| 飞牛 NAS | `http://192.168.100.101:5666` |
| Minecraft | 游戏内服务器地址填 `192.168.100.102`（端口默认 25565） |
| Web 测试服务器各应用 | `http://192.168.100.103:8080` 等（见 VM 103 章节） |

### 4. IP 固定策略（重要）

Debian VM 安装时直接配静态 IP；飞牛 DHCP 获取后改静态。**建议同时在路由器后台将 .101/.102/.103 加入 DHCP 静态绑定/地址保留**，防止地址被其他设备（手机、电视等）抢占导致访问不到。

### 5. 外网访问与远程管理（进阶）

#### 5.1 需求拆解：管理类 vs 服务类

外网访问分两种完全不同的需求，策略不同：

| 类型 | 目标 | 谁访问 | 推荐策略 |
|------|------|--------|----------|
| **管理类** | PVE(8006)、飞牛(5666)、NPM(81) | 只有你自己 | **VPN / 官方远程**，不裸奔公网 |
| **服务类** | Minecraft(25565)、网站(80/443) | 外网玩家/访客 | 公网 IPv4 转发 或 IPv6 放行 |

> 核心原则：**管理入口直接暴露公网 = 把家门钥匙挂门口**。即使有公网 IP，也不要把 PVE 8006 端口转发出去。

#### 5.2 有公网 IPv4 时

除了向运营商申请公网 IP（打客服电话，说明需要公网 IP 做服务器；或办企业宽带），还需 4 件事：

**① 域名 + DDNS（必须）**：运营商公网 IP 是动态的。注册域名（阿里云/腾讯云/Cloudflare，约 30-60 元/年），内网跑 DDNS 客户端（如 `ddns-go`，可装在任意 VM 上），IP 变化自动更新 DNS。

**② 路由器端口转发**（只转服务类端口）：

| 外网端口 | → 内网 | 用途 |
|----------|--------|------|
| 25565 | 192.168.100.102:25565 | Minecraft 玩家 |
| 80/443 | 192.168.100.103:80/443 | 网站测试 |
| ~~8006~~ | ~~PVE~~ | **不要转发**（管理走 VPN） |

**③ HTTPS 证书**：网站测试用 Let's Encrypt 免费证书（NPM 里可自动申请）。

**④ 安全加固**：MC 开启正版验证/白名单；NPM 管理界面改强密码；网站如需半公开可加访问密码。

#### 5.3 没有公网 IPv4，只有 IPv6 时

中国家庭宽带（电信/联通/移动）普遍自带 IPv6。**IPv6 无 NAT**，桥接模式下 VM 自动获得公网 IPv6 地址，无需端口转发，只需放行防火墙。

**① 确认 IPv6 已通**（SSH 到 PVE）：
```bash
ip -6 addr show vmbr0
# 看到 240e:/2408:/2409: 开头 = 已有公网 IPv6（中国运营商段）
# 只有 fe80: 开头 = 只有链路本地地址，IPv6 没通（去路由器开启）
```

**② 路由器开启 IPv6 + 防火墙放行**：
- 光猫/路由器 IPv6 模式设为桥接/Native
- 路由器 IPv6 防火墙默认**阻断全部入站**，需放行 25565（MC）、80/443（网站）等端口
- PVE 和 VM 内防火墙（ufw）同样放行对应端口

**③ DDNS 支持 AAAA 记录**：IPv6 地址随运营商前缀租约变化，DDNS 客户端需更新 AAAA 记录（ddns-go 支持）。

**④ Minecraft 监听 IPv6**（关键坑）：
```bash
# server.properties 中 server-ip= 留空（监听所有接口，含 IPv6）
# 启动 JVM 加参数：
java -Djava.net.preferIPv6Addresses=true -Xms4G -Xmx5G -jar paper.jar nogui
```

**⑤ 玩家连接方式**：
- 服务器地址填 `你的域名:25565`（域名解析 AAAA 记录）
- 或直接填 `[240e:xxxx:...]:25565`（IPv6 地址必须用方括号）

**IPv6 注意事项**：
- 玩家必须有 IPv6 网络（手机 4G/5G 基本都有；部分公司/校园网没有）
- 前缀随光猫重启/租约变化 → DDNS 定期刷新
- 双栈最佳：IPv4 + IPv6 同时开，玩家哪个通走哪个

#### 5.4 Tailscale 工作原理（简述）

Tailscale = **基于 WireGuard 的自动组网工具**：让「你家内网的 PVE」和「任何地方的你的设备」直接组成一个加密的虚拟局域网，仿佛 PVE 就在身边。三大要点：

**① 底层加密隧道：WireGuard**
- 每台设备生成一对密钥（公钥 + 私钥）
- 发送方用目标设备的公钥加密数据，通过 UDP 直发；接收方用私钥解密
- 中间任何人（运营商/黑客）看不到内容；WireGuard 是 Linux 内核级协议，几乎不占资源

**② 控制平面 vs 数据平面分离（核心设计）**
- **控制平面**：Tailscale 协调服务器只负责「牵线搭桥」——登记设备、交换公钥、告知彼此地址（登录/握手时短暂通信）
- **数据平面**：真正的访问流量在设备之间**点对点直连**，不经过 Tailscale 服务器
- 对比传统 VPN（如 OpenVPN）：所有流量绕道自家 VPN 服务器，服务器是瓶颈；Tailscale 直连，速度取决于两端网络

**③ NAT 穿透（打洞）—— 为什么不需要公网 IP**
- 你家 PVE 在路由器（NAT）后面。PVE 主动向 Tailscale 协调服务器发 UDP 包，路由器 NAT 记录映射关系
- 协调服务器把双方「公网地址:端口」互相交换后，Mac 与 PVE 的 NAT 映射地址**直接互发 UDP**——路由器看到「来自刚通信过的地址」即放行
- 若打洞失败（对称型 NAT/严格防火墙），自动降级走 **DERP 中继服务器**（数据仍加密，只是非直连）

**用你的场景走一遍**：在公司访问 `https://100.x.y.z:8006` → Tailscale 客户端查表（目标=PVE 的公钥+地址）→ 用 PVE 公钥加密数据 UDP 直发 → 你家路由器转给内网 PVE → 解密处理返回。**在外网用 Tailscale IP 访问，和在家体验完全一样。**

**为什么比端口转发安全**：

| 对比 | 端口转发方案 | Tailscale |
|------|------------|-----------|
| 暴露面 | 8006 对全世界开放，任何人可扫描爆破 | 只对你的 Tailscale 网络设备开放，公网不可见 |
| 认证 | 靠密码/证书 | 登录授权 + 设备密钥 + 加密隧道（三重） |
| 加密 | 取决于服务自身 | 全链路 WireGuard 加密 |
| 公网 IP | 必需 | 不需要（打洞） |

> 一句话：端口转发是「把门开在公网上靠锁挡人」；Tailscale 是「门根本不在公网上，只有你兜里有钥匙」。

#### 5.5 远程管理执行方案（推荐：Tailscale + FN Connect）

**决策**：远程管理**不走公网端口转发**，统一采用 **Tailscale（PVE + 测试服务器）** 和 **飞牛 FN Connect（NAS）**。两者都不需要公网 IP、不暴露任何端口到公网。

**① 在 PVE 上安装 Tailscale**

```bash
# SSH 到 PVE
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up --ssh
# 终端会输出一个 https://login.tailscale.com/... 链接
# 在浏览器（本机或手机）打开，用你的账号（Google/微软/GitHub 等）登录授权
```

> `--ssh` 参数让 Tailscale 接管 SSH，之后在外网可直接 `ssh root@<PVE的tailscale-IP>`，且所有设备间流量加密。

**② 在你的 MacBook / 手机上装 Tailscale 客户端**

- Mac：App Store 搜索 Tailscale 或官网下载，登录**同一账号**
- 手机（iOS/Android）：App Store 装 Tailscale，登录同一账号
- 登录后，Mac/手机与 PVE 自动组成虚拟局域网，互相能看到对方

**③ 确认组网成功**

```bash
# PVE 上查看 tailscale 网络内的设备
tailscale status
# 输出里应看到你的 Mac 和手机（100.x.y.z 开头的 Tailscale 内网 IP）

# 在你的 Mac 上测试
ping 100.x.y.z        # 换成 PVE 的 tailscale IP
```

**④ 远程管理各服务的访问方式**（外网时用 Tailscale IP 访问，等效于在家）：

| 服务 | 在家访问 | 在外访问（Tailscale） |
|------|----------|----------------------|
| PVE 管理界面 | `https://192.168.100.83:8006` | `https://100.x.y.z:8006` |
| PVE SSH | `ssh root@192.168.100.83` | `ssh root@100.x.y.z` |
| Web 测试服务器 NPM | `http://192.168.100.103:81` | `http://100.x.y.z:81`（或给 VM 也装 Tailscale） |

> 可选进阶：给 VM 102/103 也装 Tailscale（Debian 上同一条安装命令），这样外网可直接管理每台 VM，而不只是 PVE。飞牛 VM 的 Tailscale 可在飞牛应用中心找相关应用或 Docker 方式安装。

**⑤ 飞牛 NAS：用官方 FN Connect（零配置）**

1. 飞牛 Web 界面 → 系统设置 → 账号 → 注册/登录飞牛云端账号
2. 开启 **FN Connect** 远程访问
3. 之后在外网浏览器打开飞牛提供的专属链接（如 `xxx.fnconnect.net`）即可访问 NAS，无需任何端口转发
4. 可选：飞牛里装 Tailscale 应用（应用中心搜索），与 PVE 组同一内网，访问更直接

**⑥ 安全加固（管理面）**

- PVE SSH 建议仅允许密钥登录（禁用密码）：
  ```bash
  # PVE 上编辑 /etc/ssh/sshd_config
  # PasswordAuthentication no
  # 然后 systemctl restart ssh
  ```
- PVE Web 界面：数据中心 → 权限 → 用户 → 为 root 启用**双重认证（TOTP）**
- 飞牛：开启账号 2FA、设置访问码
- **保持路由器上不做任何 8006/5666 的端口转发**

#### 5.6 外网访问整体策略（最终版）

```
【管理类】自己远程访问
  PVE / VM 后台     → Tailscale 虚拟局域网（无需公网 IP，加密）
  飞牛 NAS          → FN Connect 官方远程（零配置）

【服务类】对外提供服务（需要公网可达）
  Minecraft 25565   → 公网 IPv4 端口转发，或 IPv6 防火墙放行
  网站测试 80/443   → 公网 IPv4 端口转发，或 IPv6 防火墙放行
```

---

## 第三部分：各虚拟机部署

## VM 101：飞牛 OS（fnOS）NAS —— 老照片存储

### 1. 创建虚拟机

```bash
qm create 101 --name fnos-nas --memory 4096 --cores 2 --cpu cputype=host \
  --net0 virtio,bridge=vmbr0 --scsihw virtio-scsi-pci \
  --ide2 local:iso/fnos-0.9.32-1178.iso,media=cdrom \
  --boot order=ide2;scsi0 \
  --ostype l26

# 系统盘 32G（NVMe 上的 local-lvm）
qm set 101 --scsi0 local-lvm:32

# 数据盘：860 EVO 整盘直通（无需 IOMMU，virtio-scsi 透传块设备）
qm set 101 --scsi1 /dev/disk/by-id/ata-Samsung_SSD_860_EVO_512GB_S3Z1NB0M123456

# 关闭内存气球
qm set 101 --balloon 0
```

### 2. 启动安装

```bash
qm start 101
```

noVNC 控制台走安装向导：选简体中文 → 安装磁盘选 **32G 系统盘（scsi0）** → 等待完成重启。

> 官方确认 fnOS 在 PVE 下受支持，CPU 类型必须 `host`（已在命令中配置）。

### 3. 初始化 NAS（重点：数据安全）

1. 控制台显示 Web 地址 `http://192.168.100.x:5666`，改为**静态 IP `192.168.100.101`**
2. 登录后创建存储空间：
   - 文件系统：**选 Btrfs**（带 checksum 校验，能检测静默数据损坏——对老照片至关重要；ext4 无此能力）
   - 硬盘：选直通的 512G 860 EVO（scsi1）
   - 存储模式：单盘（未来加盘后可在同一存储空间下组 RAID 1）
3. 开启 SMB/NFS 共享，供 Mac/Windows/手机访问

> **为什么选 Btrfs**：Btrfs 对每个文件块计算校验和，定期 scrub 能发现「硬盘没坏但数据悄悄损坏」的情况（SSD 长期断电/老化可能发生）。飞牛还支持 Btrfs 快照，误删照片可秒级回滚。

### 4. 老照片数据安全（核心章节）

**① 3-2-1 备份策略（必须执行）**

> RAID 只防硬盘损坏，不防误删、勒索病毒、火灾、雷击。单盘运行下，**备份就是你的全部防线**。

**3-2-1 原则**：**3** 份数据（原始 + 2 份备份）、**2** 种存储介质、**1** 份在异地。

| 层级 | 位置 | 介质 | 频率 | 说明 |
|------|------|------|------|------|
| 原始 | 飞牛 NAS（860 EVO） | SSD | — | 日常访问 |
| 备份 1 | **2TB WD Elements 移动硬盘**（USB 直通给 VM 101） | 机械盘 | 每周 | 飞牛「备份」应用定时任务 |
| 备份 2 | 网盘/云盘（加密后） | 云端 | 每月 | 飞牛「同步」或 rclone；异地防火灾/盗窃 |

**② 每周本地备份（2TB 移动硬盘）配置**

1. 移动硬盘插 PVE 机箱后置 USB 3.0 口，SSH 到 PVE 确认设备：
   ```bash
   lsusb | grep -i western
   # 记下 厂商ID:产品ID，如 1058:0748
   ```
2. USB 直通给 VM 101（用 vendor:product ID 比端口位置稳定，PVE 重启不失配）：
   ```bash
   qm set 101 --usb0 host=1058:0748
   ```
3. 飞牛「存储空间」格式化 2TB 盘（可选加密）→「备份」应用新建任务：源=照片存储空间，目标=2TB 盘，计划每周，保留最近 N 个版本
4. 移动盘设置**硬盘休眠**，减少机械盘磨损
5. 注意：USB 直通后该盘仅 VM 101 可见，宿主机不可访问；备份场景为顺序写入，SMR 盘无压力

**③ 每月异地备份**

飞牛「飞牛同步」→ 绑定网盘（百度/阿里/Onedrive 等）→ 照片目录设为同步 → 建议先加密（备份设置里有加密选项，防云端泄露隐私）。

**④ 日常防护**

- **Btrfs 快照**：存储空间 → 快照计划 → 每日一次，保留 30 天（误删/勒索病毒可回滚）
- **照片只读保护**：共享文件夹权限设为「只读」（家人用只读账号访问，防止误删）
- **命令行备份**（SSH 到任意 Linux 机器，rsync 到备份盘）：
  ```bash
  rsync -av --delete \
    liang@192.168.100.101:/volume1/photos/ \
    /mnt/backup_disk/photos/
  find /mnt/backup_disk/photos/ -type f -exec sha256sum {} + > photos.sha256
  ```

**⑤ 每年一次「恢复演练」**

真正检验备份是否有效的方法：**从备份盘恢复几张照片到新目录，确认能打开**。备份了 10 年却从未恢复过 = 没有备份。

**⑥ 未来升级 RAID 1（推荐，主板已预留）**

购买一块与 860 EVO 同规格的 **512G SATA SSD**（同容量/同接口即可，无需同品牌），插入空闲 SATA3 口：

```bash
# SSH 到 PVE：确认新盘设备名，直通给 VM 101（scsi2 加第二块盘）
ls -l /dev/disk/by-id/ | grep -i <新盘品牌>
qm set 101 --scsi2 /dev/disk/by-id/ata-新盘设备名
```

然后在飞牛里：存储空间 → 添加硬盘 → 选择新盘 → **升级为 RAID 1（镜像）**。之后任意一块盘损坏，照片不丢，直接热替换坏盘即可。这是低成本、高收益的升级路径，建议数据积累到 200G 前完成。

---

## VM 102：Debian Minecraft 服务器

### 1. 创建虚拟机

```bash
qm create 102 --name mc-debian --memory 6144 --cores 4 --cpu cputype=host \
  --net0 virtio,bridge=vmbr0 --scsihw virtio-scsi-pci \
  --ide2 local:iso/debian-13-netinst-amd64.iso,media=cdrom \
  --boot order=ide2;scsi0 \
  --ostype l26

qm set 102 --scsi0 local-lvm:32
qm set 102 --balloon 0
qm start 102
```

### 2. 安装 Debian 13

主机名 `mc-server`，静态 IP `192.168.100.102/24`，网关 `192.168.100.1`，软件只勾选 **SSH server**。

### 3. 安装 Java + 部署 Paper

```bash
ssh root@192.168.100.102
apt update && apt upgrade -y
apt install -y openjdk-21-jre-headless screen wget

mkdir -p /opt/mc && cd /opt/mc
wget https://api.papermc.io/v2/projects/paper/versions/1.21.4/builds/123/downloads/paper-1.21.4-123.jar -O paper.jar

java -Xms4G -Xmx5G -jar paper.jar nogui   # 首次运行生成配置
sed -i 's/eula=false/eula=true/' eula.txt
java -Xms4G -Xmx5G -jar paper.jar nogui
```

> 3400G 单核性能有限，适合 1~8 人小服。

### 4. systemd 开机自启

创建 `/etc/systemd/system/mc-server.service`：

```
[Unit]
Description=Minecraft Paper Server
After=network.target

[Service]
WorkingDirectory=/opt/mc
ExecStart=/usr/bin/java -Xms4G -Xmx5G -jar paper.jar nogui
Restart=on-failure
User=root

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable --now mc-server
```

玩家连接：`192.168.100.102:25565`

---

## VM 103：轻量级 Web 应用测试服务器

**定位**：网站正式上线前的测试环境、聊天机器人、各种 agentic 应用（AI Agent / 工作流自动化等）。统一用 **Debian 13 + Docker Compose**，一个 VM 内跑多个互不干扰的容器；通过 **Nginx Proxy Manager** 做反向代理，用「域名 + 路径」区分不同应用，测试完直接删容器即可，不影响系统。

### 1. 创建虚拟机

```bash
qm create 103 --name web-test --memory 2048 --cores 2 --cpu cputype=host \
  --net0 virtio,bridge=vmbr0 --scsihw virtio-scsi-pci \
  --ide2 local:iso/debian-13-netinst-amd64.iso,media=cdrom \
  --boot order=ide2;scsi0 \
  --ostype l26

qm set 103 --scsi0 local-lvm:32
qm start 103
```

> 内存按需调整：同时跑 3~4 个轻量容器 2G 够用；跑 agentic 应用（如 Dify、n8n）建议 `qm set 103 --memory 4096`，注意总预算。

安装 Debian 13（主机名 `web-test`，静态 IP `192.168.100.103`，只勾选 SSH server）：

### 2. 安装 Docker + Docker Compose

```bash
ssh root@192.168.100.103
apt update && apt upgrade -y
curl -fsSL https://get.docker.com | sh
systemctl enable --now docker
apt install -y docker-compose-plugin    # docker compose 子命令
docker --version && docker compose version
```

### 3. 部署 Nginx Proxy Manager（统一入口）

```bash
mkdir -p /opt/proxy && cd /opt/proxy
cat > docker-compose.yml <<'EOF'
services:
  nginx-proxy-manager:
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: npm
    restart: always
    ports:
      - '80:80'      # HTTP 入口
      - '443:443'    # HTTPS 入口
      - '81:81'      # NPM 管理界面
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
EOF
docker compose up -d
```

管理界面：`http://192.168.100.103:81`（默认账号 `admin@example.com` / `changeme`，首次登录强制改密）。

> 之后每个应用都在 NPM 里添加一条「Proxy Host」规则，如 `test01.home.local → 192.168.100.103:8080`；访问 `http://test01.home.local` 即可。如需 HTTPS 测试，可申请自签证书或 Let's Encrypt（内网域名需要配 DNS 或 hosts）。

### 4. 应用模板（按需复制，一个应用一个 compose 目录）

**① 网站测试（示例：静态站点 / Nginx + 简单 PHP）**

```bash
mkdir -p /opt/apps/website && cd /opt/apps/website
cat > docker-compose.yml <<'EOF'
services:
  web:
    image: nginx:alpine
    container_name: website-test
    restart: always
    ports:
      - '8080:80'
    volumes:
      - ./html:/usr/share/nginx/html:ro   # 把你的测试站点文件放 ./html
EOF
mkdir -p html && echo '<h1>Hello Test Site</h1>' > html/index.html
docker compose up -d
```

测试地址：`http://192.168.100.103:8080`（或经 NPM 加域名规则）。

**② 聊天机器人（NapCat + 框架）**

```bash
mkdir -p /opt/apps/napcat && cd /opt/apps/napcat
cat > docker-compose.yml <<'EOF'
services:
  napcat:
    image: mlikiowa/napcat-docker:latest
    container_name: napcat
    restart: always
    ports:
      - '6099:6099'   # NapCat WebUI
      - '3001:3001'   # OneBot WebSocket
    volumes:
      - ./config:/app/napcat
    privileged: true
EOF
docker compose up -d
```

- WebUI：`http://192.168.100.103:6099/webui`，用**机器人小号**扫码登录
- 机器人逻辑：可在本 VM 里再起一个 Python 容器跑 **NoneBot2 / LangBot**，或直接用 QQ 群管理类框架（如 Lagrange 的 docker 镜像）

**③ Agentic 应用（AI Agent / 工作流）**

按项目挑一个（轻量优先，2G 内存下别同时全开）：

```bash
# n8n —— 工作流自动化（低代码 agent 编排）
mkdir -p /opt/apps/n8n && cd /opt/apps/n8n
cat > docker-compose.yml <<'EOF'
services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: always
    ports:
      - '5678:5678'
    environment:
      - N8N_HOST=192.168.100.103
    volumes:
      - ./data:/home/node/.n8n
EOF
docker compose up -d

# Dify —— 可视化 LLM 应用平台（较重，建议先加内存到 4G）
# Open WebUI —— LLM 聊天前端（配合 Ollama 使用）
# LangFlow / Flowise —— 拖拽式 agent 构建
```

常用端口速查：n8n `5678`、Dify `80`、Open WebUI `3000`、LangFlow `7860`、Ollama `11434`。

### 5. 测试环境管理建议

- **一个应用 = 一个 compose 目录**（`/opt/apps/<名字>/`），删除测试项目时直接 `docker compose down && rm -rf /opt/apps/<名字>`
- 数据卷（数据库等）放应用目录下，删容器不删数据；确认不要了再删目录
- 上线前测试网站：把真实站点文件挂进 nginx 容器，测完即弃，**不影响生产**
- 与 VM 102/101 隔离：测试用的数据库、缓存都跑在容器里，不会污染 NAS 和游戏服

---

## 第四部分：补充说明

### 1. 常见问题

**Q1：飞牛 OS 安装时看不到直通盘？**
- 系统盘（scsi0）和直通盘（scsi1）分开确认；直通盘不需要 IOMMU
- 确认 860 EVO 未被 PVE 挂载为存储：`pvesm status`，有则先卸载

**Q2：为什么数据盘选 Btrfs 而不是 ext4？**
Btrfs 有 checksum + 快照 + scrub，能检测并报告静默数据损坏，适合不可再生的珍贵数据；ext4 无校验能力。ZFS 校验更强但吃内存（16G 机器上飞牛只分 4G，不划算）。

**Q3：Minecraft 卡顿？**
CPU 类型必须 `host`；已关 balloon；JVM 5G 上限别超；人多换 Fabric + Lithium/Starlight。

**Q4：核显直通给飞牛做硬件转码（Jellyfin/Emby）？**
B450M Pro4-F + 3400G 支持。BIOS 已开 SVM/IOMMU/Above 4G（见第一部分），PVE 侧：

```bash
# 编辑 /etc/default/grub，GRUB_CMDLINE_LINUX_DEFAULT 加参数
#   amd_iommu=on iommu=pt video=efifb:off
# update-grub 后重启 PVE

# 查看核显 IOMMU 分组
lspci | grep -i vga        # 找到 3400G 核显的 BDF，如 00:08.1
# 把 00:08.1 和 00:08.2（HDMI 音频）加入 /etc/modprobe.d/vfio.conf：
#   options vfio-pci ids=1002:15dd,1002:15de
# 然后 qm set 101 --hostpci0 00:08.1,00:08.2
```

注意：核显直通后 **宿主机失去显示输出**（无头运行），且需显卡驱动（飞牛内置）。老照片存储本身不需要它，属可选进阶。

**Q5：备份 vs RAID 的区别？**
RAID 1 镜像 = 硬盘坏了数据还在（硬件冗余）；备份 = 误删/勒索/火灾后还能恢复（逻辑+物理冗余）。单盘运行时备份是唯一防线；加盘 RAID 1 后备份依然是必需的第二道防线。

**Q6：内存不够？**
基础分配 12G/16G。若三台满载 swap 抖动：停掉 VM 103 上不用的测试容器、飞牛降到 3G；跑重型 agentic 应用时临时给 VM 103 加内存（`qm set 103 --memory 4096`）用完再降；终极方案加内存条升 32G。

### 2. 完整创建命令（SSH 到 PVE 后执行，替换 ISO 文件名与磁盘路径）

```bash
# ---- VM 101 飞牛 NAS ----
qm create 101 --name fnos-nas --memory 4096 --cores 2 --cpu cputype=host \
  --net0 virtio,bridge=vmbr0 --scsihw virtio-scsi-pci \
  --ide2 local:iso/fnos-0.9.32-1178.iso,media=cdrom --boot order=ide2;scsi0 --ostype l26
qm set 101 --scsi0 local-lvm:32
qm set 101 --scsi1 /dev/disk/by-id/ata-Samsung_SSD_860_EVO_512GB_S3Z1NB0M123456
qm set 101 --balloon 0

# ---- VM 102 Minecraft Debian ----
qm create 102 --name mc-debian --memory 6144 --cores 4 --cpu cputype=host \
  --net0 virtio,bridge=vmbr0 --scsihw virtio-scsi-pci \
  --ide2 local:iso/debian-13-netinst-amd64.iso,media=cdrom --boot order=ide2;scsi0 --ostype l26
qm set 102 --scsi0 local-lvm:32
qm set 102 --balloon 0

# ---- VM 103 Web 测试服务器 ----
qm create 103 --name web-test --memory 2048 --cores 2 --cpu cputype=host \
  --net0 virtio,bridge=vmbr0 --scsihw virtio-scsi-pci \
  --ide2 local:iso/debian-13-netinst-amd64.iso,media=cdrom --boot order=ide2;scsi0 --ostype l26
qm set 103 --scsi0 local-lvm:32
```

> 注意：`ata-Samsung_SSD_860_EVO_512GB_S3Z1NB0M123456` 为示例，请以 `ls -l /dev/disk/by-id/ | grep -i samsung` 实际输出替换；ISO 文件名、存储名（`local-lvm`）同理按实际替换。

### 3. 部署顺序建议

1. **先配 BIOS**（IOMMU/SVM/Above 4G）→ 装 PVE 系统
2. **再传镜像** → 依次创建 101 → 102 → 103（建议 101 最先，NAS 初始化 + 备份配置好后，照片有地方放）
3. **配网络**：路由器 DHCP 静态绑定 .101/.102/.103
4. **最后做备份验证**：NAS 备份任务建好后，先手动跑一次并做恢复演练
