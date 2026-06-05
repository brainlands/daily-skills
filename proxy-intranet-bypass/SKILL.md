---
name: proxy-intranet-bypass
description: "Configure proxy clients to route company intranet, private domains, Git servers, internal docs, OA, APIs, and RFC1918 LAN traffic DIRECT instead of through proxy nodes. Use when users report opening Clash, FlClash, Clash Verge, Clash Verge Rev, Shadowrocket, Quantumult X, V2RayN, or similar tools makes intranet unavailable. Keywords: 开代理访问不了内网, 代理内网, 内网直连, bypass, DIRECT, fake-ip-filter, nameserver-policy, rules, TUN, fake-ip, 28.x.x.x, 198.18.x.x."
---

# 代理内网直通配置助手

解决"开代理后访问不了公司内网"问题，让内网域名、私网 IP 直连不走代理。

## 核心原则

优先**覆写/Override**方案，避免订阅更新后丢配置。仅当用户明确要求时才指导修改 profile 源文件。

## 三类配置

所有方案围绕三类配置展开：

| 配置 | 目的 |
| --- | --- |
| `rules` / 分流规则 | 内网域名 + 私网 IP → `DIRECT` |
| `fake-ip-filter` | 防止内网域名被分配假 IP（`28.x.x.x` / `198.18.x.x`） |
| `nameserver-policy` / Host | 内网域名使用正确 DNS 解析 |

## Workflow

1. **收集信息**：代理软件、内网域名/后缀、是否 TUN/fake-ip、操作系统。不重复用户已提供的信息。
2. **按场景输出**：
   - 仅说"内网打不开" → 通用 DIRECT 规则 + 排查命令
   - 给出具体软件 → 对应软件的配置方案
3. **Clash 系优先覆写**；所有规则必须放在现有规则**最前面**。
4. **域名规则**：后缀匹配用 `DOMAIN-SUFFIX`（去掉开头点），具体主机名用 `DOMAIN`。
5. **私网 IP 段**（所有工具通用）：`10.0.0.0/8`、`172.16.0.0/12`、`192.168.0.0/16`。
6. 修改前备份，修改后**保存 + 启用覆写 + 重启客户端**。
7. 提供验证命令和判断标准。
8. **特殊情况**：仅 Git 域名解析到不可达内网 IP → 非代理问题，见 [reference.md](reference.md)。

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

## Validation & Troubleshooting

重启代理后验证：

```bash
nslookup git.your-company.com
curl -I https://git.your-company.com
```

| 判断 | 说明 |
| --- | --- |
| `nslookup` 返回真实 IP（非 `28.x`/`198.18.x`） | fake-ip 排除生效 |
| `curl` 拿到响应 | DIRECT 规则生效 |
| 仍返回 fake-ip | 检查：改错文件 / 覆写未启用 / 未重启 / 域名不匹配 |
| 能解析但 HTTP 不通 | 检查：规则顺序 / 是否 Global 模式 / 公司网络连接 |

### 常见问题

| 现象 | 检查项 |
| --- | --- |
| nslookup 返回 `28.x`/`198.18.x` | `fake-ip-filter` 是否包含内网域名 |
| 能 ping 但网页/Git 不通 | DIRECT 规则是否在最前面 |
| 订阅更新后失效 | 是否使用覆写而非直改订阅 |
| Clash 系修改无效 | 是否改 `profiles/` 源文件而非运行时文件 |
| FlClash 脚本报 `module` 错误 | 是否使用了 `function main(config)` |
| 公司域名可解析但仍走代理 | 是否 Rule 模式，规则是否被前面截获 |
| 仅 Git 域名解析到不可达内网 IP | 非代理问题，见 [reference.md](reference.md) |
| 开代理后网页可访问但 git pull SSH 不通 | TUN 对 SSH 流量处理不同于 HTTPS；或 `/etc/hosts` 有重复条目。见 [reference.md](reference.md) |
| `/etc/hosts` 同域名多条记录 | macOS 优先使用第一条匹配记录，后面的条目不生效。见 [reference.md](reference.md) |

## Assets

`assets/` 中有可视化材料，仅当用户需要配图或分享时使用。
