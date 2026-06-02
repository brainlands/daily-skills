---
name: proxy-intranet-bypass
description: '帮助用户配置代理软件，让公司内网流量直连、不走代理节点，解决「开了代理就上不了内网」的问题。支持 Clash/FlClash/Clash Verge/Shadowrocket/Quantumult X/V2RayN 等主流代理工具。当用户提到"代理内网"、"内网直连"、"代理 bypass"、"内网不通"、"代理和公司内网冲突"等场景时触发。'
---

# 代理内网直通配置助手

帮助用户一键配置代理软件的「内网直通」规则，解决开了代理后公司内网（Git、文档系统、内部 API）无法访问的问题。

## 核心原理（三板斧）

不管用什么代理软件，配置本质只改三处：

| 序号 | 改什么 | 为什么 |
|------|--------|--------|
| 1 | **fake-ip-filter** | 防止内网域名被分配假 IP（28.x.x.x），导致根本连不上 |
| 2 | **nameserver-policy** | 让内网域名走国内 DNS，拿到真实 IP |
| 3 | **rules**（最前面加 DIRECT） | 让内网流量直连，不走代理节点 |

## 执行流程

### Step 1：收集信息

向用户确认以下信息（已提供则跳过）：

1. **代理软件**：Clash / FlClash / Clash Verge / Clash Verge Rev / Shadowrocket / Quantumult X / V2RayN / Clash for Windows
2. **公司内网域名**：例如 `your-company.com`、`your-internal.cn`（支持多个）
3. **是否开了 TUN 模式 / fake-ip 模式**：如果不确定，默认按「开了」处理

### Step 2：根据代理软件输出对应配置

---

#### Clash 系（直接修改 profile 文件）

**找到配置文件**（不是运行时 `config.yaml`，是 `profiles/` 下的源文件）：

| 软件 | 路径 |
|------|------|
| FlClash (macOS) | `~/Library/Application Support/com.follow.clash/profiles/xxx.yaml` |
| Clash Verge (macOS) | `~/.config/clash-verge/profiles/xxx.yaml` |
| Clash Verge Rev (macOS) | `~/.config/clash-verge-rev/profiles/xxx.yaml` |
| Clash (Windows) | `C:\Users\<username>\.config\clash\profiles\xxx.yaml` |

**三处修改：**

```yaml
# 1️⃣ fake-ip-filter 追加内网域名（在数组末尾 ] 前追加）
fake-ip-filter:
  # ... 原有内容保留 ...
  - '+.your-company.com'
  - '+.your-internal.cn'

# 2️⃣ nameserver-policy 追加内网 DNS
nameserver-policy:
  '+.your-company.com': ['223.5.5.5', '119.29.29.29']
  '+.your-internal.cn': ['223.5.5.5', '119.29.29.29']

# 3️⃣ rules 最前面加直连规则（必须在所有其他规则之前！）
rules:
  - 'DOMAIN-SUFFIX,your-company.com,DIRECT'
  - 'DOMAIN-SUFFIX,your-internal.cn,DIRECT'
  - 'IP-CIDR,10.0.0.0/8,DIRECT,no-resolve'
  - 'IP-CIDR,172.16.0.0/12,DIRECT,no-resolve'
  - 'IP-CIDR,192.168.0.0/16,DIRECT,no-resolve'
  # ... 原有规则保留 ...
```

---

#### FlClash 覆写脚本（推荐，订阅更新不覆盖）

步骤：配置 → 当前 profile 右键 → 更多 → 覆写 → 切换到「脚本」模式 → 新建脚本：

```javascript
function main(config) {
  // 1️⃣ rules 最前面插入内网直连规则
  config.rules.unshift(
    'DOMAIN-SUFFIX,your-company.com,DIRECT',
    'DOMAIN-SUFFIX,your-internal.cn,DIRECT',
    'IP-CIDR,10.0.0.0/8,DIRECT,no-resolve',
    'IP-CIDR,172.16.0.0/12,DIRECT,no-resolve',
    'IP-CIDR,192.168.0.0/16,DIRECT,no-resolve'
  )

  // 2️⃣ fake-ip-filter 追加内网域名
  if (config.dns && config.dns['fake-ip-filter']) {
    config.dns['fake-ip-filter'].push(
      '+.your-company.com',
      '+.your-internal.cn'
    )
  }

  // 3️⃣ nameserver-policy 追加内网 DNS
  if (config.dns) {
    if (!config.dns['nameserver-policy']) {
      config.dns['nameserver-policy'] = {}
    }
    config.dns['nameserver-policy']['+.your-company.com'] = ['223.5.5.5', '119.29.29.29']
    config.dns['nameserver-policy']['+.your-internal.cn'] = ['223.5.5.5', '119.29.29.29']
  }

  return config
}
```

> ⚠️ FlClash 必须用 `function main(config)` 格式，不支持 `module.exports`！

保存后：勾选脚本（空心○ → 实心●）→ 确认「脚本」模式 → 重启 FlClash。

---

#### Clash Verge / Clash Verge Rev 覆写（推荐）

覆写 → 标准模式 → 在对应区域添加：

```yaml
# Rules 区域 → prepend-rules
prepend-rules:
  - DOMAIN-SUFFIX,your-company.com,DIRECT
  - DOMAIN-SUFFIX,your-internal.cn,DIRECT
  - IP-CIDR,10.0.0.0/8,DIRECT,no-resolve
  - IP-CIDR,172.16.0.0/12,DIRECT,no-resolve
  - IP-CIDR,192.168.0.0/16,DIRECT,no-resolve

# DNS 区域 → fake-ip-filter 末尾追加
fake-ip-filter:
  - '+.your-company.com'
  - '+.your-internal.cn'
```

---

#### Shadowrocket（iOS 小火箭）

配置 → 当前配置 ⓘ → 编辑纯文本 → `[Rule]` 最前面加：

```
DOMAIN-SUFFIX,your-company.com,DIRECT
DOMAIN-SUFFIX,your-internal.cn,DIRECT
IP-CIDR,10.0.0.0/8,DIRECT
IP-CIDR,172.16.0.0/12,DIRECT
IP-CIDR,192.168.0.0/16,DIRECT
```

`[Host]` 部分（可选）：

```
*.your-company.com = server:223.5.5.5
*.your-internal.cn = server:119.29.29.29
```

---

#### Quantumult X（iOS）

分流 → 引用 → 编辑 → 分流规则最前面加：

```
host-suffix, your-company.com, direct
host-suffix, your-internal.cn, direct
ip-cidr, 10.0.0.0/8, direct
ip-cidr, 172.16.0.0/12, direct
ip-cidr, 192.168.0.0/16, direct
```

> 如果开了 MitM，把内网域名从 MitM 列表里排除。

---

#### V2RayN（Windows）

设置 → 路由设置 → 预路由规则集：

域名规则：
```json
{ "domain": ["domain:your-company.com", "domain:your-internal.cn"], "outboundTag": "direct" }
```

IP 规则：
```json
{ "ip": ["10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16"], "outboundTag": "direct" }
```

---

### Step 3：验证配置

配置完成并重启代理后，执行以下验证：

```bash
# 检查 DNS 解析是否拿到真实 IP（不是 28.x.x.x）
nslookup git.your-company.com

# 检查 HTTP 是否可访问
curl -I https://git.your-company.com

# 如果在公司内网，ping 内网 IP
ping 10.x.x.x
```

**判断标准：**
- ✅ `nslookup` 返回的不是 `28.x.x.x` / `198.18.x.x` → fake-ip-filter 生效
- ✅ `curl` 拿到 HTTP 响应 → 直连规则生效
- ❌ 仍返回 `28.x.x.x` → 配置未生效，检查是否改错文件或未重启

## 常见问题速查

| 现象 | 原因 | 解法 |
|------|------|------|
| nslookup 返回 28.x.x.x | fake-ip 还在劫持内网域名 | fake-ip-filter 加内网域名 |
| 能 ping 通但 HTTP 不通 | 流量走了代理节点 | rules 最前面加 DIRECT |
| 改完重启还是不生效 | 改的是运行时 config.yaml | 改 profiles/ 下的源文件 |
| 订阅更新后配置丢了 | 订阅覆盖了修改 | 改用覆写（Override）方式 |
| 规则加了但没生效 | 规则顺序靠后 | 内网规则必须放最前面 |
| 覆写报错 `Can't find variable: module` | FlClash 不支持 module.exports | 改用 `function main(config)` |
| 覆写脚本没生效 | 没勾选 / 没切脚本模式 | 勾选 ● + 确认脚本模式 |

## 注意事项

- **修改前备份**：修改任何配置文件前，先备份原文件
- **用覆写而非直改**：优先推荐覆写（Override）方式，订阅更新不丢失
- **不要用全局模式**：确保代理软件处于「规则模式（Rule）」而非「全局模式（Global）」
- **国内流量不走代理**：大多数订阅已内置 `GEOIP,CN,DIRECT`，国内网站不会消耗代理流量

## 辅助材料

`assets/` 目录下提供了可视化说明材料：

- `assets/proxy-bypass-cover.html` — 信息图网页版（浏览器打开即可查看，包含痛点分析、三板斧流程图、踩坑速查表）
- `assets/proxy-bypass-cover.png` — 信息图截图版（适合直接分享或嵌入文档）

可作为教程配图或分享给同事时的可视化参考。
