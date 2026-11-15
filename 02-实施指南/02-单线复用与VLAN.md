> **维护说明（2026-11-15）**：本文由原始资料整理而来，尚未在你的现场设备上验证。执行前必须先阅读 [技术勘误与实施前核验](../00-重要说明/技术勘误与实施前核验.md)；涉及 VLAN ID、端口绑定、FTTR 业务映射、固件菜单和 UCI 命令时，以现场配置为准。文中的 VLAN 示例值（原文示例为 41）、IP 地址、账号格式、端口号及灯态均不可直接视为你的实际值。

---

# 方案 B 单线复用：VLAN 配置（只有一根网线时用）

> **前置**：如果你的弱电箱到 MT6000 之间能走两根网线，别用这个文件，回去用 `03` 的 B-简 双线方案。
> 本章适合：只有一根网线，必须让"宽带"和"内网"跑在同一根线上。

---

## 1. 搞清楚要打什么标签

| VLAN | 用途 | 标签方式 | 谁认它 |
|---|---|---|---|
| **VLAN 示例值（原文示例为 41）** | 宽带业务（PPPoE 报文） | **带 tag（tagged）** | 光猫上联光口 → 网口1 → MT6000 |
| **VLAN 1** | 家里内网（192.168.8.x） | **不带 tag（untagged）** | 光猫网口 → MT6000 LAN |

一根网线里，标签帧和普通帧可以共存 —— 这就是"单线复用"。

```
光猫网口1（trunk口）
   │
   ├── VLAN 示例值（原文示例为 41） tagged  ──────────►  MT6000: eth.wan.41  → PPPoE 拨号
   │
   └── VLAN 1 untagged ──────────►  MT6000: br-lan      → 内网
                                    （同时回灌给光猫的下行光口和网口3/4）
```

---

## 2. 光猫侧配置（V176-20）

### 2.1 桥接 WAN 连接

```
网络 → 宽带设置 → 选中 INTERNET 那条
  连接模式：桥接
  VLAN 模式：改写（Tag）        ← 必须
  VLAN ID：填写现场实际值（原文 41 仅为示例）                    ← 按你实际抄的值
  802.1p：保持原值
  LAN 端口绑定：只勾 网口1
  SSID 绑定：全部取消
  下行光口/FTTR：不勾
  DHCP 使能：取消
  NAT 使能：取消
```

### 2.2 让网口1 同时承载内网 VLAN 1

关键在 **VLAN 绑定**（不同固件叫"VLAN 绑定"/"端口VLAN"/"LAN口多业务"）：

```
网络 → VLAN 绑定 / LAN口多业务
  网口1：勾选 VLAN 示例值（原文示例为 41）（tagged）＋ VLAN 1（untagged）
  网口3、网口4：勾选 VLAN 1（untagged）
  下行光口(FTTR/PON1)：勾选 VLAN 1（untagged）
```

> **注意**：有些 V176-20 固件不允许在"LAN口多业务"里对同一个口配两个 VLAN。
> 这种情况改用 `网络 → VLAN 绑定` 页面，手工添加条目。如果两个页面都不支持 → 放弃单线复用，改双线方案。

### 2.3 光猫 LAN / DHCP

```
网络 → LAN 设置
  LAN IP：192.168.8.2（避开 MT6000 的 192.168.8.1）
  DHCP 服务器：❌ 关闭
```

---

## 3. MT6000 侧配置（OpenWrt / GL.iNet）

> GL.iNet 官方界面（4.x）对 VLAN 支持有限，**推荐用 LuCI 或 SSH 命令行**。

### 3.1 方式一：LuCI 界面

登录 `http://192.168.8.1` → 系统 → 高级设置（LuCI）→ 网络 → 接口

**① 创建 VLAN 示例值（原文示例为 41） 虚拟接口**
```
网络 → 交换机 / 设备  →  添加设备（Device）
  名称：eth1.41        （eth1 = 你的 2.5G WAN 口，用 `ip link` 确认）
  类型：802.1Q
  基设备：eth1
  VLAN ID：填写现场实际值（原文 41 仅为示例）
```

**② 修改 WAN 接口**
```
网络 → 接口 → wan → 编辑
  设备：eth1.41
  协议：PPPoE
  用户名/密码：宽带账号密码
  MTU：1492
```

**③ 把 WAN 物理口加入 br-lan（实现内网透传）**
```
网络 → 接口 → 设备 → br-lan → 配置
  网桥端口：勾上 eth1（就是那个 2.5G WAN 物理口本身）
```
> 这一步是让 untagged 的内网帧从 eth1 进出。VLAN 示例值（原文示例为 41） 的 tagged 帧由 eth1.41 单独处理，互不干扰。

### 3.2 方式二：SSH 直接改（更可靠）

```bash
# 1) SSH 进 MT6000
ssh root@192.168.8.1

# 2) 确认 WAN 物理口名字（MT6000 通常是 eth1，双 2.5G 口是 eth1/eth0）
ip link show | grep -E "eth[0-9]"
#   典型输出：eth0 = LAN侧2.5G, eth1 = WAN侧2.5G

# 3) 建 VLAN 示例值（原文示例为 41） 子接口
uci set network.wan41=device
uci set network.wan41.name='eth1.41'
uci set network.wan41.type='8021q'
uci set network.wan41.ifname='eth1'
uci set network.wan41.vid='41'
uci commit network

# 4) WAN 改为用这个设备 + PPPoE
uci set network.wan.device='eth1.41'
uci set network.wan.proto='pppoe'
uci set network.wan.username='你的宽带账号'
uci set network.wan.password='你的宽带密码'
uci set network.wan.mtu='1492'
uci set network.wan.ipv6='1'
uci commit network

# 5) 把 eth1 加进 br-lan（内网透传）
uci add_list network.@device[0].ports='eth1'
uci commit network

# 6) 重启网络
/etc/init.d/network restart
```

### 3.3 验证

```bash
# 看 VLAN 子接口是否起来
ip -d link show eth1.41
# 看 PPPoE 是否拨上
ifstatus wan | grep -i address
# 看公网 IP
curl -s https://myip.ipip.net
```

---

## 4. 常见坑（VLAN 方案专属）

| 坑 | 现象 | 解决 |
|---|---|---|
| VLAN ID 记错 | 拨号报 678/651 超时 | 回光猫原配置核对，移动多为 41，也有 100/101/3961 等 |
| VLAN 模式选了"透传" | 拨号超时 | 必须"改写(Tag)"，让光猫打上 tag |
| eth1 没加进 br-lan | 拨号成功，但光猫 Wi-Fi 和从网关全断 | 补第 5 步 |
| 光猫把下行光口绑给了 WAN | K156-20 全红 | 取消 WAN 对下行光口的绑定 |
| MT6000 双 2.5G 口搞混 | 拨不上 / 内网不通 | `ip link` 确认哪个是 WAN 侧 |
| 交换机芯片不支持 802.1Q 混合 | VLAN 示例值（原文示例为 41） 通但内网不通 | MT6000 用 DSA 架构，支持；老固件用 swconfig 的要换固件 |
| MTU 设错 | 部分网站打不开、大文件下载卡住 | PPPoE 用 1492；IPv6 用 1492 或 1480 |

---

## 5. 这个方案值不值得

**如果你能走两根线，就别用 VLAN。**

VLAN 单线复用属于"物理条件受限时的补救方案"，它的故障表现（能连 Wi-Fi 但上不了网、从网关断）非常难在现场判断。

能走双线就走双线，省下这一整章的坑。
