---
name: proxy-intranet-bypass
description: "Configure proxy clients so company intranet, private domains, Git servers, internal docs, OA, APIs, and RFC1918 LAN traffic go DIRECT instead of through proxy nodes. Use when users report opening Clash, FlClash, Clash Verge, Clash Verge Rev, Shadowrocket, Quantumult X, V2RayN, or similar tools makes intranet unavailable; keywords include 开代理访问不了内网, 代理内网, 内网直连, bypass, DIRECT, fake-ip-filter, nameserver-policy, rules, TUN, fake-ip, 28.x.x.x, 198.18.x.x."
---

# 代理内网直通配置助手

帮助用户解决“开代理后访问不了公司内网”的问题。目标是让内网域名、内网 IP、公司 Git/文档/OA/API 直连，不走代理节点。

## Core Pattern

优先推荐“覆写/Override”方案，避免订阅更新后丢配置。只有在用户明确要直接改配置文件时，才指导修改 profile 源文件。

核心只处理三类配置：

| 配置 | 目的 |
| --- | --- |
| `rules` / 分流规则 | 把内网域名和私网 IP 放在最前面并走 `DIRECT` |
| `fake-ip-filter` | 避免内网域名被分配 `28.x.x.x` 或 `198.18.x.x` 假 IP |
| `nameserver-policy` / Host | 让内网域名使用可解析到真实地址的 DNS |

## Workflow

1. 先收集缺失信息：代理软件、内网域名或域名后缀、是否使用 TUN/fake-ip、操作系统。不要询问用户已经给出的信息。
2. 如果用户只说“内网打不开”，先给通用 DIRECT 规则和排查命令；如果给出具体软件，输出对应软件配置。
3. 对 Clash 系工具优先给覆写方案；提醒用户规则必须放在现有规则前面。
4. 域名规则用后缀时去掉开头的点，例如 `git.example.com` 可用 `DOMAIN-SUFFIX,example.com,DIRECT`；具体主机名可用 `DOMAIN,git.example.com,DIRECT`。
5. 私网 IP 规则默认包含：
   - `10.0.0.0/8`
   - `172.16.0.0/12`
   - `192.168.0.0/16`
6. 让用户修改前备份配置，修改后保存、启用覆写并重启代理客户端。
7. 最后给验证命令和判断标准。

## Clash / FlClash / Clash Verge

### 通用 YAML 模板

把规则放到 `rules` 最前面：

```yaml
rules:
  - DOMAIN-SUFFIX,your-company.com,DIRECT
  - DOMAIN,git.your-company.com,DIRECT
  - IP-CIDR,10.0.0.0/8,DIRECT,no-resolve
  - IP-CIDR,172.16.0.0/12,DIRECT,no-resolve
  - IP-CIDR,192.168.0.0/16,DIRECT,no-resolve
```

如果启用了 fake-ip，追加：

```yaml
dns:
  fake-ip-filter:
    - "+.your-company.com"
    - "git.your-company.com"
  nameserver-policy:
    "+.your-company.com":
      - 223.5.5.5
      - 119.29.29.29
```

### FlClash 覆写脚本

路径：配置 -> 当前 profile -> 更多 -> 覆写 -> 脚本模式。

```javascript
function main(config) {
  config.rules = config.rules || []
  config.rules.unshift(
    'DOMAIN-SUFFIX,your-company.com,DIRECT',
    'DOMAIN,git.your-company.com,DIRECT',
    'IP-CIDR,10.0.0.0/8,DIRECT,no-resolve',
    'IP-CIDR,172.16.0.0/12,DIRECT,no-resolve',
    'IP-CIDR,192.168.0.0/16,DIRECT,no-resolve'
  )

  config.dns = config.dns || {}
  config.dns['fake-ip-filter'] = config.dns['fake-ip-filter'] || []
  config.dns['fake-ip-filter'].push(
    '+.your-company.com',
    'git.your-company.com'
  )

  config.dns['nameserver-policy'] = config.dns['nameserver-policy'] || {}
  config.dns['nameserver-policy']['+.your-company.com'] = ['223.5.5.5', '119.29.29.29']

  return config
}
```

FlClash 使用 `function main(config)`，不要写 `module.exports`。

### Clash Verge / Clash Verge Rev 覆写

标准模式中添加：

```yaml
prepend-rules:
  - DOMAIN-SUFFIX,your-company.com,DIRECT
  - DOMAIN,git.your-company.com,DIRECT
  - IP-CIDR,10.0.0.0/8,DIRECT,no-resolve
  - IP-CIDR,172.16.0.0/12,DIRECT,no-resolve
  - IP-CIDR,192.168.0.0/16,DIRECT,no-resolve

fake-ip-filter:
  - "+.your-company.com"
  - "git.your-company.com"
```

### 直接修改 profile 文件

只在用户不能使用覆写时建议直接修改。提醒用户找 `profiles/` 下的订阅源文件，不要改运行时 `config.yaml`。

常见路径：

| 软件 | macOS 路径 |
| --- | --- |
| FlClash | `~/Library/Application Support/com.follow.clash/profiles/*.yaml` |
| Clash Verge | `~/.config/clash-verge/profiles/*.yaml` |
| Clash Verge Rev | `~/.config/clash-verge-rev/profiles/*.yaml` |

## Shadowrocket

在 `[Rule]` 最前面加：

```text
DOMAIN-SUFFIX,your-company.com,DIRECT
DOMAIN,git.your-company.com,DIRECT
IP-CIDR,10.0.0.0/8,DIRECT
IP-CIDR,172.16.0.0/12,DIRECT
IP-CIDR,192.168.0.0/16,DIRECT
```

必要时在 `[Host]` 添加：

```text
*.your-company.com = server:223.5.5.5
git.your-company.com = server:223.5.5.5
```

## Quantumult X

在分流规则最前面加：

```text
host-suffix, your-company.com, direct
host, git.your-company.com, direct
ip-cidr, 10.0.0.0/8, direct
ip-cidr, 172.16.0.0/12, direct
ip-cidr, 192.168.0.0/16, direct
```

如果开了 MitM，把内网域名从 MitM 主机列表里排除。

## V2RayN

在路由规则中添加 direct 出站：

```json
{
  "domain": ["domain:your-company.com", "full:git.your-company.com"],
  "outboundTag": "direct"
}
```

```json
{
  "ip": ["10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16"],
  "outboundTag": "direct"
}
```

## Validation

让用户在重启代理后检查：

```bash
nslookup git.your-company.com
curl -I https://git.your-company.com
ping <internal-ip>
```

判断：

- `nslookup` 返回真实 IP，而不是 `28.x.x.x` 或 `198.18.x.x`，说明 fake-ip 排除生效。
- `curl` 能拿到响应，说明 DIRECT 规则生效。
- 仍是 fake-ip 时，重点检查是否改错文件、覆写未启用、未重启客户端、域名规则不匹配。
- 能解析但 HTTP 不通时，重点检查规则顺序、是否处于 Global 全局模式、公司网络/VPN 是否已连接。

## Troubleshooting

| 现象 | 优先检查 |
| --- | --- |
| `nslookup` 返回 `28.x.x.x` 或 `198.18.x.x` | `fake-ip-filter` 是否包含内网域名 |
| 能 ping 但网页或 Git 不通 | `DIRECT` 规则是否在最前面 |
| 改完订阅更新后失效 | 是否使用了覆写而不是直改订阅 |
| Clash 系修改无效 | 是否改的是 `profiles/` 源文件而不是运行时文件 |
| FlClash 脚本报 `module` 相关错误 | 是否使用了 `function main(config)` |
| 公司域名可解析但仍走代理 | 是否在 Rule 模式，规则是否被前面的代理规则截获 |

## Assets

`assets/` 中有可视化材料。只有当用户需要教程配图、分享图或说明页时再使用：

- `assets/proxy-bypass-cover.html`
- `assets/proxy-bypass-cover.png`
