# QuantumultX 安全防护测试指南

> ⚠️ **初稿**。`config.conf` 尚未进行安全防护优化，本文仅依据现有配置与已核实的差异做范写。
> 优化完成后，需按实际改动补全「测试前准备」与「测试步骤」，并核实标注为「待确认」的项。

| # | 测试项 | 验证目标 |
|:--:|---|---|
| 1 | 规则与重写 | 分流规则、HTTPDNS 拦截重写是否按预期工作 |
| 2 | DNS 泄露 | 域名解析是否泄露给运营商或非预期的 DNS 服务商 |
| 3 | WebRTC 泄露 | 网页能否通过 STUN 探测到真实 IP |
| 4 | IPv6 泄露 | IPv6 流量是否绕过隧道直接出网 |
| 5 | 通话功能 | 防护措施是否误伤正常的音视频通话 |
| 6 | 日常回归 | 分流是否正确，常用应用是否正常 |

---

## 一、与 Shadowrocket 的关键差异

| 项目 | Shadowrocket | QuantumultX |
|---|---|---|
| DNS 配置 | `[General]` 的 `dns-server` 一项混写 | `[dns]` 段，`doh-server` 与 `server` 分开写 |
| 系统 DNS | 由 `dns-server` 中是否含 `system` 决定 | `no-system` 一项开关，**当前已启用，即不使用系统 DNS** |
| IPv6 | `ipv6 = false` | `no-ipv6`，**当前已启用** |
| 不使用 Fake-IP 的域名 | `always-real-ip` | `dns_exclusion_list` |
| WebRTC / STUN | `stun-response-ip` 返回假 IP | `udp_drop_list = 443, STUN, QUIC`，**直接丢弃 STUN 包** |
| 策略组类型 | `select` / `url-test` | `static` / `url-latency-benchmark` |
| HTTPDNS 拦截 | 模块，含 `[Rule]` 分流规则 | `QX-HTTPDNS.Block.conf`，**只有 URL 重写，没有分流规则** |

> 💡 **WebRTC 测试的预期结果与 Shadowrocket 不同。** QuantumultX 是直接丢弃 STUN 包，Trickle ICE 里应表现为**没有 srflx 候选**，而不是显示某个假 IP。

---

## 二、已知问题（优化阶段需处理）

### 1. `[dns]` 段混有大量明文 DNS

`doh-server` 已配置了加密的 doh.pub 与 AliDNS，但 `server` 又列了 `223.5.5.5`、`119.29.29.29`、`114.114.114.114`、`119.28.28.28` 等明文地址，另有一批域名被单独指定到明文 DNS 解析（如 `server = /apple.com/119.29.29.29`）。

**待确认**：QuantumultX 对 `doh-server` 与 `server` 是并发查询还是有优先级。若为并发，明文查询每次都会发出，加密将形同虚设。

### 2. China 规则集未拆分，且同样收录了 browserleaks.com

QuantumultX 版 `China.list` 共 3753 行（未像 Shadowrocket 那样拆成 `China_Domain.list`），**其中包含 `browserleaks.com`**。该规则集以 `force-policy = direct` 引用，因此检测站会走直连，测试结果会失真。

处理方式参考 Shadowrocket：在 `[filter_local]` 中把检测站显式指向代理策略，且本地规则优先于远程规则。

### 3. China 规则集的 IP 规则缺少 `no-resolve`

该列表内 17 条 `IP-CIDR` 规则**全部没有 `no-resolve`**。

**待确认**：QuantumultX 遇到 IP 类规则时是否也会触发本地 DNS 解析（Shadowrocket 会）。若会，则与 Shadowrocket 当初的情况相同，是 DNS 泄露源。

### 4. `geoip, cn, direct` 未带 `no-resolve`

位于 `[filter_local]`，同上，需确认是否会触发本地解析。

### 5. HTTPDNS 拦截能力弱于其他客户端

`QX-HTTPDNS.Block.conf` 只包含 URL 重写，没有分流规则，**HTTPS 请求必须开启 MITM 才能拦截**。好处是不含 IP 规则，不会引入 DNS 泄露。

### 6. 配置内存在 `https:///` 笔误

多了一个斜杠，需修正。

---

## 三、测试前准备（草稿）

- [ ] 导入最新配置，确认当前使用的是这一份
- [ ] 确认分流模式为**规则分流**，而非全局直连或全局代理
- [ ] 确认 HTTPDNS 拦截重写已启用
- [ ] 选好节点，确认能正常上网
- [ ] 记下真实 IP 备用：先关闭 QuantumultX，访问 `ip.sb` 记录 IP，再重新开启

> ⚠️ 后续所有测试都要和这个真实 IP 做对比。检测页会显示真实 IP，截图外发前记得打码。

---

## 四、测试步骤（草稿）

### 第 1 步 · 规则与重写生效情况

**操作** 　**待确认**：QuantumultX 是否有类似 Shadowrocket「测试规则」的功能。若没有，改用实际访问 + 查看连接日志的方式验证。

**预期结果** 　国内域名走 direct，境外域名走对应策略组，检测站走代理。

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
| 出现当前运营商 | ❌ 最严重 |

<br>

### 第 3 步 · WebRTC 泄露测试

**操作** 　同 Shadowrocket：打开 Trickle ICE，逐个测试下列 STUN 地址。

```
stun:stun.l.google.com:19302
stun:stun.chat.bilibili.com:3478
stun:stun.miwifi.com:3478
```

**预期结果** 　`udp_drop_list` 含 `STUN`，三个地址均应**没有 srflx 候选**。

| 结果 | 判定 |
|---|---|
| 无 srflx，仅 host | ✅ STUN 被丢弃 |
| 出现节点 IP | ⚠️ STUN 未被丢弃，但该路径走了代理，不泄露 |
| 出现真实 IP | ❌ 泄露 |

<br>

### 第 4 步 · IPv6 泄露测试

**操作** 　先关闭 QuantumultX，打开 https://test-ipv6.com 确认该网络有 IPv6；再开启后重测。

**预期结果** 　`no-ipv6` 已启用，应无 IPv6 地址，或仅显示节点的 IPv6。

<br>

### 第 5 步 · 通话功能验证

**操作** 　Google Meet、微信语音、FaceTime 等各试一个。

**预期结果** 　能正常通话。`udp_drop_list` 同时丢弃了 443 与 QUIC，**若通话或网页异常，优先怀疑这一项**。

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

QuantumultX 中 `[filter_local]` 的优先级高于 `[filter_remote]`，因此检测站的修正规则写在本地即可覆盖远程规则集。

---

## 六、测试进度与结果

> 待配置优化并完成测试后填写。

| 测试项 | iPhone | Mac | 说明 |
|---|:--:|:--:|---|
| 第 1 步 规则与重写 | ⬜ | ⬜ | |
| 第 2 步 DNS 泄露 | ⬜ | ⬜ | |
| 第 3 步 WebRTC 泄露 | ⬜ | ⬜ | |
| 第 4 步 IPv6 泄露 | ⬜ | ⬜ | |
| 第 5 步 通话功能 | ⬜ | ⬜ | |
| 第 6 步 日常回归 | ⬜ | ⬜ | |

<small>✅ 通过 ｜ ⏸ 待补测 ｜ ⬜ 待测</small>

---

## 七、待办清单

- [ ] **完成 `config.conf` 的安全防护优化**
- [ ] **核实 `doh-server` 与 `server` 的查询顺序** —— 决定明文 DNS 是否必须清理
- [ ] **核实 IP 类规则与 `geoip` 是否触发本地解析** —— 决定是否需要补 `no-resolve`
- [ ] **处理检测站被 China.list 分到直连的问题**
- [ ] **依据优化结果补全本文的测试步骤**
- [ ] **执行测试并记录结果**
