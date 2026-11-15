# Huawei OptiXstar FTTR 家庭组网知识库

> 面向华为 OptiXstar FTTR 主/从光网关与 GL.iNet 路由器的家庭网络改造资料库。重点研究：**保留 FTTR 从网关与光纤覆盖，同时由自有路由器承担 PPPoE 拨号、NAT、DHCP 和家庭网络策略**。

[![Knowledge Base](https://img.shields.io/badge/Knowledge%20Base-Chinese-blue)]()
[![Status](https://img.shields.io/badge/Status-持续校核-orange)]()

## 1. 项目目标

本仓库整理原始资料，并补充技术勘误、实施前检查、配置验证、回滚和运维规范。它不是运营商对某一省份、某一固件的官方施工手册；所有涉及 VLAN、端口绑定、FTTR 管理通道和运营商业务模板的操作，必须先以现场设备页面、配置备份和装维确认结果为准。

目标设备场景（以设备铭牌最终核验为准）：

- 华为 OptiXstar V176-20：FTTR 主光网关，XG-PON/非对称 10G-EPON 等具体上行制式以设备版本为准。
- 华为 OptiXstar K156-20：FTTR 从光网关，数量 3 台；请核对铭牌，避免与 V156-20 等型号混写。
- GL.iNet GL-MT6000（Flint 2）：承担 PPPoE、NAT、DHCP、DNS、防火墙及远程访问等路由职责。

## 在线图表预览

> **预览状态：待确认部署。** 仓库元数据显示 GitHub Pages 已启用（`has_pages: true`），但目前无法确认站点是否已成功部署。请先在仓库的 **Settings → Pages** 检查发布来源并完成部署；部署成功后，以下预览链接可用。

- [接线图.html（GitHub Pages 预览）](https://darveydev.github.io/Huawei-OptiXstar-Family-KnowledgeBase/06-%E5%9B%BE%E8%A1%A8%E4%B8%8E%E9%80%9F%E6%9F%A5/%E6%8E%A5%E7%BA%BF%E5%9B%BE.html) · [仓库源文件](https://github.com/darveydev/Huawei-OptiXstar-Family-KnowledgeBase/blob/main/06-%E5%9B%BE%E8%A1%A8%E4%B8%8E%E9%80%9F%E6%9F%A5/%E6%8E%A5%E7%BA%BF%E5%9B%BE.html)
- [速查卡-一页纸.html（GitHub Pages 预览）](https://darveydev.github.io/Huawei-OptiXstar-Family-KnowledgeBase/06-%E5%9B%BE%E8%A1%A8%E4%B8%8E%E9%80%9F%E6%9F%A5/%E9%80%9F%E6%9F%A5%E5%8D%A1-%E4%B8%80%E9%A1%B5%E7%BA%B8.html) · [仓库源文件](https://github.com/darveydev/Huawei-OptiXstar-Family-KnowledgeBase/blob/main/06-%E5%9B%BE%E8%A1%A8%E4%B8%8E%E9%80%9F%E6%9F%A5/%E9%80%9F%E6%9F%A5%E5%8D%A1-%E4%B8%80%E9%A1%B5%E7%BA%B8.html)

## 2. 从哪里开始

1. 阅读 [技术勘误与实施前核验](00-重要说明/技术勘误与实施前核验.md)，再填写 [现场参数采集表](00-重要说明/现场参数采集表.md)。
2. 阅读 [总体方案与决策树](01-总体方案/00-结论速览.md)。
3. 对照 [组网方案对比](01-总体方案/02-组网方案对比.md) 选择双线回灌、单线 VLAN 复用或旁路由。
4. 依次阅读 [实施指南](02-实施指南/)、[设备配置](03-设备配置/)、[远程访问](04-远程访问/)。
5. 改造前完成 [回滚手册](05-运维与排障/01-回滚手册.md) 中的备份项；改造后执行 [验收清单](05-运维与排障/02-验收测试清单.md)。

## 3. 目录结构

```text
.
├── README.md
├── CHANGELOG.md
├── LOG.md
├── 00-重要说明/
│   └── 技术勘误与实施前核验.md
├── 01-总体方案/
│   ├── 00-结论速览.md
│   ├── 01-FTTR架构与桥接原理.md
│   └── 02-组网方案对比.md
├── 02-实施指南/
│   ├── 01-主网关桥接与双线回灌.md
│   ├── 02-单线复用与VLAN.md
│   └── 03-网口与光纤接线.md
├── 03-设备配置/
│   └── 01-GL-MT6000配置清单.md
├── 04-远程访问/
│   └── 01-公网IP与DDNS.md
├── 05-运维与排障/
│   ├── 01-回滚手册.md
│   └── 02-验收测试清单.md
└── 06-图表与速查/
    ├── 接线图.html
    └── 速查卡-一页纸.html
```

## 4. 核心原则

- **先确认业务，再改配置**：PPPoE 用户名/密码、Internet VLAN ID、802.1p、WAN 业务模板、端口绑定均不能靠猜。
- **FTTR 不等于普通 AP Mesh**：主从光网关间可能存在独立管理/承载 VLAN 和运营商私有配置。主网关改桥接后，从网关是否继续可用取决于实际业务映射，不可仅凭“桥接”字样判断。
- **优先双线回灌**：若布线允许，宽带 WAN 与家庭 LAN 分开走线，通常比单线 VLAN trunk 更容易验证和回滚。
- **单线复用是条件方案**：只有在主光网关和 MT6000 两端均能明确配置对应 VLAN、tagged/untagged、PVID 与端口成员关系时才实施。
- **先本地验证再远程暴露**：默认不开放管理端口到公网。优先 Tailscale/WireGuard 私网访问。
- **配置命令不是通用模板**：GL.iNet 固件版本、OpenWrt 版本、DSA/交换芯片拓扑会影响 UCI 结构；执行前备份并核对 `uci show`。
- **任何断网风险操作必须有回滚路径**：不要在远程会话中直接改 WAN/VLAN/防火墙并同时切断管理链路。

## 5. 重要技术边界

### VLAN ID
原始资料中的 VLAN 41 只是示例值，不能视为中国移动固定值。不同地市、OLT、套餐、装维模板可能不同。以原 WAN 配置或运营商工单为准。

### MT6000 端口
MT6000 官方资料列出 1 个 2.5G WAN、1 个 2.5G WAN/LAN、4 个千兆 LAN。WAN/LAN 口可以切换角色，但实际端口映射、交换机配置与菜单名称取决于固件。不要将网口编号直接等同于 `eth0/eth1`。

### 公网 IP 与 DDNS
DDNS 只更新域名记录，不会自动穿透 CGNAT，也不会自动创建防火墙/端口转发。IPv4 和 IPv6 必须分别检测；同一时间比较同一地址族的 WAN 地址和外部探测地址。

### 安全
不建议为了保留手工桥接而盲目关闭 TR-069/运营商远程管理；这可能影响故障诊断、业务下发或设备注册。不要将 SSH、LuCI、NAS 管理端口直接暴露到公网。

## 6. 官方参考资料

- [GL.iNet GL-MT6000 官方规格](https://docs.gl-inet.cn/router/4/resources/specification/gl_mt6000_spec/)
- [GL.iNet GL-MT6000 产品页](https://www.gl-inet.com/products/gl-mt6000)
- [华为 iFTTR 产品资料（含 V176-20/K156-20 规格示例）](https://e.huawei.com/marketingcloud/pep/asset/20000001/Material/8e0d4a75a0084e329dfc661efb657370/M3T1A669N1201194072861212870/%E5%8D%8E%E4%B8%BAiFTTR%E5%85%A8%E5%85%89%E6%99%BA%E6%85%A7%E5%AE%B6%E5%BA%AD%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%88V100R025C00-%E5%BD%A9%E9%A1%B5_25H2_.pdf)

> 官方产品规格不能代替你家运营商定制固件的 WAN/VLAN/FTTR 配置核验。请在 Issues 中提供脱敏后的型号、固件版本、端口业务绑定截图和拓扑，以便针对具体版本修订。

## 7. 贡献与记录

- 每次修订更新 `LOG.md`。
- 影响配置的变更同步更新 `CHANGELOG.md`。
- 不提交宽带账号密码、超管密码、PPPoE 密码、API Token、公网 IP、序列号、MAC 地址或未经脱敏的配置备份。
- 未经现场验证的结论必须标注“待核验”，不得写成确定事实。
