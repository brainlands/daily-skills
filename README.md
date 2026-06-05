# Daily Skills

日常工作技能集合，用来沉淀一些可以直接复用的 Codex/Agent skill。

这个仓库目前只有一个技能：`proxy-intranet-bypass`。

## proxy-intranet-bypass

用于解决「开代理后访问不了公司内网」的问题。

常见场景：

- 开了代理、Clash、FlClash、Clash Verge、Shadowrocket、Quantumult X 后，公司内网打不开
- Git、内部文档、内部 API、OA、测试环境等内网服务无法访问
- 公司内网域名被代理软件错误解析，出现 fake-ip、DNS 污染、内网不通
- 需要配置内网直连、代理绕过、bypass、DIRECT、fake-ip-filter、nameserver-policy、rules

技能位置：

```text
proxy-intranet-bypass/SKILL.md
```

### 踩坑记录

- **Git 域名解析到不可达内网 IP**：公司已将 Git 从内网迁至公网，但办公网 DNS 仍返回旧内网地址（如 `10.x.x.x`），导致 Git 不可用。这不是代理配置问题，而是 DNS 解析到已废弃地址。优先恢复系统 DNS 为自动获取（DHCP 会下发公司自建 DNS），兜底方案是手动绑定 hosts 指向公网 IP。详见 [reference.md](proxy-intranet-bypass/reference.md)。

## Search Keywords

代理内网, 内网直连, 开代理访问不了内网, 代理 bypass, Clash 内网不通, FlClash 内网直连, Clash Verge 内网直连, Shadowrocket 内网直连, Quantumult X 内网直连, fake-ip-filter, nameserver-policy, DIRECT, DOMAIN-SUFFIX, IP-CIDR

## Roadmap

后续会继续补充更多适合日常工作使用的实用 skill。
