# Stash 安全防护测试指南

> 本指南用于验证 `config.yaml` 优化后的 DNS / WebRTC 安全防护是否真正生效。
> 按顺序执行第 1～7 步，每一步都给出了操作方法、预期结果，以及不符合预期时的排查方向。
> 规则如何匹配、何时触发本地 DNS 解析，见 [附录 分流规则匹配与 DNS 解析过程](#附录-分流规则匹配与-dns-解析过程)。

|   #   | 测试项           | 验证目标                                                     |
| :---: | ---------------- | ------------------------------------------------------------ |
|   1   | 规则与规则集     | 协议规则、`no-resolve`、规则集格式与顺序、检测站规则是否生效 |
|   2   | DNS 泄露         | 域名解析是否泄露给运营商或非预期的 DNS 服务商                |
|   3   | WebRTC 泄露      | 网页能否通过 STUN 探测到真实 IP                              |
|   4   | IPv6 泄露        | IPv6 流量是否绕过隧道直接出网                                |
|   5   | 通话功能         | 默认模式（拦截 STUN）对各类通话的影响                        |
|   6   | 通话模式（可选） | 切换到通话模式后，STUN 是否都走代理、不暴露真实 IP           |
|   7   | 日常回归         | 分流是否正确，常用应用是否正常                               |

---

## 一、与 Shadowrocket、QuantumultX、Loon 的关键差异

| 项目                  | Shadowrocket                                  | QuantumultX                                 | Loon                                                   | Stash                                                                                     |
| --------------------- | --------------------------------------------- | ------------------------------------------- | ------------------------------------------------------ | ----------------------------------------------------------------------------------------- |
| 配置格式              | `.conf`，分 `[段名]`                          | `.conf`，分 `[段名]`                        | `.lcf`，分 `[段名]`                                    | **`.yaml`（Clash 系语法），缩进敏感**                                                     |
| 加密 DNS              | `dns-server` + 加密的 `fallback-dns-server`   | `doh-server`，没有加密备用 DNS              | `doh-server` + `doq-server`，失败回落明文 `dns-server` | `nameserver`（doh.pub、AliDNS 两个 DoH **并发**，取最快）；**不回落**明文                   |
| 防止 IP 规则触发解析  | `GEOIP,CN,DIRECT,no-resolve`                  | 不支持 `no-resolve`，改用 `host-keyword, .` | `GEOIP,CN,DIRECT,no-resolve`                           | `GEOIP,CN,DIRECT,no-resolve`（官方文档明确支持）                                          |
| 规则优先级            | 从上到下                                      | 先按类型分档，同档内才看本地/远程           | 域名类先于 IP 类；本地 > 插件 > 订阅                   | **严格从上到下**；覆写（stoverride）的规则**插在最前**                                    |
| 不使用 Fake-IP 的域名 | `always-real-ip`                              | `dns_exclusion_list`                        | `real-ip`                                              | `dns.fake-ip-filter`                                                                      |
| WebRTC / STUN         | `stun-response-ip` 返回假 IP                  | `udp_drop_list` 丢弃 STUN                   | `disable-stun = true` 丢弃 STUN                        | **`PROTOCOL,STUN,REJECT`** 规则拦截（没有 `stun-response-ip`，也没有全局开关）             |
| QUIC                  | `block-quic = all-proxy`                      | `udp_drop_list` 含 `QUIC`                   | `disable-udp-ports = 443,80`                           | **`PROTOCOL,QUIC,REJECT`** 规则拦截，不误伤 UDP 443 上的非 QUIC 流量                      |
| 通话模式切换          | 注释 `stun-response-ip` + 放开配套规则        | 改 `udp_drop_list` + 放开配套规则           | 改 `[General]` 两行 + 放开 11 行规则                   | **只放开 1 行** `PROTOCOL,STUN,兜底线路`                                                  |
| 节点不支持 UDP 时     | `udp-policy-not-supported-behaviour = REJECT` | `fallback_udp_policy = reject`              | `udp-fallback-mode = REJECT`                           | 无对应配置；由 `PROTOCOL,STUN,REJECT` 兜底（见已知问题 7）                                |
| 硬编码 DNS            | `hijack-dns` 列出 11 个 IP                    | 无对应配置                                  | `hijack-dns = *:53`                                    | 无对应配置项                                                                              |
| IPv6                  | 配置项控制                                    | 配置项控制                                  | `ip-mode = ipv4-only`                                  | **App 内开关**「启用 Tunnel IPv6 路由」，默认关闭，配置文件无对应项                        |
| HTTPDNS 拦截          | 模块                                          | 远程重写                                    | 插件（IP 规则均带 `no-resolve`）                       | **未启用**：现成覆写有 16 条 IP-CIDR 缺 `no-resolve`（见已知问题 3）                       |
| 主代理 `PROXY`        | 内置                                          | 内置 `proxy`                                | 自定义 `PROXY` 策略组                                  | 自定义 `PROXY` 策略组（原「手动选择」）                                                   |

> 💡 **WebRTC 测试的预期结果与 QuantumultX、Loon 相同。** Stash 直接拦截 STUN，Trickle ICE 里应表现为**没有 srflx 候选**，而不是显示某个假 IP。

---

## 二、已知问题与限制

### 1. `PROTOCOL,STUN`、`PROTOCOL,QUIC` 依赖 Stash 版本

官方[规则类型](https://stash.wiki/rules/rule-types)文档列出了 `PROTOCOL` 的可选值 TCP / UDP / HTTP / TLS / QUIC / STUN，但更新日志中 3.0.0 引入 `PROTOCOL` 时只列了 TCP / HTTP / HTTPS / UDP / QUIC，**STUN 的最低版本未知**。若导入配置时报错，改用 `script.shortcuts` 中按端口匹配的备用规则：

```yaml
  # - PROTOCOL,QUIC,REJECT,no-track
  - SCRIPT,quic,REJECT,no-track
  # - PROTOCOL,STUN,REJECT,no-track
  - SCRIPT,webrtc-stun,REJECT,no-track
```

⚠️ 备用的 `webrtc-stun` 只覆盖 Google 的 19302-19309 端口，**拦不住 3478 端口的国内 STUN**（如 `stun.chat.bilibili.com`），防护明显变弱，应尽快升级 Stash。

### 2. 规则集格式：YAML 与 MRS

- Loyalsoldier 的 `.txt` 文件内容其实是 YAML（首行 `payload:`），已改为 `format: yaml`。**原配置写的是 `format: text`，这几个规则集很可能一直是空的**，第 1 步会核对条目数。
- 广告规则集改为 Cats-Team AdRules 的 **MRS 二进制格式**（`format: mrs`），第 1 步需确认 Stash 能正常加载。

### 3. HTTPDNS 覆写不能直接启用

`Stash-HTTPDNS.Block.stoverride` 中有 16 条 `IP-CIDR` 未带 `no-resolve`，而覆写的规则会**插入到本配置所有规则之前**。启用后，几乎所有域名请求都会先在本地解析，第 2 步必然失败。如需 HTTPDNS 拦截，复制其规则自建覆写，并为这 16 条补上 `no-resolve`。

### 4. 默认模式会影响部分通话

`PROTOCOL,STUN,REJECT` 拦截所有 STUN 报文，ICE 连通检测与 UDP TURN 也会被一并拦截。需要通话时切换到「通话模式」，见第 6 步。

### 5. 未收录的国内小站会走代理

这是 `no-resolve` 防 DNS 泄漏的代价。遇到时在 `rules:` 中为其单独加一条 `- DOMAIN-SUFFIX,域名,DIRECT`，放在 `MATCH` 之前。

### 6. 被拒绝的连接不会出现在连接记录中

QUIC 与 STUN 两条 REJECT 规则都带了 `no-track`，被拒绝的连接不会显示，避免大量记录刷屏。因此这两项要通过网页结果来验证（第 1 步的 HTTP/3 检测、第 3 步的 Trickle ICE）。如需在记录中直接看到，可临时去掉 `no-track`。

### 7. 节点不支持 UDP 时的行为未核实

Clash 内核中，UDP 流量命中的策略若不支持 UDP，会**跳过这条规则继续向下匹配**。通话模式下 STUN 若因此落到 bilibili、`final-direct-ip` 等直连规则，就会暴露真实 IP。配置在通话规则下方保留了 `PROTOCOL,STUN,REJECT` 作为保险，无论 Stash 是否采用这一行为都能兜住。第 6 步有一项可选测试用于核实。

### 8. 需网页登录的公共 Wi-Fi

`nameserver` 只有两个 DoH，没有 `system`，也不会回落到明文 DNS。酒店、机场等 Wi-Fi 在登录前通常只放行自家 DNS，登录页可能打不开，临时关闭 Stash 完成登录即可。

### 9. 分流模式下存在无法根治的关联风险

网站可以在页面中嵌入一个走直连的资源，从服务端把真实 IP 与你的访问关联起来。这是分流代理的固有特性，只有全局代理能规避。

---

## 三、测试前准备

- [ ] 导入最新配置，**配置能正常加载、没有报错**（报错见已知问题 1；YAML 缩进错误也会导致加载失败）
- [ ] 更新全部规则集合，在规则集合列表中核对条目数（名称与入口因版本略有差异）：

  | 规则集                | 约条目数 | 说明                                          |
  | --------------------- | -------: | --------------------------------------------- |
  | `final-direct-domain` |   11.1 万 | 显示 0 条说明 `format: yaml` 未生效           |
  | `final-direct-ip`     |    9,600 | 同上                                          |
  | `direct-local`        |      130 | 同上                                          |
  | `adblock`             |   20.2 万 | 显示 0 条或报错说明 **MRS 未能加载**          |
  | `youtube` / `github`  |  178 / 64 | MetaCubeX 纯文本规则集，作为对照              |

- [ ] 覆写列表中**没有**启用 `Stash-HTTPDNS.Block.stoverride`（见已知问题 3）
- [ ] 运行模式为**规则模式**，而不是全局代理或全局直连
- [ ] 策略组中出现 **`PROXY`**，并已在其中**重新选好节点**（组名由「手动选择」改为 `PROXY`，原来的选择记录不会保留）；「兜底线路」当前选择的是 `PROXY`
- [ ] 选好节点，确认能正常上网；第 6 步需要所选节点**支持 UDP 转发**（Clash 格式节点需 `udp: true`）
- [ ] 在 App「网络设置」中查看「启用 Tunnel IPv6 路由」的当前状态并记下（第 4 步要用）
- [ ] 记下真实 IP 备用：先关闭 Stash，访问 `ip.sb` 记录 IP，再重新开启
- [ ] 打开 Stash 的**连接**记录页，清空已有记录，方便后续查看每条连接命中的规则与策略

> ⚠️ 后续所有测试都要和这个真实 IP 做对比。检测页会显示真实 IP，截图外发前记得打码。

---

## 四、测试步骤

### 第 1 步 · 规则与规则集生效情况

**操作** 　用 Safari 依次访问下列地址，再到连接记录中查看每条连接命中的规则与策略。

**预期结果**

| 访问                          | 预期命中规则                                    | 预期策略     | 验证目的                                                 |
| ----------------------------- | ----------------------------------------------- | ------------ | -------------------------------------------------------- |
| `https://www.taobao.com`      | `RULE-SET,final-direct-domain`                  | DIRECT       | **Loyalsoldier 规则集改为 yaml 后已生效**                |
| `https://www.zhihu.com`       | `RULE-SET,final-direct-domain`                  | DIRECT       | 同上                                                     |
| `https://www.icloud.com`      | `RULE-SET,apple`                                | 苹果服务     | MetaCubeX 规则集正常                                     |
| `https://x.com`               | `RULE-SET,twitter`                              | Twitter      | 同上                                                     |
| `https://telegram.org`        | `RULE-SET,telegram`                             | Telegram     | 同上                                                     |
| `https://www.google.com`      | `RULE-SET,google`                               | 谷歌服务     | 同上                                                     |
| `https://gemini.google.com`   | `RULE-SET,gemini`                               | **AI服务**   | 本地关键词规则已注释，不再截走 gemini 规则集             |
| `https://www.youtube.com`     | `RULE-SET,youtube`                              | **YouTube**  | **youtube 已排到 google 之前**，不再被 google 规则集截走 |
| `https://github.com`          | `RULE-SET,github`                               | **兜底线路** | **github 已排到 microsoft 之前**，不再落到微软服务（直连） |
| `https://browserleaks.com`    | `DOMAIN-SUFFIX,browserleaks.com`                | **兜底线路** | 本地检测站规则生效                                       |
| `http://httpbin.org`          | `MATCH`                                         | 兜底线路     | **`GEOIP` 的 `no-resolve` 生效**，未做本地解析           |
| `http://neverssl.com`         | `MATCH`                                         | 兜底线路     | 同上                                                     |
| `https://cloudflare-quic.com` | 页面显示 **HTTP/2**，而不是 HTTP/3              | —            | **`PROTOCOL,QUIC` 生效**，浏览器已回落到 TCP             |

> 💡 `httpbin.org`、`neverssl.com` 已核实不在任何已启用的规则集中，专门用来检验「未收录域名」的走向。

> 💡 原配置中 `myip.la` 被 `final-direct-domain` 分到直连。检测站规则现在排在所有规则集之前，结果是确定的。

**不符合时**

- **导入配置时报错**：多半是 `PROTOCOL,STUN` 或 `PROTOCOL,QUIC` 未被识别，按「已知问题 1」改用备用规则。
- **`www.taobao.com`、`www.zhihu.com` 命中 `GEOIP` 或 `MATCH`**：`final-direct-domain` 未加载，回到「测试前准备」检查条目数。
- **`httpbin.org` 显示命中 `GEOIP`**：`no-resolve` 未生效，检查该行是否为 `- GEOIP,CN,DIRECT,no-resolve`。
- **`www.youtube.com` 走了谷歌服务、`github.com` 走了微软服务**：检查 `rules:` 中 `youtube` 是否在 `google` 之前、`github` 是否在 `microsoft` 之前。
- **任意请求的策略显示为某个意外节点**：检查 `PROXY` 策略组是否存在、是否已选好节点。
- **cloudflare-quic.com 显示 HTTP/3**：`PROTOCOL,QUIC` 未生效，临时去掉该行的 `no-track`，在连接记录中查看 UDP 443 连接命中了哪条规则。

<br>

### 第 2 步 · DNS 泄露测试

**操作**

1. 打开 https://browserleaks.com/dns
2. **先看页面顶部的 IP**，必须是节点 IP。若是真实 IP，说明该站没走代理，下方结果无效
3. 再看列出的 DNS 服务器

**预期结果**

| 结果                                                 | 判定                     |
| ---------------------------------------------------- | ------------------------ |
| 仅节点那一侧的解析服务器（如 Cloudflare、Google 等） | ✅ 通过                   |
| 出现腾讯、阿里（doh.pub / AliDNS）                   | ❌ 发生了本地解析，需排查 |
| 出现当前运营商                                       | ❌ 最严重                 |

**不符合时**

1. 回到第 1 步，确认 `browserleaks.com` 走兜底线路、`httpbin.org` 命中 `MATCH`。
2. 检查覆写列表：是否启用了带 IP 规则的覆写（尤其 HTTPDNS 覆写，见已知问题 3）。覆写的规则排在最前，其中不带 `no-resolve` 的 IP 规则会让所有域名先在本地解析。
3. 出现运营商时，确认 `dns.nameserver` 中没有 `system`，且没有覆写改写了 `dns` 段（覆写中的数组会**插入到原数组之前**，带 `#!replace` 时会整体替换）。

<br>

### 第 3 步 · WebRTC 泄露测试（默认模式）

**操作**

1. 打开 https://webrtc.github.io/samples/src/content/peerconnection/trickle-ice/
2. 点 **Remove server** 删掉默认服务器
3. 填入一个 STUN 地址 › **Add Server** › **Gather candidates**
4. 查看结果中是否有 **Type 为 `srflx`** 的行
5. 删除该服务器，换下一个，重复以上步骤

```
stun:stun.l.google.com:19302
stun:stun.chat.bilibili.com:3478
stun:stun.miwifi.com:3478
stun:106.12.251.31:3478
```

> 💡 最后一个是 `stun.chat.bilibili.com` 的国内 IP，用来检验**以 IP 形式指定、端口非 19302** 的 STUN 是否也被拦截。它在 `final-direct-ip` 中，没被拦截的话会走直连暴露真实 IP。IP 可能变化：关闭 Stash 后在 Mac 终端执行 `dig +short stun.chat.bilibili.com` 获取最新 IP（开启 Stash 时会得到 `198.18.x.x` 假 IP，不能用）。

6. 再打开 https://browserleaks.com/webrtc ，查看「Public IP Address」一栏

**预期结果** 　`PROTOCOL,STUN,REJECT` 生效时，四个地址均应**没有 srflx 候选**；browserleaks 的 WebRTC 页不应显示任何公网 IP。

| Trickle ICE 结果  | 判定                                      |
| ----------------- | ----------------------------------------- |
| 无 srflx，仅 host | ✅ STUN 被拦截                             |
| 出现节点 IP       | ⚠️ STUN 未被拦截，但该路径走了代理，不泄露 |
| 出现真实 IP       | ❌ 泄露                                    |

> 💡 host 候选应为 mDNS 的 `xxxx.local` 形式，不会暴露局域网 IP。

> 💡 **原配置只拦截 19302-19309 端口**，后三个地址在原配置下都会命中直连规则、暴露真实 IP。四个地址都没有 srflx，即可确认 `PROTOCOL,STUN` 是按报文识别、与端口无关。

**不符合时**

- 确认 `rules:` 中生效的是 `- PROTOCOL,STUN,REJECT,no-track`，且其上方的通话模式行 `# - PROTOCOL,STUN,兜底线路` 仍处于注释状态。
- **只有 3478 端口或 IP 形式的 STUN 出现真实 IP**：检查是否因已知问题 1 改用了按端口匹配的备用规则；若是，需升级 Stash。
- **出现 IPv6 形式的真实地址**：见第 4 步，IPv6 流量可能没有经过 Stash。

<br>

### 第 4 步 · IPv6 泄露测试

> ⚠️ 必须在**确实提供 IPv6 的网络**下测试，否则结果无意义。

**操作**

1. **先关闭 Stash**，打开 https://test-ipv6.com ，确认能看到 IPv6 地址，以此证明该网络有 IPv6
2. 开启 Stash，保持「启用 Tunnel IPv6 路由」**关闭**（默认状态），重新测试 test-ipv6.com，并重做第 3 步中 browserleaks 的 WebRTC 检测
3. 打开「启用 Tunnel IPv6 路由」，重复第 2 步

**预期结果**

| 结果                              | 判定              |
| --------------------------------- | ----------------- |
| 无 IPv6 地址，或仅显示节点的 IPv6 | ✅ 通过            |
| 出现不属于节点的 IPv6 地址        | ❌ IPv6 绕过了隧道 |

> 💡 官方文档说明：Stash Tunnel 默认**仅启用 IPv4**，大多数请求因 Fake-IP 与 HTTP 代理机制不受影响；但**直接以 IPv6 地址访问**的流量，在开关关闭时不会进入 Stash。test-ipv6.com 的「无 DNS 的 IPv6 连接」一项就属于这种情况，是本步最需要关注的结果。

**不符合时** 　开关关闭时出现真实 IPv6、打开后消失，说明需要常开该开关；打开后若出现无法联网等兼容性问题（官方提示网络不支持 IPv6 时开启可能导致兼容性问题），则只在提供 IPv6 的网络下开启。结论记入第六节。

<br>

### 第 5 步 · 通话功能验证（默认模式）

**操作** 　在默认配置下各试一次，记录能否接通、音视频是否正常：

- [ ] 微信语音 / 视频
- [ ] FaceTime
- [ ] Telegram 语音
- [ ] WhatsApp 语音
- [ ] Google Meet（网页或 App）

**预期结果** 　微信走直连、协议自有，应正常；其余通话依赖 STUN 的程度不同，**可能接不通、只能单向、或延迟明显变高**，这是默认模式拦截 STUN 的已知代价，不算测试失败，只需记录结果，作为日后是否常开通话模式的依据。

> 💡 建议无论哪种模式都打开 App 自身的防护：Telegram「隐私与安全 › 通话 › 点对点 › 永不」；WhatsApp「隐私 › 高级 › 在通话中保护 IP 地址」。

<br>

### 第 6 步 · 通话模式验证（可选）

**操作**

1. 在 `config.yaml` 的 `rules:` 中放开**一行**：`# - PROTOCOL,STUN,兜底线路` → `- PROTOCOL,STUN,兜底线路`。其下方的 `- PROTOCOL,STUN,REJECT,no-track` **保持不动**
2. 重新加载配置，重复第 3 步的 Trickle ICE 测试（含 IP 形式的那一个）
3. 在连接记录中查看各 STUN 连接命中的规则
4. 重复第 5 步中失败的通话项
5. **（可选，核实已知问题 7）** 把「兜底线路」临时切到一个**不支持 UDP** 的节点，重做 Trickle ICE：应**没有 srflx**（被下方 REJECT 兜住），而不是出现真实 IP
6. **测试结束后改回默认模式**（重新注释该行），并重新执行一次第 3 步，确认恢复为无 srflx

**预期结果**

| 检查项                          | 预期                                     |
| ------------------------------- | ---------------------------------------- |
| 四个 STUN 地址的 srflx          | **均为节点 IP**（与 ip.sb 一致）         |
| 四个 STUN 连接在记录中的规则    | 均命中 `PROTOCOL,STUN` → 兜底线路        |
| 第 5 步中失败的通话             | 恢复正常                                 |
| 不支持 UDP 的节点（可选项）     | 无 srflx；若出现真实 IP，说明 REJECT 未兜住，须立即改回默认模式并反馈 |

> 💡 Stash 严格从上到下匹配，协议规则排在最前，所以**不需要**像 Loon、QuantumultX 那样另加 STUN 的 `DOMAIN` 规则：`stun.chat.bilibili.com` 不会先被 bilibili 规则集接走，IP 形式的 STUN 也不会先命中 `final-direct-ip`。

**不符合时**

- **某个 STUN 出现真实 IP**：在连接记录中查看它命中的规则。若命中的是 bilibili、`final-direct-ip` 等直连规则，说明通话模式那一行没有生效，或被放到了其它规则之后。
- **通话仍失败**：检查所选节点是否支持 UDP 转发（`udp: true`）；不支持时 STUN 会被下方的 REJECT 拦截，通话失败。

<br>

### 第 7 步 · 日常功能回归

| 测试项                                | 预期结果                                                                    |
| ------------------------------------- | --------------------------------------------------------------------------- |
| 淘宝、B站、微信、支付宝、知乎         | 正常且快，走直连（B站走「哔哩哔哩」组，默认 DIRECT）                        |
| Google、YouTube、Twitter              | 正常，走代理；YouTube 走「YouTube」组                                       |
| GitHub（网页、`git clone`）           | 正常，走「兜底线路」                                                        |
| App Store、iCloud                     | 正常，走「苹果服务」（默认 DIRECT）                                         |
| AI 服务（ChatGPT、Claude、Gemini）    | 走「AI服务」策略组                                                          |
| TikTok、PayPal、加密货币交易所        | 走各自设定的策略组，账户无异常提示                                          |
| 切换 `PROXY` 组中的节点               | 默认选 `PROXY` 的策略组（兜底线路、谷歌服务等）随之切换                     |
| 随机打开几个冷门国内网站              | 能打开；若走了兜底线路导致变慢，按需单独加 DIRECT 规则                      |
| 常用 App 的广告                       | 开屏、信息流广告明显减少；某功能异常时，在连接记录中查看是否被 `adblock` 误拦 |
| 公共 Wi-Fi 登录页                     | 打不开时临时关闭 Stash 完成登录                                             |
| 小米路由器管理页 `miwifi.com`         | 连接小米路由器 Wi-Fi 时能正常打开                                           |
| Stash 运行一整天                      | 未被系统强制关闭（iOS 限制 Stash 内存 50 MB）                               |

---

## 五、注意事项

### 判定顺序不能颠倒：先确认 IP，再看 DNS

1. 先看检测页顶部的 IP 是不是节点 IP。**若是真实 IP，说明该站没走代理，后面的 DNS 结论一律无效。**
2. IP 正确的前提下，再看解析服务器。

### 「出现腾讯、阿里」是否算泄露，取决于域名本该走哪条路

| 域名类型                   | 解析来源   | 判定                                 |
| -------------------------- | ---------- | ------------------------------------ |
| 国内直连域名（淘宝、京东） | 腾讯、阿里 | ✅ 正常，配置本就如此，可分到就近 CDN |
| 走代理的境外域名           | 节点那一侧 | ✅ 正常，代理域名在远端解析           |
| 走代理的境外域名           | 腾讯、阿里 | ❌ 异常，说明触发了本地解析           |
| 任意域名                   | 当前运营商 | ❌ 最严重                             |

**原理** 　Fake-IP 模式下，App 的 DNS 查询由 Stash 直接返回假 IP，不产生上游查询；随后按分流规则决定走向——走代理时把**域名**交给节点解析，走直连时才用 `nameserver`（doh.pub、AliDNS）在本机解析。

**例外** 　以下情况本机仍会发起查询：

- 域名请求走到了不带 `no-resolve` 的 IP 类规则（已核实：当前 3 条 IP 类规则 `direct-lan`、`final-direct-ip`、`GEOIP` 均带 `no-resolve`）
- `fake-ip-filter` 列表内的域名（已移除 5 条 Google / golang 境外域名）
- 节点域名自身的解析（`proxy-server-nameserver`，两个 DoH）
- DoH 服务器自身域名的引导解析（`default-nameserver`，223.5.5.5、119.29.29.29 明文，仅查询 `doh.pub`、`dns.alidns.com`）
- 启用了带不含 `no-resolve` IP 规则的覆写
- 运行模式切到全局直连，或「兜底线路」被切到 `DIRECT`

### 规则严格从上到下匹配，顺序就是优先级

Stash 不区分本地与远程，也不按规则类型分档，第一条命中的规则即生效。因此：

- **宽泛规则会截走后面的规则集**：原配置的 `DOMAIN-KEYWORD,google` 把 gemini 规则集的 Gemini 域名分到谷歌服务，已注释。
- **公司级规则集会截走子品牌规则集**：MetaCubeX 的 `google` 包含 `youtube` 的全部 178 条，`microsoft` 包含 `github` 的全部 64 条，所以子品牌规则集必须排在前面。新增规则集时同样要注意。
- **覆写的规则永远排在最前**：启用任何覆写前，先确认其中的 IP 类规则都带 `no-resolve`。

### 解析服务器的 IPv6 地址不是泄露

browserleaks 的 DNS 结果里出现 `2400:cb00::`、`2607:f8b0::`、`2620:171::` 等地址，那是**解析服务器自身的 IPv6**，与你设备的 IPv6 无关。

---

## 六、测试进度与结果

> 测试环境：待填写 ｜ 设备 ｜ 所在地、本地网络、是否提供 IPv6、代理节点、Stash 版本

| 测试项                   | iPhone |  Mac  | 说明 |
| ------------------------ | :----: | :---: | ---- |
| 第 1 步 规则与规则集     |   ⬜    |   ⬜   |      |
| 第 2 步 DNS 泄露         |   ⬜    |   ⬜   |      |
| 第 3 步 WebRTC 泄露      |   ⬜    |   ⬜   |      |
| 第 4 步 IPv6 泄露        |   ⬜    |   ⬜   |      |
| 第 5 步 通话功能         |   ⬜    |   ⬜   |      |
| 第 6 步 通话模式（可选） |   ⬜    |   ⬜   |      |
| 第 7 步 日常回归         |   ⬜    |   ⬜   |      |

<small>✅ 通过 ｜ ⏸ 待补测 ｜ ⬜ 待测</small>

---

## 七、待办清单

- [x] **完成 `config.yaml` 的安全防护优化** —— 2026-09-27（规则集格式与 `no-resolve`、STUN / QUIC 协议拦截、检测站规则、`PROXY` 策略组、通话模式、DNS 解析器、规则集顺序、广告规则集改为 MRS）
- [ ] **确认 `PROTOCOL,STUN`、`PROTOCOL,QUIC` 被识别** —— 配置加载无报错，第 1 步 HTTP/3 检测、第 3 步四个地址均无 srflx
- [ ] **确认规则集加载** —— `final-direct-domain` 约 11.1 万条、`adblock`（MRS）约 20.2 万条
- [ ] **确认 `GEOIP,CN,DIRECT,no-resolve` 生效** —— 第 1 步 `httpbin.org` 命中 `MATCH`
- [ ] **确定「启用 Tunnel IPv6 路由」是否常开** —— 第 4 步，需在提供 IPv6 的网络下进行，如国内宽带或蜂窝数据
- [ ] **核实节点不支持 UDP 时的行为** —— 第 6 步可选项
- [ ] **第 1～7 步（iPhone）**
- [ ] **Mac 端完整跑一遍** —— 第 1～7 步
- [ ] **根据第 5、6 步结果决定默认模式** —— 若通话模式稳定且 REJECT 保险可靠，可评估将通话模式作为默认：既不泄露真实 IP，通话也正常

---

## 附录 分流规则匹配与 DNS 解析过程

> 本节说明 Stash 如何为一个请求选择规则、何时会发起本地 DNS 解析，以及 `GEOIP,CN,DIRECT,no-resolve` 为什么能防止 DNS 泄漏。
> 依据：官方文档 [stash.wiki](https://stash.wiki/)（规则类型、规则集合、DNS 服务、覆写文件、IPv6 兼容性）。

### 1. Stash 平时并不做 DNS 解析（Fake-IP）

App 访问 `x.com` 时，先向系统查询它的 IP。Stash 不去真实解析，而是直接回给 App 一个假 IP（`198.18.x.x`），并记下「这个假 IP 对应 `x.com`」。App 拿假 IP 发起连接，Stash 截住连接，查出域名 `x.com`，再去匹配分流规则。

**到这一步，还没有任何 DNS 查询发出去。**

> `fake-ip-filter` 中的域名例外：它们不用假 IP，App 查询时 Stash 就会用 `nameserver` 在本地真实解析。所以该列表里不应放境外域名。
>
> 这个列表**只决定返回真实 IP 还是假 IP**，与 WebRTC 防护无关：两种情况下连接都会进入 Stash 按规则分流。STUN 域名保留假 IP 反而更好，Stash 能按域名分流。

### 2. 不同类型的规则需要的信息不同

| 规则类型                                         | 判断时需要什么   | 是否需要 DNS 解析                                             |
| ------------------------------------------------ | ---------------- | ------------------------------------------------------------- |
| `DOMAIN` / `DOMAIN-SUFFIX` / `DOMAIN-KEYWORD`、`domain` 类规则集 | 只需要域名字符串 | **不需要**                                  |
| `PROTOCOL` / `DST-PORT` / `SCRIPT`（仅用端口）   | 协议、端口       | 不需要                                                        |
| `GEOIP` / `IP-CIDR` / `IP-ASN`、`ipcidr` 类规则集 | 需要真实 IP     | **需要**：域名请求必须先解析出真实 IP；带 `no-resolve` 则跳过 |
| `MATCH`                                          | 不需要任何信息   | 不需要                                                        |

一个**域名请求**只要走到不带 `no-resolve` 的 IP 类规则，Stash 就会用 `nameserver`（doh.pub 腾讯、AliDNS 阿里）在本地解析它。国内 DNS 服务商因此看到「这个用户在查某个境外域名」，这就是 **DNS 泄漏**。

### 3. 规则匹配顺序

```
覆写（stoverride）中的规则   ← 插入到最前
  ↓
config.yaml 的 rules:        ← 从上到下，第一条命中即生效
  PROTOCOL,QUIC → PROTOCOL,STUN → 检测站 → 局域网 / 广告 → 各服务规则集 → 直连规则集 → GEOIP → MATCH
```

与 Loon 的最大区别：**没有「域名类规则先匹配」这一层**。IP 类规则写在哪里，域名请求就在哪里被解析，所以不带 `no-resolve` 的 IP 规则越靠前，泄漏面越大。

### 4. 用实际请求走一遍

#### 例 1　`www.zhihu.com`：规则集收录的国内域名

```
… RULE-SET,final-direct-domain（+.zhihu.com）→ 命中 → DIRECT
```

直连需要真实 IP，Stash 用国内 DoH 解析，这是正常的：国内网站本就该用国内 DNS，可以分到就近 CDN。

#### 例 2　`x.com`：规则集收录的境外域名

```
… RULE-SET,twitter（+.x.com）→ 命中 → Twitter 策略组（代理）
```

走代理时，Stash 把域名 `x.com` 原样交给节点，**由节点在海外解析**，本地不做任何查询。

#### 例 3　`httpbin.org`：任何规则集都没收录的境外域名

**`GEOIP` 不带 `no-resolve` 时（原配置）：**

```
… 域名类规则集全部未命中
RULE-SET,final-direct-ip,no-resolve → 跳过
GEOIP,CN → 本地解析 httpbin.org（doh.pub / AliDNS）   ← ⚠️ 泄漏发生在这里
         → 美国 IP，不命中
MATCH → 兜底线路（代理）→ 节点在海外再解析一次
```

腾讯或阿里的 DNS 已看到这个境外域名；而最终走的是代理，本地这次解析的结果**根本没用上**，白白泄漏了一次。

**加上 `no-resolve` 之后：**

```
… 域名类规则集全部未命中
GEOIP,CN,DIRECT,no-resolve → 目标是域名，跳过，不解析
MATCH → 兜底线路（代理）
```

本地不做任何查询。

#### 例 4　`stun.chat.bilibili.com:3478`：网页发起的 STUN

```
原配置：SCRIPT,webrtc-stun（仅 19302-19309）→ 不命中
        … RULE-SET,bilibili → 哔哩哔哩（默认 DIRECT）→ STUN 服务器看到真实 IP   ← ⚠️ WebRTC 泄漏
现配置：PROTOCOL,STUN,REJECT → 命中 → 拦截，无 srflx
通话模式：PROTOCOL,STUN,兜底线路 → 命中 → 经节点发出，srflx 为节点 IP
```

#### 例 5　Telegram App 直连 `149.154.167.51`：纯 IP 请求

```
… 域名类规则：没有域名可比，全部跳过
RULE-SET,direct-lan / final-direct-ip（no-resolve）→ 不是局域网、国内 IP，不命中
GEOIP,CN,DIRECT,no-resolve → 本身就是 IP，照常判断 → 不是 CN，不命中
MATCH → 兜底线路（代理）
```

`no-resolve` 只跳过域名请求，纯 IP 请求照常按 IP 规则与 `GEOIP` 分流。MetaCubeX 的 `telegram` 规则集只有域名、没有 IP 段，所以 Telegram 的 IP 直连走的是兜底线路，而不是「Telegram」策略组。

#### 例 6　`xiaodian-abc.cn`：没被收录的国内小网站（副作用）

```
原配置：GEOIP,CN → 解析出国内 IP → 命中 → DIRECT
现配置：GEOIP,CN,no-resolve → 跳过 → MATCH → 兜底线路（默认代理）
```

会绕道代理，可能变慢，一般仍能打开。遇到时单独加一条 `- DOMAIN-SUFFIX,xiaodian-abc.cn,DIRECT` 即可。

### 5. 如何自行验证

在 Stash 的连接记录中查看每条连接命中的规则与策略：

| 看到的规则                                | 含义                                                                  |
| ----------------------------------------- | --------------------------------------------------------------------- |
| `RULE-SET,final-direct-domain` → DIRECT   | 被规则集收录，正常分流                                                |
| `MATCH` → 兜底线路                        | 未被收录的域名，未做本地解析                                          |
| 域名请求显示命中 `GEOIP` 或某个 IP 规则集 | ⚠️ 域名走到了 IP 类规则并被解析，检查 `no-resolve` 与已启用的覆写      |
| 找不到某条连接                            | 可能被带 `no-track` 的 REJECT 规则拦截（QUIC、STUN），见已知问题 6    |
