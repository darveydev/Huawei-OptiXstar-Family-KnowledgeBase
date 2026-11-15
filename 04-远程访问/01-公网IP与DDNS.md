> **维护说明（2026-11-15）**：本文由原始资料整理而来，尚未在你的现场设备上验证。执行前必须先阅读 [技术勘误与实施前核验](../00-重要说明/技术勘误与实施前核验.md)；涉及 VLAN ID、端口绑定、FTTR 业务映射、固件菜单和 UCI 命令时，以现场配置为准。文中的 VLAN 示例值（原文示例为 41）、IP 地址、账号格式、端口号及灯态均不可直接视为你的实际值。

---

# 公网 IP + DDNS：把家庭宽带做成网络出口

> 本章回答你的第二个问题："向移动申请公网IP后如果得到动态IP，DDNS之后可以怎么操作使用家庭宽带作为网络出口？"

---

## 1. 先说残酷现实：移动给公网 IPv4 的概率很低

| 运营商 | 家庭宽带公网 IPv4 政策 | 申请难度 |
|---|---|---|
| 中国电信 | 多数地区可申请，部分免费，部分要工单 | ★★☆☆☆ |
| 中国联通 | 北方较好申请，南方较难 | ★★★☆☆ |
| **中国移动** | **95%+ 区域强制 CGNAT**，家庭宽带基本不给公网 IPv4 | **★★★★★（常被拒）** |

据多方实测反馈：移动家庭宽带申请公网 IPv4，**多数被拒**，理由通常是"家庭宽带不提供公网 IP，需要办理专线/企业宽带"。部分地市开放了付费开通（如北京移动营业厅 20 元/月）。

### 你的实际选项（按性价比排序）

```
① 先确认你已经是公网 IP（有些人装机时就给了）  ← 免费，先查
② 申请公网 IPv4（打电话/掌厅/工信部投诉）      ← 可能白费功夫
③ 走 IPv6 + DDNS                              ← 移动 IPv6 覆盖好，免费，推荐
④ 内网穿透 / 异地组网（Tailscale、ZeroTier）    ← 100% 可行，零依赖运营商
```

> **我的建议：② 试一次（成本一通电话），同时 ③ 直接做（IPv6 现在很成熟）。真的急用就 ④。**

---

## 2. 第一步：判断你现在到底是不是公网 IP

### 2.1 标准三步法

```
步骤1：登录光猫/路由器后台，看 WAN 口拿到的 IP 地址
步骤2：手机断开 Wi-Fi，用 4G/5G 流量访问 https://myip.ipip.net，记下出口 IP
步骤3：比对两个 IP
```

| 结果 | 判定 |
|---|---|
| 完全一致，且是 1.x~223.x 的公网段（排除下面那些） | ✅ 真公网动态 IP |
| WAN IP 是 `10.x.x.x` | ❌ CGNAT（运营商内网） |
| WAN IP 是 `100.64.x.x ~ 100.127.x.x` | ❌ **CGNAT 专用段**，最典型 |
| WAN IP 是 `172.16.x.x ~ 172.31.x.x` | ❌ 内网 |
| WAN IP 是 `192.168.x.x` | ❌ 三重 NAT |
| WAN IP ≠ 出口 IP，但两个都是公网段 | ⚠️ 池化/双栈，端口映射大概率失效 |

### 2.2 快速命令（在接 MT6000 的电脑上跑）

```bash
# 看本机出口 IP（手机用同样网址对照）
curl -s https://myip.ipip.net

# 看路由跳数（CGNAT 第一跳通常是 100.64.x）
# Windows：
tracert -d -h 5 223.5.5.5
# Linux/Mac：
traceroute -n -m 5 223.5.5.5

# 如果在用 Tailscale/proxy，先关掉！否则测到的是它们
```

> ⚠️ **实测提醒**：本机查询时发现 Tailscale 在跑，走代理时出口显示为台湾中华电信，直连才是 112.38.217.15（山东移动）。**测试务必先退掉所有代理和 VPN。**

### 2.3 最权威的判定：从外网连回来

```bash
# 在家里的 MT6000 上开个临时 web 服务
# 然后用手机流量（不是 Wi-Fi）访问 http://<你的WAN_IP>:8080
```
能连上 = 真公网且可入站；连不上 = CGNAT 或被墙。

---

## 3. 第二步：向移动申请公网 IP（话术与路径）

### 3.1 三个渠道（按成功率排序）

**渠道 A：10086 转宽带报修（成功率最高）**
> 话术：
> **"我家装了监控/NAS，需要在外面远程访问。现在宽带拿到的是 100.64.x.x 的内网地址，做不了端口映射。麻烦帮我申请一个动态公网 IPv4，不需要固定 IP，动态的就行。如果不方便开通，麻烦帮我转宽带技术部门确认一下政策。"**

要点：
- ✅ 强调"**动态的就行，不要固定IP**"（降低对方心理门槛）
- ✅ 给一个"正当用途"（监控 / NAS / 远程办公）
- ✅ 要求**转宽带技术部门**，一级客服经常不懂
- ❌ 不要说"我要做服务器/做代理/做网站"（会被拒）

**渠道 B：中国移动 App / 掌上营业厅**
```
我的 → 宽带服务 → 更多 → 公网IP开通（部分省份有这个入口）
或：服务 → 办理 → 宽带 → 公网IP
```
> 部分地区收费（10~20 元/月），部分地区免费。有入口就点，没入口就走渠道 A。

**渠道 C：工信部投诉（最后手段）**
```
https://dxss.miit.gov.cn/  → 电信用户申诉
```
> 只有在你确认"同城同套餐的其他用户可以办"的情况下才用。否则可能被判定无理投诉。

### 3.2 申请成功后要做的三件事

1. **重启光猫 + MT6000**（重新拨号才会拿到新 IP 池的地址）
2. 重新查 WAN IP，确认变成公网段
3. 用外网设备测一次入站（2.3 节的方法）

### 3.3 如果被拒绝

不要纠缠，直接转 **IPv6 方案（第 5 章）** 或 **异地组网（第 6 章）**。这两条路不依赖运营商。

---

## 4. 第三步：DDNS（动态域名）

拿到动态公网 IP 后，IP 会变。DDNS 做的事就是：**把域名自动指向你当前的 IP**。

### 4.1 DDNS 工作原理

```
┌──────────┐   ① 每小时查询我的公网IP
│ MT6000   │──────────────────────────► api.ipify.org
│ DDNS客户端│◄────────────────────────── 112.38.217.15
└────┬─────┘
     │ ② 发现 IP 变了：调用 DNS 服务商 API 更新 A 记录
     ▼
┌──────────┐
│  DNS 服务商│   home.example.com  A  112.38.217.15
│ (Cloudflare)│
└────┬─────┘
     │ ③ 外部访问
     ▼
┌──────────┐
│ 你在外面  │  ssh user@home.example.com  ← 永远有效
│ 的手机/电脑│
└──────────┘
```

### 4.2 DDNS 服务商选型

| 服务 | 免费 | 支持 API | 适合 | 备注 |
|---|---|---|---|---|
| **Cloudflare** | ✅ | ✅ 最完善 | **首选** | 需要有自己的域名；API Token 权限可控 |
| **阿里云 DNS** | ✅ | ✅ | 国内访问快 | 需要域名；解析生效快 |
| **DNSPod（腾讯）** | ✅ | ✅ | 国内 | 免费版够用 |
| **DuckDNS** | ✅ | ✅ 极简 | 零门槛 | 只有 `xxx.duckdns.org` 子域 |
| **No-IP** | ✅（需每月确认） | ✅ | 老牌 | 免费版要手动续期，麻烦 |
| **花生壳** | 部分 | 有限 | 国内老牌 | 免费版限流量/带宽 |
| **GL.iNet DDNS** | ✅ | 内置 | **MT6000 自带** | 给你 `xxx.glddns.com`，零配置 |

> **推荐组合**：有域名 → **Cloudflare**（最灵活，配合后续反代/证书）；没域名想快速用 → 先用 MT6000 自带的 **GL.iNet DDNS** 试通，再升级。

### 4.3 方案一：MT6000 自带 DDNS（5 分钟搞定）

```
MT6000 后台 → 应用程序 / Applications → DDNS
  ├─ 启用
  ├─ 服务商：GL.iNet（默认）
  ├─ 主机名：dawayhome（自动生成 dawayhome.glddns.com）
  ├─ 或选 Cloudflare / 阿里云 填 API Token
  └─ 应用
```

验证：
```bash
nslookup dawayhome.glddns.com
# 应该返回你的公网 IP
```

### 4.4 方案二：Cloudflare（推荐，可控性最好）

**① 准备**
- 一个域名（阿里云/腾讯云/Namecheap 买，几十块/年）
- 把域名 NS 托管到 Cloudflare（免费）

**② 创建 API Token（不要用 Global API Key）**
```
Cloudflare 后台 → 右上头像 → My Profile → API Tokens → Create Token
  模板：Edit zone DNS
  Permissions: Zone - DNS - Edit
  Zone Resources: Include - Specific zone - 你的域名
  过期：不设或 1 年
→ 复制 Token（只显示一次）
```

**③ 在 MT6000 上配置**

GL.iNet 界面：
```
应用程序 → DDNS → 添加
  服务商：Cloudflare
  域名：home.你的域名.com
  Zone ID：（Cloudflare 域名概览页右下角）
  API Token：<粘贴>
  IP 版本：IPv4（或 IPv6）
```

或用 SSH + ddclient：
```bash
opkg update
opkg install ddclient
vi /etc/config/ddclient
```
配置示例：
```
config ddns 'global'
    option ddns_dateformat '%F %R'
    option ddns_loglines '250'
    option updatetime '10'
    option use 'web'
    option web 'https://api.ipify.org'

config service 'cloudflare'
    option enabled '1'
    option service 'cloudflare'
    option zone '你的域名.com'
    option host 'home.你的域名.com'
    option password '你的APIToken'
    option ttl '120'
    option proxied '0'        # ⚠️ 必须 0（灰云），否则拿不到真实IP
```

**④ 关键：Cloudflare 必须关掉代理（灰云）**

```
Cloudflare → DNS → 那条 A 记录 → 代理状态：DNS only（灰色云朵）
```
> 🟠 橙云 = 流量走 Cloudflare 中转，你的真实 IP 被隐藏，但**非标端口（如 SSH、非 443）无法直连**，且 Cloudflare 免费版只代理 80/443。
> 需要 SSH / 任意端口 → **必须灰云**。

### 4.5 验证 DDNS 生效

```bash
# 1) 本地查
nslookup home.你的域名.com
# 2) 外网查（用手机流量，或在外面用 https://dnschecker.org）
# 3) 真正测试：从外网 SSH 回来
ssh user@home.你的域名.com
```

---

## 5. 第四步：让外部流量真正进来（端口映射）

DDNS 只解决"找到你"，还需要"放进来"。

### 5.1 在 MT6000 上做端口转发

```
MT6000 后台 → 网络 → 防火墙 → 端口转发 → 添加
  名称：NAS-Web
  协议：TCP
  外部端口：8443
  内部 IP：192.168.8.100（你的NAS）
  内部端口：443
```

命令行方式（OpenWrt）：
```bash
uci add firewall redirect
uci set firewall.@redirect[-1].name='NAS-Web'
uci set firewall.@redirect[-1].target='DNAT'
uci set firewall.@redirect[-1].src='wan'
uci set firewall.@redirect[-1].dest='lan'
uci set firewall.@redirect[-1].dest_ip='192.168.8.100'
uci set firewall.@redirect[-1].dest_port='443'
uci set firewall.@redirect[-1].src_dport='8443'
uci set firewall.@redirect[-1].proto='tcp'
uci commit firewall
/etc/init.d/firewall restart
```

### 5.2 常见端口与建议

| 用途 | 默认端口 | 建议外部端口 | 备注 |
|---|---|---|---|
| SSH | 22 | **改成 2222+** | 22 是最常被扫描的 |
| Web (HTTP) | 80 | **不要开** | 移动家宽 80/443 常被封 |
| HTTPS | 443 | **8443 / 9443** | 家宽 80/443 多数被封 |
| WireGuard | 51820/UDP | 保持 | 只开 UDP 一个口最安全 |
| OpenVPN | 1194/UDP | 保持 | |
| RDP | 3389 | **绝对不要开** | 勒索病毒头号入口 |
| NAS (群晖) | 5000/5001 | 改掉 | |

> ⚠️ **中国移动家庭宽带常封 80/443 入站**。这不是你的配置问题，只能换端口或用反代。

### 5.3 ⚠️ 关于"IPv4 入站被封"的现实

即使拿到公网 IPv4，移动仍可能：
- 封 80/443（最常见）
- 封全部低端口
- 检测到"服务器行为"后限速/断网（违反家宽协议）

**规避方式**：全用高位端口（10000+），走非标协议，或者干脆用 IPv6。

---

## 6. IPv6 方案（移动用户的最佳选择）

### 6.1 为什么推荐

- 移动 IPv6 覆盖好（多数地区双栈已开）
- **IPv6 没有 NAT**，每个设备可以直接获得公网地址
- **不需要申请**，不依赖客服
- 端口不受 80/443 封禁影响（多数地区）

### 6.2 前提条件

| 条件 | 怎么确认 |
|---|---|
| 宽带支持 IPv6 | 光猫后台看 WAN 是否有 IPv6 地址 |
| 光猫开启 IPv6 | 桥接 WAN 的 IP 模式选 **IPv4&IPv6** |
| MT6000 开启 IPv6 | 网络 → 接口 → WAN → IPv6：DHCPv6 客户端 |
| MT6000 下发 IPv6 前缀 | LAN → IPv6：RA + DHCPv6，前缀委派（PD） |
| MT6000 防火墙放行 | 默认 IPv6 入站是**拒绝**的，要手动放行 |

### 6.3 配置步骤（MT6000 / OpenWrt）

**① WAN 侧**
```bash
uci set network.wan.ipv6='1'
uci set network.wan6=interface
uci set network.wan6.device='@wan'
uci set network.wan6.proto='dhcpv6'
uci set network.wan6.reqaddress='try'
uci set network.wan6.reqprefix='auto'
uci commit network
```

**② LAN 侧**
```bash
uci set network.lan.ip6assign='64'
uci set dhcp.lan.ra='server'
uci set dhcp.lan.dhcpv6='server'
uci set dhcp.lan.ra_management='1'
uci commit
/etc/init.d/network restart && /etc/init.d/odhcpd restart
```

**③ 放行 IPv6 入站（只放你要的端口）**
```bash
uci add firewall rule
uci set firewall.@rule[-1].name='Allow-WG-IPv6'
uci set firewall.@rule[-1].src='wan'
uci set firewall.@rule[-1].proto='udp'
uci set firewall.@rule[-1].dest_port='51820'
uci set firewall.@rule[-1].target='ACCEPT'
uci set firewall.@rule[-1].family='ipv6'
uci commit firewall
/etc/init.d/firewall restart
```

**④ 验证**
```bash
# 在 MT6000 上看拿到的 IPv6
ip -6 addr show | grep "scope global"
# 从外网（手机流量）测试
ping6 <你的IPv6地址>
# 在线检测
https://test-ipv6.com
```

### 6.4 IPv6 DDNS

IPv6 地址变化比 IPv4 更频繁（前缀会变），DDNS 更重要。

- **Cloudflare**：支持 AAAA 记录，ddclient 可配置 `usev6=webv6`
- **DuckDNS**：支持 `ipv6=` 参数
- **MT6000 自带 DDNS**：选 IPv6 模式

```bash
# ddclient 配置示例（IPv6）
config service 'cloudflare_v6'
    option enabled '1'
    option service 'cloudflare'
    option zone '你的域名.com'
    option host 'home6.你的域名.com'
    option password 'APIToken'
    option usev6 'webv6'
    option webv6 'https://api6.ipify.org'
    option proxied '0'
```

### 6.5 IPv6 的坑

| 坑 | 说明 |
|---|---|
| 移动 IPv6 前缀会定期变 | 每次变 DDNS 要更新，用短 TTL（120s） |
| 部分移动地区 IPv6 入站被封 | 实测确认，封了就只能回退 IPv4/穿透 |
| 公司/学校网络常常没有 IPv6 | 你在外面的网络也要有 IPv6 才能直连 |
| 手机 4G/5G 一般有 IPv6 | 所以手机访问家里通常没问题 |
| IPv6 地址长且难记 | **必须用 DDNS 域名**，别记 IP |

---

## 7. 兜底方案：不依赖运营商的"网络出口"

如果公网 IP 申请失败、IPv6 入站被封，用这些方式实现"从外面访问家里"：

### 7.1 Tailscale（最推荐，你已经在用）

**原理**：P2P 打洞 + DERP 中继，不需要公网 IP，不需要端口映射。

```bash
# MT6000 上装 Tailscale
opkg update && opkg install tailscale
tailscale up --advertise-routes=192.168.8.0/24 --advertise-exit-node

# 手机上装 Tailscale App，登录同一账号
# 然后在 Tailscale 后台批准 subnet routes 和 exit node
```

**能力**：
- ✅ 从任何地方访问家里所有设备（`192.168.8.x`）
- ✅ 把家里当 **exit node**（出口节点）—— 家里宽带变成你的出口
- ✅ 零配置、免费、自动 NAT 穿透
- ❌ 速度受中继影响（打洞成功时接近满速）

> **"家庭宽带作为网络出口"这个需求，Tailscale exit node 是最省事的实现方式，而且不需要公网 IP。**
> 你本机已经在跑 Tailscale（`fd7a:115c:a1e0::` 段），直接复用。

### 7.2 WireGuard 自建（公网IP/IPv6 到手后）

拿到公网 IPv4 或可用 IPv6 后，自建 WireGuard 是最快的方案：

```bash
# MT6000 上
opkg install wireguard-tools kmod-wireguard luci-proto-wireguard

# 生成密钥
wg genkey | tee /etc/wireguard/privatekey | wg pubkey > /etc/wireguard/publickey
chmod 600 /etc/wireguard/privatekey

# 服务端配置 /etc/config/network 加 interface 'wg0'
```
```bash
uci set network.wg0=interface
uci set network.wg0.proto='wireguard'
uci set network.wg0.private_key='<私钥>'
uci set network.wg0.listen_port='51820'
uci add_list network.wg0.addresses='10.10.0.1/24'

uci add network wireguard_wg0
uci set network.@wireguard_wg0[-1].description='phone'
uci set network.@wireguard_wg0[-1].public_key='<手机公钥>'
uci add_list network.@wireguard_wg0[-1].allowed_ips='10.10.0.2/32'
uci commit network

# 防火墙放行
uci add firewall rule
uci set firewall.@rule[-1].name='Allow-WG'
uci set firewall.@rule[-1].src='wan'
uci set firewall.@rule[-1].dest_port='51820'
uci set firewall.@rule[-1].proto='udp'
uci set firewall.@rule[-1].target='ACCEPT'
uci commit firewall
/etc/init.d/firewall restart
```

**开启"家里当出口"（full tunnel）**：
在 WireGuard 客户端配置里把 `AllowedIPs = 0.0.0.0/0, ::/0`，并启用 `PersistentKeepalive = 25`。
再在 MT6000 上开 NAT：
```bash
uci add firewall zone   # 或直接在 wan 上对 wg0 做 masquerade
uci set firewall.@zone[1].masq='1'
uci commit firewall
```

### 7.3 三者对比

| 方案 | 需要公网IP | 需要开端口 | 速度 | 配置难度 |
|---|---|---|---|---|
| **Tailscale** | ❌ 不需要 | ❌ 不需要 | 好（打洞成功时） | ★☆☆☆☆ |
| **WireGuard 自建** | ✅ 需要 | ✅ 需要 | **最好（满速）** | ★★★☆☆ |
| **DDNS + 端口映射** | ✅ 需要 | ✅ 需要 | 满速（但只暴露单服务） | ★★☆☆☆ |
| **Cloudflare Tunnel** | ❌ 不需要 | ❌ 不需要 | 中（走CF中转） | ★★☆☆☆ |

> **Cloudflare Tunnel** 补充：你已经在 `S:\server` 上跑过 cloudflared 隧道。同样的东西可以跑在 MT6000 上，把家里服务通过 CF 暴露成 `https://home.你的域名.com`，零端口映射、零公网 IP。缺点是不能当"整个网络的出口"，只能暴露 HTTP 服务。

---

## 8. 安全加固（暴露到公网前必做）

一旦开了端口，就有人会扫。**这节不要跳过。**

### 8.1 必做清单

- [ ] **改掉所有默认端口**（22→2222，3389→不开，5000→15000+）
- [ ] **禁用 root/password 登录**，只允许密钥
- [ ] **fail2ban** 或 MT6000 的 SSH 防爆破（限制尝试次数）
- [ ] **强密码 + 2FA**（所有暴露的管理后台）
- [ ] **关闭 UPnP**（MT6000 默认可能开着，恶意软件可自开端口）
- [ ] **不用 Windows 远程桌面直连**（3389 是最危险端口）
- [ ] **定期看 MT6000 的连接日志**

### 8.2 SSH 加固示例

```bash
# MT6000 上
uci set dropbear.@dropbear[0].Port='2222'
uci set dropbear.@dropbear[0].PasswordAuth='off'
uci set dropbear.@dropbear[0].RootPasswordAuth='off'
uci commit dropbear
/etc/init.d/dropbear restart

# 先把公钥放好再关密码登录，否则锁死自己
mkdir -p /etc/dropbear
vi /etc/dropbear/authorized_keys   # 粘贴你的公钥
```

### 8.3 一个重要的认知

> **"家庭宽带作为网络出口" ≠ "必须在公网上裸奔"。**
> 最安全的架构是：只暴露一个 **WireGuard 端口**（UDP 51820），其他所有服务都在 VPN 里面访问。
> 这样攻击面只有一个 UDP 端口，而且 WireGuard 对未认证包**完全静默**（不回包），扫描器根本发现不了。

---

## 9. 完整操作顺序（时间轴）

```
Day 0   ┌─ 确认当前是否 CGNAT（第2章）
        └─ 打 10086 申请公网 IPv4（第3章）

Day 1   ├─ 若拿到公网 IP：
        │    ├─ 重启光猫+路由器，确认 WAN IP 变公网段
        │    ├─ 配 DDNS（第4章）
        │    ├─ 配端口映射（第5章）
        │    └─ 外网实测
        └─ 若被拒：
             ├─ 立即做 IPv6（第6章）—— 半天搞定
             └─ 同时保留 Tailscale 作兜底（第7章）

Day 2   ├─ 安全加固（第8章）
        └─ 记录最终配置到本文档附录

之后    └─ 每月检查一次：IP 是否还在变、DDNS 是否正常、日志有无异常扫描
```
