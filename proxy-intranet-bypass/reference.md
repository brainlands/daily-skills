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
