# 规则集使用说明

本目录用于存放 **mihomo / OpenClash / Nikki** 使用的补充规则集（rule-provider）。
所有域名均经过 DoH 验证，并使用 `+.域名` 后缀写法（见下方“格式说明”）。

---

## 一、文件列表

| 文件 | 条数 | behavior / format | 用途 |
|---|---:|---|---|
| `HK-Bank-Domain.list` | 128 | `domain` / `text` | 香港银行、金融机构、支付基建（汇丰/恒生/渣打/中银/虚拟银行等）|
| `PT-Extra-Domain.list` | 106 | `domain` / `text` | PT 站点补充：截图 Tracker/服务器域名 + RapidDNS 子域 + 实测别名 |
| `DIY-Domain-Direct.list` | 28 | `domain` / `text` | 个人自用「域名直连」补充（小站/非大众站）|
| `DIY-Domain-Proxy.list` | 20 | `domain` / `text` | 个人自用「域名代理」补充 |
| `DIY-IP-Direct.list` | 0 | `ipcidr` / `text` | 个人自用「IP 直连」补充（预留，按需追加）|
| `DIY-IP-Proxy.list` | 0 | `ipcidr` / `text` | 个人自用「IP 代理」补充（预留，按需追加）|

> `PT-Extra-Domain.list` 与上游 `category-pt.list` 可能有少量重叠，属正常现象；两者使用同一个策略组，不影响分流结果。

---

## 二、格式说明（重要）

### 1. `domain` 行为

mihomo 的 `behavior: domain` 中：

| 写法 | 匹配范围 |
|---|---|
| `example.com` | **仅精确匹配** `example.com` |
| `+.example.com` | 匹配 `example.com` **及其所有子域** |
| `.example.com` | 仅匹配子域（不含 apex）|
| `*.example.com` | 单层通配子域 |

因此本目录的域名规则统一使用 **`+.域名`**，确保 `www.`、`tracker.`、`api.` 等子域都能命中。

### 2. `ipcidr` 行为

`DIY-IP-Direct.list` / `DIY-IP-Proxy.list` 使用纯 CIDR 文本格式，例如：

```text
1.2.3.0/24
2001:db8::/32
```

---

## 三、在配置中引用

```yaml
rule-anchor:
  ip:          &ip          {type: http, interval: 86400, behavior: ipcidr,  format: mrs}
  ip_text:     &ip_text     {type: http, interval: 86400, behavior: ipcidr,  format: text}
  domain:      &domain      {type: http, interval: 86400, behavior: domain,  format: mrs}
  domain_text: &domain_text {type: http, interval: 86400, behavior: domain,  format: text}

rule-providers:
  hkbank_domain:    { <<: *domain_text, url: "https://raw.githubusercontent.com/bjyjock/openlist/refs/heads/main/HK-Bank-Domain.list"}
  ptex_domain:      { <<: *domain_text, url: "https://raw.githubusercontent.com/bjyjock/openlist/refs/heads/main/PT-Extra-Domain.list"}
  diy_domain_direct:{ <<: *domain_text, url: "https://raw.githubusercontent.com/bjyjock/openlist/refs/heads/main/DIY-Domain-Direct.list"}
  diy_ip_direct:    { <<: *ip_text,     url: "https://raw.githubusercontent.com/bjyjock/openlist/refs/heads/main/DIY-IP-Direct.list"}
  diy_domain_proxy: { <<: *domain_text, url: "https://raw.githubusercontent.com/bjyjock/openlist/refs/heads/main/DIY-Domain-Proxy.list"}
  diy_ip_proxy:     { <<: *ip_text,     url: "https://raw.githubusercontent.com/bjyjock/openlist/refs/heads/main/DIY-IP-Proxy.list"}
```

规则建议放在 `cn!_domain` 等大规则集之前：

```yaml
rules:
  - RULE-SET,diy_domain_direct,直连
  - RULE-SET,diy_ip_direct,直连,no-resolve
  - RULE-SET,diy_domain_proxy,默认代理
  - RULE-SET,diy_ip_proxy,默认代理,no-resolve
  - RULE-SET,pt_domain,PTDownload          # 上游 category-pt.list
  - RULE-SET,ptex_domain,PTDownload        # 本目录 PT-Extra
  - RULE-SET,pt_ip,PTDownload,no-resolve   # 上游 geoip/pt.list
  - RULE-SET,hkbank_domain,HKBank          # 本目录 HK-Bank
  - RULE-SET,cn!_domain,默认代理
  - MATCH,漏网之鱼
```

---

## 四、配置模板

仓库根目录提供三个模板（按使用场景选择）：

| 文件 | 说明 |
|---|---|
| `byDEF_CS.yaml` | 精简裸核模板，使用系统 DNS |
| `CS.yaml` | 完整模板：fake-ip + DoH + 规则分流 |
| `CS_mosdns.yaml` | 配合 mosdns 的模板（DNS 分流交给 mihomo，mosdns 负责纯净解析）|
| `byDEF_CS_Universal.yaml` | 通用版：香港 / 台湾 / 日本 / 狮城 / 美国 |
| `CS_Universal.yaml` | 通用版（完整 DNS）|
| `CS_mosdns_Universal.yaml` | 通用版（mosdns）|

通用版相对原版：
- **移除** 荷兰节点分组；
- **新增** 台湾、狮城节点分组；
- 区域顺序统一为 **香港 → 台湾 → 日本 → 狮城 → 美国**。

---

## 五、维护流程

### 新增域名

1. 判断走直连还是代理，写入对应 `DIY-Domain-*.list`；
2. 使用 `+.域名` 后缀格式；
3. 如果是 PT 站点，优先写入 `PT-Extra-Domain.list`。

### 验证域名

```bash
# A / AAAA / NS / CNAME
curl -s "https://dns.google/resolve?name=example.com&type=A"
curl -s "https://cloudflare-dns.com/dns-query?name=example.com&type=A" \
     -H "accept: application/dns-json"
```

- `Status=0`：域名存在（即使暂无 A 记录）；
- `Status=3`：NXDOMAIN，应剔除；
- `Status=2`：SERVFAIL，可加 `cd=1` 关闭 DNSSEC 校验重试。

### 去重

与上游规则集（`category-pt`、`geosite` 等）比对，避免重复；
同一策略组内的重复无害，跨组重复需确认是否为“覆盖规则”。

---

## 六、注意事项

1. 本仓库规则仅供个人学习与自用，请遵守当地法律法规及站点服务条款。
2. PT 站点普遍使用 Cloudflare，**IP 规则命中率很低**，建议以域名规则为主。
3. `geoip/pt.list` 为上游快照，可能不包含最新 IP；不要完全依赖 IP 分流。
4. 修改后请使用 `mihomo -t -f xxx.yaml` 或 OpenClash 配置检查验证 YAML。

---

## 七、致谢

- [MetaCubeX/meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat)
- [v2fly/domain-list-community](https://github.com/v2fly/domain-list-community)
- [blackmatrix7/ios_rule_script](https://github.com/blackmatrix7/ios_rule_script)
- [qichiyuhub/rule](https://github.com/qichiyuhub/rule)

最后更新：2026-04-02
