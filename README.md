# openwrt-ddns-mac

为 OpenWrt 当前 master 的 luci-app-ddns / ddns-scripts 增加 IPv6 按 MAC 获取目标内网设备地址的补丁。

LuCI 复用现有 `network.getHostHints()` 生成“设备名称 — MAC”列表；IPv4 源列表不包含 MAC。已保存但当前未出现在 host hints 的 MAC 会保留为“未知设备 — MAC”。

ddns-scripts 在现有 `get_current_ip()` 中增加 `ip_source=mac`，从 Linux IPv6 neighbor table 读取指定 MAC。候选仅接受 REACHABLE/DELAY/PROBE，排除 link-local、loopback、unspecified、multicast；ULA 是否允许沿用现有 `upd_privateip`。只剩 STALE 缓存时检测失败，不会直接拿离线设备的旧地址更新 DDNS。

补丁基于 2026-10-03 读取到的 openwrt/luci:master 与 openwrt/packages:master。

## 应用

```sh
patch -p1 < patches/0001-luci-app-ddns-ipv6-mac.patch
patch -p1 < patches/0002-ddns-scripts-ipv6-mac.patch
```

## 编译

```sh
make package/luci-app-ddns/compile V=s
make package/ddns-scripts/compile V=s
```

## 路由器验证

```sh
ip -6 neigh show | grep -i 'lladdr AA:BB:CC:DD:EE:FF'
/usr/lib/ddns/dynamic_dns_lucihelper.sh -6 -m AA:BB:CC:DD:EE:FF -- get_local_ip
```

配置示例：

```uci
config service 'nas'
	option enabled '1'
	option use_ipv6 '1'
	option ip_source 'mac'
	option ip_mac 'AA:BB:CC:DD:EE:FF'
```

Linux neighbor table 不提供远端 RFC 4941 临时地址标志，因此当前实现不能仅凭该表绝对判断远端 IPv6 是否为临时地址；实现避免了直接取第一条，并要求邻居处于活动状态。
