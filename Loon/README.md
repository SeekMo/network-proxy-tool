# Loon 安全防护测试指南

> ⚠️ **初稿**。`config.lcf` 尚未进行安全防护优化，本文仅依据现有配置与已核实的差异做范写。
> 优化完成后，需按实际改动补全「测试前准备」与「测试步骤」，并核实标注为「待确认」的项。

| # | 测试项 | 验证目标 |
|:--:|---|---|
| 1 | 规则与插件 | 分流规则、HTTPDNS 拦截插件是否按预期工作 |
| 2 | DNS 泄露 | 域名解析是否泄露给运营商或非预期的 DNS 服务商 |
| 3 | WebRTC 泄露 | 网页能否通过 STUN 探测到真实 IP |
| 4 | IPv6 泄露 | IPv6 流量是否绕过隧道直接出网 |
| 5 | 通话功能 | 防护措施是否误伤正常的音视频通话 |
| 6 | 日常回归 | 分流是否正确，常用应用是否正常 |

---

## 一、与 Shadowrocket 的关键差异

| 项目 | Shadowrocket | Loon |
|---|---|---|
| DNS 配置 | `dns-server` 一项混写各协议 | `dns-server`、`doh-server`、`doq-server`、`doh3-server` **分项配置** |
| IPv6 控制 | `ipv6 = false` | `ip-mode = ipv4-only`，**当前已启用** |
| 不使用 Fake-IP 的域名 | `always-real-ip` | `real-ip` |
| WebRTC / STUN | `stun-response-ip` 返回假 IP | `disable-stun = true`，**直接禁止 STUN，当前已启用** |
| UDP 回退行为 | `udp-policy-not-supported-behaviour` | `udp-fallback-mode`，**当前为 REJECT** |
| 节点筛选 | 策略组内 `policy-regex-filter` | `[Remote Filter]` 段独立定义 |
| 扩展方式 | 模块（`.sgmodule`） | 插件（`.plugin`） |

> 💡 **WebRTC 测试的预期结果与 Shadowrocket 不同。** Loon 用 `disable-stun` 直接禁止，Trickle ICE 里应表现为**没有 srflx 候选**，而不是显示 `1.1.1.1`。

---

## 二、已知问题（优化阶段需处理）

### 1. `dns-server` 含明文 DNS 且包含 `system`

当前为 `223.5.5.5,119.29.29.29,114.114.114.114,223.6.6.6,system`。`system` 会把查询直接交给运营商，明文地址则可被链路上的旁观者看到和篡改。

`doh-server`、`doq-server`、`doh3-server` 均已配置加密地址，**待确认**：Loon 对这几项是并发查询还是有优先级。若为并发，`dns-server` 里的明文查询每次都会发出。

### 2. `GEOIP, CN, DIRECT` 未带 `no-resolve`

位于 `[Rule]` 段。**待确认**：Loon 遇到 IP 类规则时是否会触发本地 DNS 解析。若会，所有未被前面规则命中的域名都会在本机解析一次，是 DNS 泄露源。

> 💡 Shadowrocket 当初正是这个问题，通过给 IP 规则补 `no-resolve` 解决。

### 3. China 规则集未加载域名部分

`[Remote Rule]` 引用了 `China.list`，但该列表仅 63 行（IP 与关键词规则），**域名部分在 `China_Domain.list` 中，当前未引用**。这会导致大量国内域名匹配不到直连规则，最终落到 `FINAL → 兜底线路` 走代理。

> ✅ 好消息：Loon 版 China.list 的 21 条 IP 规则**全部带 `no-resolve`**，不会引入泄露。

### 4. 配置内存在 `https:///` 笔误

多了一个斜杠，需修正。

### 5. `disable-udp-ports = 443,80` 的实际影响待评估

需确认它与 QUIC 屏蔽、通话功能之间是否存在冲突。

### 6. HTTPDNS 拦截插件无泄露风险

`Loon-HTTPDNS.Block.plugin` 已启用，其 24 条 IP 规则**全部带 `no-resolve`**，可保持启用。

---

## 三、测试前准备（草稿）

- [ ] 导入最新配置，确认当前使用的是这一份
- [ ] 确认运行模式为**规则模式**，而非直连或全局
- [ ] 确认 HTTPDNS 拦截插件已启用
- [ ] 选好节点，确认能正常上网
- [ ] 记下真实 IP 备用：先关闭 Loon，访问 `ip.sb` 记录 IP，再重新开启

> ⚠️ 后续所有测试都要和这个真实 IP 做对比。检测页会显示真实 IP，截图外发前记得打码。

---

## 四、测试步骤（草稿）

### 第 1 步 · 规则与插件生效情况

**操作** 　**待确认**：Loon 是否有类似 Shadowrocket「测试规则」的功能。若没有，改用实际访问 + 查看连接日志的方式验证。

**预期结果** 　国内域名走 DIRECT，境外域名走对应策略组，检测站走代理，HTTPDNS 域名被拒绝。

<br>

### 第 2 步 · DNS 泄露测试

**操作**

1. 打开 https://browserleaks.com/dns
2. **先看页面顶部的 IP**，必须是节点 IP。若是真实 IP，说明该站没走代理，下方结果无效
3. 再看列出的 DNS 服务器

**预期结果**

| 结果 | 判定 |
|---|---|
| 仅节点那一侧的解析服务器 | ✅ 通过 |
| 出现腾讯、阿里 | ❌ 发生了本地解析，需排查 |
| 出现当前运营商 | ❌ 最严重，多半是 `dns-server` 里的 `system` 所致 |

<br>

### 第 3 步 · WebRTC 泄露测试

**操作** 　同 Shadowrocket：打开 Trickle ICE，逐个测试下列 STUN 地址。

```
stun:stun.l.google.com:19302
stun:stun.chat.bilibili.com:3478
stun:stun.miwifi.com:3478
```

**预期结果** 　`disable-stun = true`，三个地址均应**没有 srflx 候选**。

| 结果 | 判定 |
|---|---|
| 无 srflx，仅 host | ✅ `disable-stun` 生效 |
| 出现节点 IP | ⚠️ `disable-stun` 未生效，但该路径走了代理，不泄露 |
| 出现真实 IP | ❌ 泄露 |

<br>

### 第 4 步 · IPv6 泄露测试

**操作** 　先关闭 Loon，打开 https://test-ipv6.com 确认该网络有 IPv6；再开启后重测。

**预期结果** 　`ip-mode = ipv4-only` 会拒绝 IPv6 连接，应无 IPv6 地址。

<br>

### 第 5 步 · 通话功能验证

**操作** 　Google Meet、微信语音、FaceTime 等各试一个。

**预期结果** 　能正常通话。`disable-stun` 与 `disable-udp-ports` 都可能影响通话，**若异常，优先怀疑这两项**。

<br>

### 第 6 步 · 日常功能回归

| 测试项 | 预期结果 |
|---|---|
| 淘宝、B站、微信、支付宝 | 正常且快，走直连 |
| Google、YouTube、Twitter | 正常，走代理 |
| App Store、iCloud | 正常 |
| AI 服务、TikTok、PayPal、加密货币 | 走各自设定的策略组 |

---

## 五、注意事项

### 判定顺序不能颠倒：先确认 IP，再看 DNS

先看检测页顶部的 IP 是不是节点 IP。**若是真实 IP，说明该站没走代理，后面的 DNS 结论一律无效。**

### 「出现腾讯、阿里」是否算泄露，取决于域名本该走哪条路

| 域名类型 | 解析来源 | 判定 |
|---|---|---|
| 国内直连域名 | 腾讯、阿里 | ✅ 正常，可分到就近 CDN |
| 走代理的境外域名 | 节点那一侧 | ✅ 正常 |
| 走代理的境外域名 | 腾讯、阿里 | ❌ 异常，说明触发了本地解析 |
| 任意域名 | 当前运营商 | ❌ 最严重 |

### 本地规则优先于远程规则

Loon 中 `[Rule]` 的优先级高于 `[Remote Rule]`，检测站的修正规则写在本地即可覆盖远程规则集。

---

## 六、测试进度与结果

> 待配置优化并完成测试后填写。

| 测试项 | iPhone | Mac | 说明 |
|---|:--:|:--:|---|
| 第 1 步 规则与插件 | ⬜ | ⬜ | |
| 第 2 步 DNS 泄露 | ⬜ | ⬜ | |
| 第 3 步 WebRTC 泄露 | ⬜ | ⬜ | |
| 第 4 步 IPv6 泄露 | ⬜ | ⬜ | |
| 第 5 步 通话功能 | ⬜ | ⬜ | |
| 第 6 步 日常回归 | ⬜ | ⬜ | |

<small>✅ 通过 ｜ ⏸ 待补测 ｜ ⬜ 待测</small>

---

## 七、待办清单

- [ ] **完成 `config.lcf` 的安全防护优化**
- [ ] **核实 `dns-server` 与各加密 DNS 项的查询顺序** —— 决定明文与 `system` 是否必须清理
- [ ] **核实 IP 类规则与 `GEOIP` 是否触发本地解析** —— 决定是否需要补 `no-resolve`
- [ ] **评估是否引用 `China_Domain.list`** —— 同时注意是否会引入检测站走直连的问题
- [ ] **依据优化结果补全本文的测试步骤**
- [ ] **执行测试并记录结果**
