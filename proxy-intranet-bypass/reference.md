# Git 域名解析到不可达内网 IP（DNS / hosts 修复）

## 典型场景

公司因带宽压力将 Git 服务从内网迁移到公网线路，但办公网 DNS 仍将 Git 域名解析到旧的内网地址（如 `10.1.238.36`），该地址已被切断。表现为其他内网域名正常，唯独 Git 不通。

这不是代理配置问题，而是 DNS 解析到了已废弃的内网地址。

## 判断方法

```bash
nslookup git.your-company.com
```

如果返回 `10.x.x.x` 等内网地址，但公司 IT 通知应走公网 IP，则按以下优先级修复。

## 方案一（首选）：恢复 DNS 自动获取

公司通常会部署自建 DNS，通过 DHCP 自动下发，能正确将 Git 域名解析到公网地址。用户只需将系统 DNS 设置改为"自动获取"即可。

- **macOS：** 系统偏好设置 → 网络 → 选择当前连接 → DNS → 删除所有自定义 DNS 条目 → 应用。
- **Windows：** 网络适配器 → IPv4 属性 → 选择"自动获得 DNS 服务器地址" → 确定。

如果机器没有 DHCP（如机房打包机），手动配置公司自建 DNS：

```
10.130.205.51
10.130.205.55
114.114.114.114
```

> 以上为示例地址，实际以公司 IT 通知为准。

## 方案二（兜底）：手动绑定 hosts

如果 DNS 被科学上网工具自动篡改、或无法恢复自动获取，则手动绑定 hosts。

macOS / Linux：

```bash
sudo sh -c 'echo "<公网IP> git.your-company.com" >> /etc/hosts'
```

Windows（以管理员身份编辑 `C:\Windows\System32\drivers\etc\hosts`）：

```text
<公网IP> git.your-company.com
```

## 注意事项

- 如果科学上网工具会自动修改系统 DNS 服务器，即使配置了 hosts 也可能被覆盖，需要同时处理 DNS 和 hosts。
- 绑定 hosts 后如果仍不生效，检查浏览器 DNS 缓存（可用无痕模式测试）。
- 如果公网 IP 有办公网 IP 白名单限制，在家通过 OpenVPN 等连接时必须清除 hosts 中的绑定记录，否则无法访问。

## 方案三（hosts 冲突排查）：检查 `/etc/hosts` 重复条目

如果已经通过方案二绑定了公网 IP 到 hosts，但 git pull 仍然失败，可能是 `/etc/hosts` 中存在多条同域名记录：

```bash
grep git.your-company.com /etc/hosts
```

若输出类似：
```
10.1.238.36 git.your-company.com   ← 内网 IP，排在前面
121.199.40.19 git.your-company.com ← 公网 IP，排在后面
```

**macOS 系统解析器优先使用第一条匹配记录**，因此 `git.your-company.com` 实际解析到内网 IP `10.1.238.36`，公网 IP 条目被覆盖而未生效。

修复：删除内网 IP 条目，只保留公网 IP 条目：

```bash
sudo sed -i '' '/^10\.1\.238\.36 git\.your-company\.com/d' /etc/hosts
sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder
```

验证：

```bash
grep git.your-company.com /etc/hosts   # 应只剩公网 IP 条目
ping -c1 git.your-company.com          # 应连通公网 IP
git pull                               # 应正常
```

## 补充说明：TUN 模式下 SSH 与 HTTPS 的差异

开代理后网页（HTTPS 443）能访问内网 Git 服务器，但 `git pull`（SSH 22）不通，原因：

- **HTTPS 流量**：Clash TUN + DIRECT 规则能正确处理 HTTPS 连接直连，不影响连接。
- **SSH 流量**：SSH 经过 TUN 接口后，连接路径发生变化（NAT/源 IP 等），导致密钥交换阶段被远端拒绝——表现为 `kex_exchange_identification: Connection closed by remote host`。

解决方案优先级：
1. 让 Git 域名解析到公网 IP（hosts 或 DNS），SSH 走公网线路，无论有无代理都能通。
2. 在 FlClash 覆写脚本中加 `tun.route-exclude-address` 将内网 IP 绕过 TUN 接口（仅适用于需要在内网 IP 上操作的场景）。
