# openwrt-ddns-mac

为 OpenWrt 当前 master 的 `luci-app-ddns` / `ddns-scripts` 增加 **IPv6 按 MAC 地址查找目标内网设备地址** 的支持。

这个项目不是一个独立的 DDNS 客户端，而是两个补丁：

- `patches/0001-luci-app-ddns-ipv6-mac.patch`：修改 LuCI 的 `luci-app-ddns`，在 IPv6 的 IP 地址来源中增加 **MAC Address**，并从 `network.getHostHints()` 提供可选设备列表。
- `patches/0002-ddns-scripts-ipv6-mac.patch`：修改 `ddns-scripts`，增加 `ip_source=mac`，从 Linux IPv6 neighbor table（`ip -6 neigh`）按 MAC 查找目标设备的 IPv6 地址。

补丁基于 2026-10-03 读取到的 `openwrt/luci:master` 与 `openwrt/packages:master`。

## 适用场景

典型场景是：

> OpenWrt 路由器本身没有使用目标设备的 IPv6 地址，但希望 DDNS 记录指向局域网内某台设备的 IPv6 地址。

例如 NAS、服务器、电脑等设备使用 IPv6 SLAAC 后地址可能发生变化。启用本补丁后，DDNS 可以根据设备的 MAC 地址，从路由器的 IPv6 邻居表中找到该设备当前的活动 IPv6 地址，并把这个地址用于 DDNS 更新。

## 工作原理

普通 `ddns-scripts` 的 `ip_source` 通常用于获取路由器自己的 IPv4/IPv6 地址。本项目增加：

```text
ip_source=mac
        │
        ▼
ip -6 neigh show
        │
        ├─ 按 MAC 地址匹配
        ├─ 只接受 REACHABLE / DELAY / PROBE
        ├─ 排除 :: / ::1
        ├─ 排除 link-local（fe80::/10）
        ├─ 排除 multicast（ff00::/8）
        └─ 根据 upd_privateip 决定是否允许 ULA（fc00::/7）
        │
        ▼
得到目标设备 IPv6
        │
        ▼
ddns-scripts 执行正常的 DDNS 更新
```

实现会优先选择 `REACHABLE`，其次 `DELAY`，最后 `PROBE`。如果只有 `STALE` 等非活动状态的邻居缓存，不会直接使用该旧地址更新 DDNS。

Linux neighbor table 不提供远端 RFC 4941 临时地址标志，因此本项目**无法仅凭 neighbor table 判断某个 IPv6 是否为临时地址**。如果一个 MAC 同时存在多个候选 IPv6，当前实现按活动状态和地址排序选择一个候选地址。

## 前置条件

使用前需要准备：

1. OpenWrt 源码树。
2. 与补丁基线匹配的 `luci` 和 `packages` 源码。
3. 已安装/选择 `luci-app-ddns` 和 `ddns-scripts`。
4. 目标内网设备必须能够在 OpenWrt 路由器的 IPv6 neighbor table 中出现。
5. 目标设备的 MAC 地址必须已知。

建议先确认目标设备能够被 OpenWrt 看到：

```sh
ip -6 neigh show
```

然后按 MAC 检查：

```sh
ip -6 neigh show | grep -i 'lladdr AA:BB:CC:DD:EE:FF'
```

将 `AA:BB:CC:DD:EE:FF` 替换成实际 MAC。

## 获取源码

如果还没有 OpenWrt 源码，可以先准备标准 OpenWrt 编译环境和源码，然后进入 OpenWrt 源码根目录。

本项目中的补丁需要分别作用于：

- `luci-app-ddns`：OpenWrt 的 `luci` feed
- `ddns-scripts`：OpenWrt 的 `packages` feed

如果使用标准 OpenWrt 源码树，通常目录结构类似：

```text
openwrt/
├── package/
├── feeds/
├── luci/
└── ...
```

实际目录位置取决于你的 OpenWrt 源码和 feed 管理方式。

## 应用补丁

进入 **OpenWrt 源码根目录**，执行：

```sh
patch -p1 < /path/to/openwrt-ddns-mac/patches/0001-luci-app-ddns-ipv6-mac.patch
patch -p1 < /path/to/openwrt-ddns-mac/patches/0002-ddns-scripts-ipv6-mac.patch
```

如果你已经把本项目目录放在 OpenWrt 源码根目录下，则可以使用：

```sh
patch -p1 < patches/0001-luci-app-ddns-ipv6-mac.patch
patch -p1 < patches/0002-ddns-scripts-ipv6-mac.patch
```

### 检查补丁是否正确应用

正常情况下，`patch` 应该显示各文件成功应用，没有 `FAILED`。

可以检查关键内容：

```sh
grep -n 'ip_source.*mac' net/ddns-scripts/files/usr/lib/ddns/dynamic_dns_functions.sh
grep -n 'ip_mac' net/ddns-scripts/files/usr/lib/ddns/dynamic_dns_functions.sh
```

LuCI 部分可以检查：

```sh
grep -n 'MAC Address' applications/luci-app-ddns/htdocs/luci-static/resources/view/ddns/overview.js
```

如果补丁无法应用，通常表示当前 `luci` / `packages` 源码与本项目补丁基线不同。不要强行忽略 `patch` 的冲突，应先确认源码版本，再手工适配补丁。

## 编译

在 OpenWrt 源码根目录执行：

```sh
make package/luci-app-ddns/compile V=s
make package/ddns-scripts/compile V=s
```

如果你的源码树使用 feeds 中的包目录，或者需要完整构建固件，则按照对应 OpenWrt 构建流程选择这些包。

编译成功后，将生成的对应 `ipk` 安装到目标 OpenWrt 路由器。

## 安装到路由器

将编译生成的 `luci-app-ddns` 和 `ddns-scripts` 包复制到路由器，然后使用 OpenWrt 的包管理器安装。

例如：

```sh
opkg install /tmp/luci-app-ddns_*.ipk
opkg install /tmp/ddns-scripts_*.ipk
```

如果当前 OpenWrt 版本使用新的 `apk` 包管理方式，请使用该版本对应的安装方法。

安装/升级后建议重启相关服务；如果 LuCI 页面没有立即显示新选项，可以重新登录 LuCI 或重启 `uhttpd`。

## LuCI 配置方法

登录 OpenWrt LuCI，进入：

```text
服务 → 动态 DNS
```

创建或编辑一个 DDNS 服务。

### 1. 启用 IPv6

在服务配置中启用：

```text
使用 IPv6 / IPv6
```

本项目的 MAC 地址来源**仅对 IPv6 有效**。

### 2. 选择 MAC Address

在高级设置的 **IP address source（IP 地址来源）** 中：

```text
IPv6
  └── MAC Address
```

选择 **MAC Address**。

IPv4 的 IP 地址来源不会出现 MAC Address。

### 3. 选择目标设备

随后会出现 **MAC Address** 字段。

LuCI 会利用现有的 `network.getHostHints()` 生成设备列表，通常显示为：

```text
设备名称 — AA:BB:CC:DD:EE:FF
```

例如：

```text
NAS — AA:BB:CC:DD:EE:FF
```

如果之前保存过某个 MAC，但当前它没有出现在 host hints 中，LuCI 仍会保留该配置，并显示为：

```text
未知设备 — AA:BB:CC:DD:EE:FF
```

也可以直接填写合法的 MAC 地址。

### 4. 保存配置

保存并应用配置。

对应的 UCI 配置核心内容类似：

```uci
config service 'nas'
    option enabled '1'
    option use_ipv6 '1'
    option ip_source 'mac'
    option ip_mac 'AA:BB:CC:DD:EE:FF'
```

实际配置还需要包含 DDNS 服务商、域名、认证信息、事件网络等正常的 `ddns-scripts` 配置。

## UCI 手工配置

不使用 LuCI 时，可以直接编辑：

```sh
vi /etc/config/ddns
```

一个最小的 MAC IPv6 来源示例：

```uci
config service 'nas'
    option enabled '1'
    option use_ipv6 '1'
    option ip_source 'mac'
    option ip_mac 'AA:BB:CC:DD:EE:FF'
```

注意：这只是本项目新增的 IP 获取部分。要真正完成 DDNS 更新，还必须按照所使用的 DDNS 服务商配置 `service_name`、`domain`、`username`、`password` 等必要选项。

例如完整结构可以是：

```uci
config service 'nas'
    option enabled '1'
    option use_ipv6 '1'
    option service_name '你的DDNS服务商'
    option domain 'nas.example.com'
    option username '你的用户名或Token'
    option password '你的密码或Token'
    option interface 'wan'
    option ip_source 'mac'
    option ip_mac 'AA:BB:CC:DD:EE:FF'
```

具体认证字段和 `service_name` 的取值取决于 DDNS 服务商，请以 OpenWrt 当前版本的 `ddns-scripts` 文档和服务商要求为准。

## 命令行验证

### 查看 IPv6 neighbor table

首先确认 MAC 对应的设备确实存在：

```sh
ip -6 neigh show | grep -i 'lladdr AA:BB:CC:DD:EE:FF'
```

可能看到类似：

```text
2001:db8:1234:5678::1234 dev br-lan lladdr aa:bb:cc:dd:ee:ff REACHABLE
```

其中：

- `2001:db8:1234:5678::1234`：目标 IPv6
- `br-lan`：邻居所在接口
- `aa:bb:cc:dd:ee:ff`：目标 MAC
- `REACHABLE`：当前邻居状态

### 直接测试 ddns-scripts

项目增加了 `dynamic_dns_lucihelper.sh` 的 `-m` 参数：

```sh
/usr/lib/ddns/dynamic_dns_lucihelper.sh -6 -m AA:BB:CC:DD:EE:FF -- get_local_ip
```

这里：

- `-6)：使用 IPv6
- `-m`：指定 MAC 地址，并自动设置 `ip_source=mac)
- `-- get_local_ip`：执行本地 IP 获取测试

如果成功，日志中应能看到类似：

```text
MAC address: AA:BB:CC:DD:EE:FF
Found IPv6 address: 2001:db8:...
Current IPv6 '2001:db8:...' detected for MAC 'AA:BB:CC:DD:EE:FF'
```

如果没有活动邻居，会出现类似：

```text
No active IPv6 neighbor found for MAC 'AA:BB:CC:DD:EE:FF'
```

这时不会拿 `STALE` 等旧缓存地址作为结果。

## 为什么有 MAC，但找不到 IPv6？

首先检查：

```sh
ip -6 neigh show
```

再检查指定 MAC：

```sh
ip -6 neigh show | grep -i 'lladdr AA:BB:CC:DD:EE:FF'
```

重点确认邻居状态。

本实现只接受：

```text
REACHABLE
DELAY
PROBE
```

以下状态不会被直接使用：

```text
STALE
FAILED
INCOMPLETE
NOARP
```

因此，如果目标设备长时间没有通信，neighbor table 中可能只剩 `STALE`，此时 DDNS 更新会失败，而不是错误地继续使用可能已经失效的 IPv6。

可以先让目标设备产生 IPv6 流量，再重新检查：

```sh
ip -6 neigh show | grep -i 'lladdr AA:BB:CC:DD:EE:FF'
```

## IPv6 地址过滤规则

`get_ipv6_by_mac()` 会过滤以下地址：

- `::`
- `::1`
- `fe80::/10` link-local
- `ff00::/8` multicast

对于 ULA：

```text
fc00::/7
```

是否允许由现有 `ddns-scripts` 的 `upd_privateip` 设置决定。

因此，如果你的目标是把公网 IPv6 DDNS 指向目标设备，应确认最终选择的是可从公网访问的 IPv6 地址，并正确配置 `upd_privateip`。

## 临时 IPv6 地址注意事项

很多终端设备会同时拥有：

- 稳定/持久的 IPv6 地址
- RFC 4941 privacy/temporary IPv6 地址

Linux IPv6 neighbor table 本身没有一个可靠的字段告诉本补丁“这个地址是不是远端 RFC 4941 临时地址”。

因此，本项目不能保证在多个 IPv6 地址同时匹配同一 MAC 时一定选择你期望的那个地址。

当前实现不会简单地取 `ip -6 neigh` 输出的第一条，而是：

1. 按 MAC 过滤；
2. 过滤不可用地址；
3. 过滤不允许的 ULA；
4. 只接受活动邻居状态；
5. 按 `REACHABLE → DELAY → PROBE` 排序；
6. 同等级情况下按 IPv6 地址排序；
7. 取第一个候选。

如果你的设备启用了隐私地址，需要结合实际网络环境验证 DDNS 最终记录是否符合预期。

## IPv4 使用说明

MAC Address 来源**不是 IPv4 功能**。

LuCI 中：

- IPv4：仍然使用 Network / URL / Interface / Script
- IPv6：可以额外使用 MAC Address

如果关闭 IPv6，LuCI 会自动把原来的 MAC 来源切回 Network，并清除 `ip_mac` 配置。

## 常见问题

### Q1：LuCI 中没有 MAC Address 选项

确认：

1. 已安装本项目修改后的 `luci-app-ddns`；
2. 当前 DDNS 服务已启用 IPv6；
3. 当前源码确实应用了 `0001-luci-app-ddns-ipv6-mac.patch`；
4. 重新登录 LuCI，必要时重启 `uhttpd`。

### Q2：命令提示 `source 'mac' requires IPv6`

这是预期行为。MAC 来源只支持 IPv6。

检查配置：

```uci
option use_ipv6 '1'
option ip_source 'mac'
```

### Q3：提示 `ip_mac not set`

说明：

```uci
option ip_source 'mac'
```

已经启用，但没有设置：

```uci
option ip_mac 'AA:BB:CC:DD:EE:FF'
```

### Q4：提示 `Invalid MAC address`

MAC 必须是标准六组十六进制格式，例如：

```text
AA:BB:CC:DD:EE:FF
```

### Q5：为什么设备在 LuCI 列表中，但 DDNS 仍找不到 IPv6？

LuCI 的设备列表和 Linux neighbor table 是两个不同的数据来源。

LuCI 的列表主要用于让你方便选择 MAC；真正查找 IPv6 时，ddns-scripts 使用：

```sh
ip -6 neigh show
```

所以最终仍必须让目标设备出现在 OpenWrt 的 IPv6 neighbor table 中，并处于本实现接受的活动状态。

### Q6：为什么 `STALE` 不使用？

`STALE` 只表示 neighbor cache 中仍保存着历史信息，并不能证明该地址当前仍然可用。

本项目选择保守处理：只有 `REACHABLE`、`DELAY`、`PROBE` 才作为当前地址候选，避免 DDNS 被更新成离线设备的旧 IPv6。

## 回滚补丁

如果需要恢复原始 OpenWrt 行为，最简单的方法是使用干净的 OpenWrt 源码重新编译。

如果补丁已经应用但尚未修改其它内容，也可以尝试：

```sh
patch -R -p1 < patches/0002-ddns-scripts-ipv6-mac.patch
patch -R -p1 < patches/0001-luci-app-ddns-ipv6-mac.patch
```

如果补丁与当前源码状态不一致，不建议强行回滚，直接重新检出干净的 `luci` / `packages` 源码更安全。

## 补丁内容概览

### luci-app-ddns

修改内容主要包括：

- 获取 `network.getHostHints()`；
- IPv6 IP source 增加 `MAC Address`；
- 增加 `ip_mac` 配置项；
- 自动列出当前已知设备的 MAC；
- 保存但暂时未知的 MAC 显示为“未知设备”；
- IPv4 不显示 MAC source；
- 从 IPv6 切换回非 IPv6 时自动清理 MAC source；
- 增加对应 LuCI RPC ACL；
- 增加简体中文翻译。

### ddns-scripts

修改内容主要包括：

- 增加 `get_ipv6_by_mac()`；
- `get_current_ip()` 支持 `ip_source=mac`；
- `dynamic_dns_lucihelper.sh` 增加 `-m MAC`；
- `ddns.config_sample` 增加 MAC source 示例。

## 最小测试流程

如果只是想快速验证本补丁是否工作，可以按下面顺序执行：

### 1. 确认目标设备 MAC

例如：

```text
AA:BB:CC:DD:EE:FF
```

### 2. 查看 neighbor table

```sh
ip -6 neigh show | grep -i 'lladdr AA:BB:CC:DD:EE:FF'
```

确保至少有一个匹配项，并且状态是：

```text
REACHABLE
DELAY
```

或：

```text
PROBE
```

### 3. 直接测试地址获取

```sh
/usr/lib/ddns/dynamic_dns_lucihelper.sh -6 -m AA:BB:CC:DD:EE:FF -- get_local_ip
```

### 4. 检查 UCI

```sh
uci show ddns | grep -E 'use_ipv6|ip_source|ip_mac'
```

确认目标服务包含：

```text
use_ipv6='1'
ip_source='mac'
ip_mac='AA:BB:CC:DD:EE:FF'
```

### 5. 运行 DDNS

完成正常 DDNS 服务商配置后，让 `ddns-scripts` 执行更新，并检查日志确认最终使用的 IPv6 地址。

## 注意事项

- 本项目针对 OpenWrt 当前 master 的源码结构开发，旧版本 OpenWrt 不保证可以直接套用。
- 补丁依赖 `luci-app-ddns`、`ddns-scripts` 当前源码结构；如果上游发生较大变化，可能需要重新适配。
- MAC 地址只是定位邻居的条件，不代表该设备一定在线。
- 只有 Linux neighbor table 中处于活动状态的 IPv6 邻居才会被本实现采用。
- 如果目标设备使用多个 IPv6 地址，当前实现无法仅凭 neighbor table 判断哪个是 RFC 4941 temporary address。
- DDNS 服务商本身的认证、域名、TTL、IPv6 支持等问题不属于本补丁处理范围。
- 使用公网 IPv6 DDNS 前，请确认防火墙规则允许你期望的入站流量；DDNS 只负责 DNS 记录更新，不会自动开放路由器或内网设备端口。

## 相关文件

```text
patches/
├── 0001-luci-app-ddns-ipv6-mac.patch
└── 0002-ddns-scripts-ipv6-mac.patch
```

核心修改位置：

```text
luci-app-ddns/
└── htdocs/luci-static/resources/view/ddns/overview.js

ddns-scripts/
├── files/usr/lib/ddns/dynamic_dns_functions.sh
├── files/usr/lib/ddns/dynamic_dns_lucihelper.sh
└── samples/ddns.config_sample
```
