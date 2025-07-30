# portopen

**portopen (`po`)** 是一款在 Linux 服务器上**快速放行端口**的命令行/交互式工具。  
支持 **IPv4/IPv6**、**Docker `DOCKER-USER` 转发链**，并通过 **iptables-persistent** 将规则持久化。  
可选用 `/etc/portopen.conf` 配置默认策略，实现“一键化、可审计、可复用”。

<p align="center">
  <img alt="portopen" src="https://img.shields.io/badge/platform-linux-black">
  <img alt="iptables" src="https://img.shields.io/badge/iptables-nft%20&%20legacy-blue">
  <img alt="license" src="https://img.shields.io/badge/license-MIT-green">
</p>

---

## 目录
- [特性](#特性)
- [环境要求](#环境要求)
- [快速安装](#快速安装)
- [快速开始](#快速开始)
- [命令用法](#命令用法)
- [交互式菜单](#交互式菜单)
- [配置文件](#配置文件)
- [常用示例](#常用示例)
- [工作机制说明](#工作机制说明)
- [安全建议](#安全建议)
- [FAQ / 故障排查](#faq--故障排查)
- [卸载](#卸载)
- [许可](#许可)

---

## 特性
- **一键放行**：按你的默认策略（协议/来源/IPv6/Docker）放行常用端口，可选自动扫描“监听中的常见端口”。
- **IPv4/IPv6 支持**：可同步在 `ip6tables` 写规则（可全局开启或临时开启）。
- **Docker 友好**：可在 `DOCKER-USER` 链添加对应放行规则，适配严格的转发策略。
- **持久化**：使用 `iptables-persistent`/`netfilter-persistent` 保存规则，重启仍生效。
- **交互 + CLI**：既能菜单操作，也支持脚本化/自动化命令行参数。
- **可配置**：`/etc/portopen.conf` 定义默认行为，适合多机一致化、可审计。

---

## 环境要求
- 发行版：Debian/Ubuntu（其他基于 iptables 的系统通常也可用）
- 权限：需要 `root`（请使用 `sudo`）
- 依赖：`iptables`、`ip6tables`（可选）、`netfilter-persistent`（脚本会自动安装）

> 说明：现代 Ubuntu/Debian 默认使用 **iptables-nft** 后端，脚本同样适用。

---

## 快速安装

bash -c "$(wget -qO- https://raw.githubusercontent.com/octoer/portopen/main/install)"
# 或（偏好 curl 的用户）
bash -c "$(curl -fsSL https://raw.githubusercontent.com/octoer/portopen/main/install)"

**一条命令安装：**
```bash
sudo wget -O /usr/local/bin/portopen https://raw.githubusercontent.com/octoer/portopen/main/portopen \
  && sudo chmod +x /usr/local/bin/portopen \
  && sudo ln -sf /usr/local/bin/portopen /usr/local/bin/po
```

**初始化配置（可选，推荐）：**
```bash
sudo portopen init
# 或
sudo po init
```

---

## 快速开始

```bash
# 一键按配置放行（默认放行 QUICK_PORTS + 可选扫描监听端口）
sudo po quick

# 打开交互式菜单
sudo po
```

未创建配置文件时，脚本带有安全的**内置默认值**。你也可以先 `po init` 生成并编辑 `/etc/portopen.conf`。

---

## 命令用法

```text
portopen init                  # 初始化/编辑配置文件 /etc/portopen.conf
portopen quick                 # 按配置一键放行
portopen add <ports>   [--tcp|--udp|--both] [--source <CIDR>] [--ipv6] [--docker]
portopen remove <ports>[--tcp|--udp|--both] [--source <CIDR>] [--ipv6] [--docker]
portopen list                  # 查看当前规则（INPUT & DOCKER-USER）
portopen reload                # 重新加载配置文件
portopen menu                  # 打开交互菜单（默认）
```

- `<ports>` 支持**逗号或空格分隔**：`443,24981` 或 `"443 24981"`  
- 未指定 `--tcp/--udp/--both` 时，使用配置中的 `DEFAULT_PROTOCOL`（默认 `both`）  
- 未指定 `--source` 时，使用 `DEFAULT_SOURCE`（默认 `0.0.0.0/0`）  
- `--ipv6` / `--docker` 可临时覆盖配置文件中的开关  
- **优先级**：命令行参数 **>** 配置文件

---

## 交互式菜单

执行：
```bash
sudo po
```

菜单包含：
1. 初始化/编辑配置文件  
2. 一键放行  
3. 放行端口（支持 TCP/UDP/IPv6/Docker）  
4. 移除端口（支持 TCP/UDP/IPv6/Docker）  
5. 查看当前规则  
6. 重新加载配置文件  

---

## 配置文件

路径：`/etc/portopen.conf`（通过 `sudo po init` 生成）

示例：
```ini
# 默认协议：both|tcp|udp
DEFAULT_PROTOCOL="both"

# 默认来源（CIDR）：初期可 0.0.0.0/0，稳定后建议收紧
DEFAULT_SOURCE="0.0.0.0/0"

# 一键是否同时写入 IPv6 / Docker 转发规则（DOCKER-USER）
ENABLE_IPV6="no"
ENABLE_DOCKER="no"

# 一键放行的固定端口（逗号或空格分隔）
QUICK_PORTS="443 24981"

# 一键时是否扫描本机监听端口（只会采纳白名单中的端口）
SCAN_LISTEN="yes"

# 监听扫描白名单（防止误开过多）
PORT_WHITELIST="443 8443 4443 2096 2095 2087 2083 2053 30000 31698 24981"
```

> **为什么要配置文件？**  
> 固化你的默认策略，实现一键化、可审计、可复用；在多机/多人协作中保证一致性。  
> 修改后执行 `sudo po reload` 让新默认参与后续命令（不会自动回滚既有规则）。

---

## 常用示例

```bash
# 放行 443 和 24981（使用默认协议/来源）
sudo po add "443 24981"

# 仅放行 UDP 443，且同时写 IPv6 和 Docker 转发
sudo po add 443 --udp --ipv6 --docker

# 只允许固定来源访问 443/TCP（更安全）
sudo po add 443 --tcp --source 1.2.3.4/32

# 移除端口（both）
sudo po remove 24981 --both

# 查看当前规则
sudo po list
```

---

## 工作机制说明
- **链 & 顺序**：脚本在 `INPUT` 链新增 `ACCEPT` 规则，并**插入在兜底 REJECT 前**（若存在）。  
- **IPv6**：启用时在 `ip6tables` 同步写入 `INPUT` 链。  
- **Docker**：启用时会在 `DOCKER-USER` 链加入 `ACCEPT`（用于严格转发策略环境）；大多数 DNAT 场景仅放开宿主 `INPUT` 即可。  
- **持久化**：调用 `netfilter-persistent save`（或 `iptables-save`/`ip6tables-save`）确保重启后仍生效。

---

## 安全建议
- **最小暴露面**：仅放行**必要端口**，其余保留兜底 REJECT。  
- **限定来源**：有固定办公/家宽出口 IP 时，优先使用 `--source <CIDR>` 收紧来源。  
- **变更留痕**：把 `portopen.conf` 纳入版本管理，便于审计与回滚。  
- **先验证后收紧**：初期可放行 0.0.0.0/0，确认业务稳定后逐步改为精确来源。  
- **Docker 环境**：如自定义了严格的 FORWARD 政策，再启用 `--docker` 放行转发。

---

## FAQ / 故障排查

**Q: 执行成功但客户端仍连不上？**  
A: 逐步排查：  
1. `sudo ss -tulpen | grep -E '(<port>)'` 确认服务在监听正确 IP/端口；  
2. `sudo iptables -L INPUT -n --line-numbers` 确认 `ACCEPT` 规则在 REJECT 之前；  
3. 对 UDP 协议，使用 `sudo tcpdump -ni any udp port <port> -vv` 看是否有入站包；  
4. 云厂商（如 Oracle Cloud/NSG/安全列表）是否同样放行了目标端口与协议。

**Q: 使用的是 iptables-legacy，会有影响吗？**  
A: 脚本使用 `iptables`/`ip6tables` 命令，与 `nft`/`legacy` 后端均可兼容（由系统默认管理）。

**Q: 一键放行打开了不需要的端口？**  
A: 配置中将 `SCAN_LISTEN="no"`，并收紧 `PORT_WHITELIST` 与 `QUICK_PORTS`。

---

## 卸载
```bash
# 删除脚本与快捷方式
sudo rm -f /usr/local/bin/portopen /usr/local/bin/po

# （可选）删除配置文件
sudo rm -f /etc/portopen.conf

# 注意：这不会自动清除现有 iptables 规则。
# 如需回退，请使用 `po remove ...` 逐条移除并保存持久化。
```

---

## 许可
本项目采用 **MIT License**。详见 [LICENSE](./LICENSE)。

---

## 致谢
- 感谢社区对 Linux 网络与容器安全最佳实践的探索与贡献。

---

### Star & Issues
如果这个工具对你有帮助，欢迎点个 ⭐。遇到问题或有功能建议，欢迎提交 Issue！
