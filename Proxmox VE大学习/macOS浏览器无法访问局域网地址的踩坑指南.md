## 问题描述

同一局域网内，MacBook Air M4（macOS 26）可以 `ping` 通 Proxmox VE 服务器（192.168.100.83），也能通过 `ssh root@192.168.100.83` 登录，但 Chrome / Edge 浏览器访问 `https://192.168.100.83:8006/` 显示 **ERR_ADDRESS_UNREACHABLE**。另一台 Debian 13 设备浏览器访问正常。

---

## 排查过程

### 1. 服务器端排查 — 一切正常

SSH 登录 PVE 服务器，依次检查：

```bash
# pveproxy 服务状态
systemctl status pveproxy --no-pager -l

# pvedaemon 服务状态
systemctl status pvedaemon --no-pager -l

# 8006 端口是否监听
ss -tlnp | grep 8006

# iptables 规则
iptables -L -n -v

# PVE 防火墙状态
pve-firewall status

# 访问日志
tail -30 /var/log/pveproxy/access.log

# fail2ban 状态
systemctl status fail2ban --no-pager
```

所有服务正常、端口监听、无防火墙拦截、无 IP 封禁，且从另一台机器用 `curl` 访问返回 HTTP 200。服务器端无问题。

### 2. 客户端排查 — curl 能通，浏览器不通

在 MacBook 终端执行：

```bash
# curl 直接请求 — 成功，返回 HTTP 200
curl -sk -o /dev/null -w "HTTP_CODE=%{http_code}\n" --connect-timeout 10 https://192.168.100.83:8006/

# nc 测试 TCP 连接 — 成功
nc -vz -w 5 192.168.100.83 8006

# 但用 Chrome 隐身模式、关闭代理/安全 DNS 后依然 ERR_ADDRESS_UNREACHABLE
```

`curl` 和 `nc` 都能建立 TCP 连接，说明网络层没有问题，问题出在 **浏览器进程层面**。

### 3. Chrome 网络日志分析

打开 Chrome 的 `chrome://net-export/`，开始记录日志后访问目标 URL，导出 JSON 分析：

```bash
# 搜索 192.168.100.83 相关事件
# 关键发现：
#   os_error=65  → macOS 错误码 EHOSTUNREACH ("No route to host")
#   net_error=-109 → Chromium ERR_ADDRESS_UNREACHABLE
```

TCP 连接流程：

```
SOCKET_CONNECT_START → TCP_CONNECT → TCP_CONNECT_COMPLETE(os_error=65)
→ SOCKET_CONNECT_FAILED(net_error=-109) → REQUEST_FAILED
```

但同一台机器上 `curl` 却正常，说明不是内核路由表的问题，而是有东西在 **浏览器进程的 Socket 层** 拦截了连接。

### 4. macOS 网络扩展排查

```bash
# 检查系统网络扩展
systemextensionsctl list
# 输出：0 extension(s) — 无第三方扩展

# 检查内核扩展
kextstat | grep -v com.apple
# 输出：空 — 无第三方 kext

# 检查 PF 防火墙
sudo pfctl -s info
# 输出：Status: Disabled — PF 未启用

# 检查 macOS 应用防火墙
/usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate
# 输出：Firewall is disabled

# 关键：检查网络扩展配置
sudo cat /Library/Preferences/com.apple.networkextension.plist | plutil -p -
```

在 networkextension.plist 中发现：

```
ContentFilter => Enabled: false, FilterSockets: true  ← 内容过滤器，Socket 层拦截
Name: com.apple.preferences.networkprivacy-...
Enabled: true          ← 已启用
IgnoreRouteRules: true ← 绕过正常路由表
Grade: 2
Rules: [53 条应用规则，包括 com.google.Chrome, com.microsoft.edgemac 等]
```

---

## 原因分析

macOS 的 **本地网络隐私控制器**（`NEPathController`）处于损坏状态。正常情况下，当 Chrome/Edge 首次尝试访问局域网地址时，macOS 应弹出"本地网络"权限对话框请求用户授权。但此处的 `com.apple.networkextension.plist` 配置文件损坏，导致：

1. `IgnoreRouteRules=true` — 控制器接管了这些应用的流量路由，使用自己的一套规则
2. `Enabled=true` 但认证签名信息不完整 — 规则已激活但无法正确放行流量
3. Chrome/Edge 发起 TCP 连接时，socket 被过滤器劫持，返回 `EHOSTUNREACH`

`curl` 和 `nc` 不受影响是因为它们不在该控制器的应用规则列表中。

---

## 解决方法

### 方法一：删除损坏的配置文件（推荐，一劳永逸）

```bash
sudo rm /Library/Preferences/com.apple.networkextension.plist
sudo rm /Library/Preferences/com.apple.networkextension.necp.plist
sudo killall -HUP configd
```

删除后重启 Chrome/Edge，首次访问局域网地址时 macOS 会重新弹出"本地网络"权限对话框，点击**允许**即可。之后系统会重新生成干净的配置文件。

### 方法二：通过系统设置授权（如果能看到选项）

「系统设置」→「隐私与安全性」→「本地网络」，找到 Chrome 和 Edge，勾选启用。

### 方法三：用干净的 Chrome 实例临时绕过

如果急需访问，可以在终端启动一个无代理、忽略证书错误的临时 Chrome：

```bash
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --user-data-dir=/tmp/chrome-test \
  --no-proxy-server \
  --ignore-certificate-errors \
  https://192.168.100.83:8006/
```

---

## 附录：快速诊断命令速查

| 检查项 | 命令 |
|--------|------|
| 网络扩展配置 | `sudo plutil -p /Library/Preferences/com.apple.networkextension.plist` |
| 系统扩展列表 | `systemextensionsctl list` |
| PF 防火墙状态 | `sudo pfctl -s info` |
| 应用防火墙状态 | `/usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate` |
| 端口连通性 | `nc -vz -w 5 <IP> <PORT>` |
| Chrome 网络日志 | 打开 `chrome://net-export/` |
| 浏览器 net_error 含义 | `-109` = `ERR_ADDRESS_UNREACHABLE`, `-105` = `ERR_NAME_NOT_RESOLVED` |
| macOS errno 含义 | `60` = `ETIMEDOUT`, `61` = `ECONNREFUSED`, `65` = `EHOSTUNREACH` |
