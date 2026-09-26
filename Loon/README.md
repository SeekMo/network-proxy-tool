# Loon 安全防护测试指南

> 本指南用于验证 `config.lcf` 优化后的 DNS / WebRTC 安全防护是否真正生效。
> 按顺序执行第 1～7 步，每一步都给出了操作方法、预期结果，以及不符合预期时的排查方向。
> 规则如何匹配、何时触发本地 DNS 解析，见 [附录 分流规则匹配与 DNS 解析过程](#附录-分流规则匹配与-dns-解析过程)。

|   #   | 测试项           | 验证目标                                                   |
| :---: | ---------------- | ---------------------------------------------------------- |
|   1   | 规则与插件       | 分流规则、`no-resolve`、新增域名规则集、HTTPDNS 插件是否生效 |
|   2   | DNS 泄露         | 域名解析是否泄露给运营商或非预期的 DNS 服务商              |
|   3   | WebRTC 泄露      | 网页能否通过 STUN 探测到真实 IP                            |
|   4   | IPv6 泄露        | IPv6 流量是否绕过隧道直接出网                              |
|   5   | 通话功能         | 默认模式（丢弃 STUN）对各类通话的影响                      |
|   6   | 通话模式（可选） | 切换到通话模式后，STUN 是否都走代理、不暴露真实 IP         |
|   7   | 日常回归         | 分流是否正确，常用应用是否正常                             |

---

## 一、与 Shadowrocket、QuantumultX 的关键差异

| 项目                  | Shadowrocket                                  | QuantumultX                                | Loon                                                                                          |
| --------------------- | --------------------------------------------- | ------------------------------------------ | --------------------------------------------------------------------------------------------- |
| 加密 DNS              | `dns-server` + 加密的 `fallback-dns-server`   | `doh-server`，**没有加密备用 DNS**         | `doh-server` + `doq-server` **并发查询**；都失败时回落到明文 `dns-server`（223.5.5.5、119.29.29.29） |
| 防止 IP 规则触发解析  | `GEOIP,CN,DIRECT,no-resolve`                  | 不支持 `no-resolve`，改用 `host-keyword, .` | `GEOIP,CN,DIRECT,no-resolve`（同 Shadowrocket）                                               |
| 规则优先级            | 从上到下                                      | 先按**类型**分档，同档内才看本地/远程      | 域名类先于 IP 类；来源 **本地 > 插件 > 订阅**，同来源内从上到下                               |
| 不使用 Fake-IP 的域名 | `always-real-ip`                              | `dns_exclusion_list`                       | `real-ip`（三者内容已同步，142 条）                                                           |
| WebRTC / STUN         | `stun-response-ip` 返回假 IP                  | `udp_drop_list` 丢弃 STUN                  | `disable-stun = true` 丢弃 STUN（**没有** `stun-response-ip`）                                |
| 通话模式 STUN 覆盖    | 域名规则 + 12 条 UDP 端口规则                 | 仅域名规则                                 | **`PROTOCOL,STUN`**（不限域名 / IP / 端口）+ 域名规则                                         |
| 节点不支持 UDP 时     | `udp-policy-not-supported-behaviour = REJECT` | `fallback_udp_policy = reject`             | `udp-fallback-mode = REJECT`                                                                  |
| 硬编码 DNS            | `hijack-dns` 列出 11 个 IP                    | 无对应配置                                 | `hijack-dns = *:53`（所有目标的 53 端口）                                                     |
| HTTPDNS 拦截          | 模块                                          | 远程重写，只有 URL 重写                    | 插件：21 条域名规则 + 24 条 IP 规则（均带 `no-resolve`）+ URL 重写                            |
| 主代理 `PROXY`        | 内置                                          | 内置 `proxy`                               | **没有内置**，配置中自定义了 `PROXY` 策略组                                                   |

> 💡 **WebRTC 测试的预期结果与 Shadowrocket 不同，与 QuantumultX 相同。** Loon 直接丢弃 STUN，Trickle ICE 里应表现为**没有 srflx 候选**，而不是显示某个假 IP。

---

## 二、已知问题与限制

### 1. `GEOIP,CN,DIRECT,no-resolve` 的单行写法没有官方示例

官方文档只在逻辑规则示例里出现过 `(GEOIP,CN,no-resolve)`，IP-ASN 则有单行示例。第 1 步会验证它是否生效；若导入报错或不生效，改用文档示例同款写法：

```
OR,((GEOIP,CN,no-resolve)),DIRECT
```

### 2. 规则集会把检测站分到直连

`China_Domain.list` 收录了 `browserleaks.com`、`myip.la`、`whatismyip.com`。已在 `[Rule]` 加入 10 条 `DOMAIN-SUFFIX` 规则指向「兜底线路」。Loon 本地规则优先于订阅规则，结果是确定的（QuantumultX 需实测确认）。

### 3. 明文 DNS 回落

加密 DNS（DoH、DoQ）全部失败时，Loon 会回落到 `dns-server` 的 223.5.5.5、119.29.29.29 做明文查询（已去掉 `system`，不会交给运营商）。回落时只影响**直连域名与 `real-ip` 域名**，走代理的域名由节点远端解析。如需彻底禁止，可在 App「DNS 服务器」页关闭「查询回落」（单设备设置，不随配置同步）。

需网页登录的公共 Wi-Fi 在登录前通常只放行自家 DNS，登录页可能打不开，临时关闭 Loon 完成登录即可。

### 4. 默认模式会影响部分通话

`disable-stun = true` 丢弃所有 STUN 报文，ICE 连通检测与 UDP TURN 也会被一并丢弃。需要通话时切换到「通话模式」，见第 6 步。

### 5. 未收录的国内小站会走代理

这是 `no-resolve` 防 DNS 泄漏的代价。遇到时在 `[Rule]` 为其单独加一条 `DOMAIN-SUFFIX,域名,DIRECT`。

### 6. `hijack-dns = *:53` 会影响 DNS 测试类 App

所有发往 53 端口的 UDP 查询都会拿到 Fake IP（`198.18.x.x`），dig、nslookup 类 App 的结果会失真，测试时临时关闭 Loon。局域网 DNS 在 `bypass-tun` 中，不受影响。

### 7. 分流模式下存在无法根治的关联风险

网站可以在页面中嵌入一个走直连的资源，从服务端把真实 IP 与你的访问关联起来。这是分流代理的固有特性，只有全局代理能规避。

---

## 三、测试前准备

- [ ] Loon 版本 **≥ 3.2.5 (789)**（`hijack-dns` 的最低版本）
- [ ] 导入最新配置，**更新全部远程资源**（订阅、规则集、插件），让新增的规则集重新下载
- [ ] 在远程规则列表中确认各规则集均已加载成功（显示规则条数），**尤其是新增的两个**：
  - 「国内直连域名」（China_Domain.list）约 **3689** 条
  - 「苹果服务域名」（Apple_Domain.list）约 **1560** 条
  - 显示 0 条说明 `.example.com` 格式未被识别，须先排查
- [ ] 插件列表中「HTTPDNS拦截器」为**启用**状态
- [ ] 运行模式为**规则模式**，而不是全局代理或全局直连
- [ ] 策略组中出现 **`PROXY`**，并已在其中选好节点；「兜底线路」当前选择的是 `PROXY`，而不是 `DIRECT`
- [ ] 选好节点，确认能正常上网；第 6 步需要所选节点**支持 UDP 转发**，且订阅**开启 UDP**（`udp = true`）
- [ ] 记下真实 IP 备用：先关闭 Loon，访问 `ip.sb` 记录 IP，再重新开启
- [ ] 打开 Loon 的**请求记录**（仪表 › 最近请求，不同版本名称略有差异），清空已有记录，方便后续查看

> ⚠️ 后续所有测试都要和这个真实 IP 做对比。检测页会显示真实 IP，截图外发前记得打码。

---

## 四、测试步骤

### 第 1 步 · 规则与插件生效情况

**操作** 　用 Safari 依次访问下列地址，再到请求记录中查看每条请求命中的规则与策略。

**预期结果**

| 访问                                   | 预期命中规则                                | 预期策略     | 验证目的                                        |
| -------------------------------------- | ------------------------------------------- | ------------ | ----------------------------------------------- |
| `https://www.taobao.com`               | `DOMAIN-KEYWORD,taobao`（China.list）       | DIRECT       | 远程 China.list 正常                            |
| `https://www.zhihu.com`                | `DOMAIN-SUFFIX,zhihu.com`（China_Domain）   | DIRECT       | **新增的 China_Domain.list 已生效**             |
| `https://www.icloud.com`               | `DOMAIN-SUFFIX,icloud.com`（Apple_Domain）  | 苹果服务     | **新增的 Apple_Domain.list 已生效**             |
| `https://x.com`                        | `DOMAIN-SUFFIX,x.com`（Twitter.list）       | Twitter      | 远程分类规则集正常                              |
| `https://telegram.org`                 | `DOMAIN-SUFFIX,telegram.org`（Telegram.list） | Telegram   | 同上                                            |
| `https://www.google.com`               | `DOMAIN-SUFFIX,google.com`（Google.list）   | 谷歌服务     | 远程分类规则集正常                              |
| `https://gemini.google.com`            | `DOMAIN,gemini.google.com`（AI.list）       | **AI服务**   | 本地关键词规则已注释，不再截走 AI.list          |
| `https://browserleaks.com`             | 本地 `DOMAIN-SUFFIX,browserleaks.com`       | **兜底线路** | 本地检测站规则压过 China_Domain.list            |
| `http://httpbin.org`                   | `FINAL`                                     | 兜底线路     | **`GEOIP` 的 `no-resolve` 生效**，未做本地解析 |
| `http://neverssl.com`                  | `FINAL`                                     | 兜底线路     | 同上                                            |
| `http://119.29.29.29/d?dn=www.qq.com`  | 插件 URL 重写 reject                        | 被拒绝       | HTTPDNS 插件生效（明文 HTTP，无需 MitM）        |

> 💡 `httpbin.org` 与 `neverssl.com` 已核实不在任何已启用的规则集和插件中，专门用来检验「未收录域名」的走向。

> 💡 `browserleaks.com` 若显示 DIRECT，说明命中的是 China_Domain.list 而非本地规则，第 2 步结果将无效。

**不符合时**

- **导入配置时报错，或 `httpbin.org` 显示命中 `GEOIP`**：`no-resolve` 的单行写法未被识别，按「已知问题 1」改用 `OR,((GEOIP,CN,no-resolve)),DIRECT`。
- **`www.zhihu.com` 走了兜底线路**：China_Domain.list 未加载，回到「测试前准备」检查规则条数。
- **`www.icloud.com` 走了兜底线路**：Apple_Domain.list 未加载，同上。
- **`browserleaks.com` 为 DIRECT**：检查 `[Rule]` 中 10 条检测站规则是否存在且未被注释。
- **任意请求的策略显示为某个意外节点**：检查 `PROXY` 策略组是否存在、是否已选好节点。
- **HTTPDNS 地址能返回 IP 列表**：检查「HTTPDNS拦截器」插件是否启用、是否更新成功。

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

**不符合时** 　① 回到第 1 步，确认 `browserleaks.com` 为兜底线路、`httpbin.org` 命中 FINAL；② 在请求记录中查找 `browserleaks` 相关域名的记录，看命中了哪条规则；③ 出现运营商时，确认 `dns-server` 中没有 `system`，且没有启用的插件在 `[General]` 中改写了 `dns-server`。

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
stun:106.12.71.140:3478
```

> 💡 最后一个是 `stun.chat.bilibili.com` 的国内 IP，用来检验**以 IP 形式指定、端口非 19302** 的 STUN 是否也被丢弃。IP 可能变化：关闭 Loon 后在 Mac 终端执行 `dig +short stun.chat.bilibili.com` 获取最新 IP（开启 Loon 时会得到 `198.18.x.x` 假 IP，不能用）。

6. 再打开 https://browserleaks.com/webrtc ，查看「Public IP Address」一栏

**预期结果** 　`disable-stun = true`，四个地址均应**没有 srflx 候选**；browserleaks 的 WebRTC 页不应显示任何公网 IP。

| Trickle ICE 结果  | 判定                                      |
| ----------------- | ----------------------------------------- |
| 无 srflx，仅 host | ✅ STUN 被丢弃                             |
| 出现节点 IP       | ⚠️ STUN 未被丢弃，但该路径走了代理，不泄露 |
| 出现真实 IP       | ❌ 泄露                                    |

> 💡 host 候选应为 mDNS 的 `xxxx.local` 形式，不会暴露局域网 IP。

> 💡 官方文档未说明 `disable-stun` 是按报文内容识别还是按端口识别。**19302 与 3478 两个端口都没有 srflx，即可确认是按报文识别。**

**不符合时**

- 确认 `[General]` 中生效的是 `disable-stun = true`，而不是通话模式的 `disable-stun = false`。
- **只有 3478 端口或 IP 形式的 STUN 出现真实 IP**：`disable-stun` 可能只认部分端口，在 `[Rule]` 最上方加一条 `PROTOCOL,STUN,REJECT` 后重测。

<br>

### 第 4 步 · IPv6 泄露测试

> ⚠️ 必须在**确实提供 IPv6 的网络**下测试，否则结果无意义。

**操作**

1. **先关闭 Loon**，打开 https://test-ipv6.com ，确认能看到 IPv6 地址，以此证明该网络有 IPv6
2. 开启 Loon，重新测试

**预期结果**

| 结果                              | 判定              |
| --------------------------------- | ----------------- |
| 无 IPv6 地址，或仅显示节点的 IPv6 | ✅ 通过            |
| 出现不属于节点的 IPv6 地址        | ❌ IPv6 绕过了隧道 |

> 💡 `ip-mode = ipv4-only`：不查询 AAAA 记录，并拒绝 IPv6 连接。

<br>

### 第 5 步 · 通话功能验证（默认模式）

**操作** 　在默认配置下各试一次，记录能否接通、音视频是否正常：

- [ ] 微信语音 / 视频
- [ ] FaceTime
- [ ] Telegram 语音
- [ ] WhatsApp 语音
- [ ] Google Meet（网页或 App）

**预期结果** 　微信走直连、协议自有，应正常；其余通话依赖 STUN 的程度不同，**可能接不通、只能单向、或延迟明显变高**，这是默认模式丢弃 STUN 的已知代价，不算测试失败，只需记录结果，作为日后是否常开通话模式的依据。

> 💡 建议无论哪种模式都打开 App 自身的防护：Telegram「隐私与安全 › 通话 › 点对点 › 永不」；WhatsApp「隐私 › 高级 › 在通话中保护 IP 地址」。

<br>

### 第 6 步 · 通话模式验证（可选）

**操作**

1. 在 `config.lcf` 中切换到通话模式，**两处同时改**：
   - `[General]`：注释 `disable-udp-ports = 443,80` 与 `disable-stun = true`，放开下方的 `# disable-udp-ports = 80` 与 `# disable-stun = false`
   - `[Rule]`：放开「通话模式配套」下的 11 行规则（`PROTOCOL,STUN`、8 条 `DOMAIN`、`DOMAIN-KEYWORD,stun`、`PROTOCOL,QUIC,REJECT`）
2. 更新配置，重复第 3 步的 Trickle ICE 测试（含 IP 形式的那一个）
3. 在请求记录中查看各 STUN 请求命中的规则
4. 重复第 5 步中失败的通话项
5. **测试结束后改回默认模式**，并重新执行一次第 3 步，确认恢复为无 srflx

**预期结果**

| 检查项                         | 预期                                                  |
| ------------------------------ | ----------------------------------------------------- |
| 四个 STUN 地址的 srflx         | **均为节点 IP**（与 ip.sb 一致）                      |
| `stun.chat.bilibili.com`       | 命中 `DOMAIN,stun.chat.bilibili.com` → 兜底线路       |
| `stun.miwifi.com`              | 命中 `DOMAIN,stun.miwifi.com` → 兜底线路              |
| `106.12.71.140:3478`（IP 形式）| 命中 `PROTOCOL,STUN` → 兜底线路                       |
| 第 5 步中失败的通话            | 恢复正常                                              |

> 💡 IP 形式的 STUN 是 Loon 独有的检验项：它没有域名，只能靠 `PROTOCOL,STUN` 接住；否则会命中 `GEOIP,CN` 走直连并暴露真实 IP。QuantumultX 无法覆盖这种情况。

**不符合时**

- **IP 形式的 STUN 出现真实 IP**：`PROTOCOL,STUN` 未生效，在请求记录中确认它命中了哪条规则。
- **某个 STUN 域名出现真实 IP**：它未被 `DOMAIN` 规则或 `PROTOCOL,STUN` 接住，在请求记录中查看命中的规则。若是 `stun.miwifi.com`，检查 `real-ip` 中是否又出现了 `*.miwifi.com`（现已收窄为 `miwifi.com, www.miwifi.com`）。
- **通话仍失败**：检查所选节点是否支持 UDP 转发、订阅是否开启 UDP（`udp = true`）；不满足时受 `udp-fallback-mode = REJECT` 影响会直接失败。
- **观察项（非泄漏）**：打开 YouTube，在请求记录中查看是否有 UDP 443 请求走了代理。`PROTOCOL,QUIC,REJECT` 受「域名规则优先」影响，可能拦不住发往已收录域名的 QUIC。

<br>

### 第 7 步 · 日常功能回归

| 测试项                                | 预期结果                                                                         |
| ------------------------------------- | -------------------------------------------------------------------------------- |
| 淘宝、B站、微信、支付宝、知乎         | 正常且快，走直连（B站走「哔哩哔哩」组，默认 DIRECT）                             |
| Google、YouTube、Twitter              | 正常，走代理                                                                     |
| App Store、iCloud                     | 正常，走「苹果服务」（默认 DIRECT）                                              |
| AI 服务（ChatGPT、Claude、Gemini）    | 走「AI服务」策略组                                                               |
| TikTok、PayPal、加密货币交易所        | 走各自设定的策略组，账户无异常提示                                               |
| 切换 `PROXY` 组中的节点               | 默认选 `PROXY` 的策略组（兜底线路、谷歌服务等）随之切换                          |
| 随机打开几个冷门国内网站              | 能打开；若走了兜底线路导致变慢，按需单独加 DIRECT 规则                           |
| 使用硬编码 DNS 的 App                 | 正常（`hijack-dns = *:53` 返回 Fake IP 后照常按域名分流）                        |
| 公共 Wi-Fi 登录页                     | 打不开时临时关闭 Loon 完成登录                                                   |
| 节点延迟测试                          | 各节点能正常显示延迟（测速地址为 `www.gstatic.com/generate_204`）                |
| 网络可用性检测                        | 无「网络不可用」误报（检测地址为 `www.qualcomm.cn/generate_204`）                |
| AirPlay 投屏、NAS、打印机等局域网设备 | 正常；私有网段已加入 `bypass-tun`，这类流量不经过 Loon                           |
| 小米路由器管理页 `miwifi.com`         | 连接小米路由器 Wi-Fi 时能正常打开                                                |

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

**原理** 　Fake-IP 模式下，App 的 DNS 查询由 Loon 直接返回假 IP，不产生上游查询；随后按分流规则决定走向——走代理时把**域名**交给节点解析，走直连时才用 `doh-server` / `doq-server`（doh.pub、AliDNS）在本机解析。

**例外** 　以下情况本机仍会发起查询：

- 域名请求走到了不带 `no-resolve` 的 IP 类规则（已核实：当前所有 IP 类规则均带 `no-resolve`）
- `real-ip` 列表内的域名
- 节点域名自身的解析
- 运行模式切到全局直连，或「兜底线路」被切到 `DIRECT`
- 加密 DNS 全部失败时，回落到明文 `dns-server`

### 本地规则优先于订阅规则（与 QuantumultX 不同）

Loon 的规则来源优先级为**本地 > 插件 > 订阅**，与规则类型无关。因此检测站、通话模式的本地规则一定生效；反过来，本地的宽泛规则也会截走远程规则集的正确分流。

例：原配置的本地 `DOMAIN-KEYWORD,apple / google / youtube` 会把 AI.list 的 Gemini、Apple 智能域名分到谷歌服务、苹果服务（DIRECT），把 Advertising.list 的 `applovin`、`googleads` 从广告拦截中放过，把 Global.list 的 `appledaily` 分到直连。这 3 条现已注释，由远程规则集负责。

### 解析服务器的 IPv6 地址不是泄露

browserleaks 的 DNS 结果里出现 `2400:cb00::`、`2607:f8b0::`、`2620:171::` 等地址，那是**解析服务器自身的 IPv6**，与你设备的 IPv6 无关。

---

## 六、测试进度与结果

> 测试环境：待填写 ｜ 设备 ｜ 所在地、本地网络、是否提供 IPv6、代理节点

| 测试项                   | iPhone |  Mac  | 说明 |
| ------------------------ | :----: | :---: | ---- |
| 第 1 步 规则与插件       |   ⬜    |   ⬜   |      |
| 第 2 步 DNS 泄露         |   ⬜    |   ⬜   |      |
| 第 3 步 WebRTC 泄露      |   ⬜    |   ⬜   |      |
| 第 4 步 IPv6 泄露        |   ⬜    |   ⬜   |      |
| 第 5 步 通话功能         |   ⬜    |   ⬜   |      |
| 第 6 步 通话模式（可选） |   ⬜    |   ⬜   |      |
| 第 7 步 日常回归         |   ⬜    |   ⬜   |      |

<small>✅ 通过 ｜ ⏸ 待补测 ｜ ⬜ 待测</small>

---

## 七、待办清单

- [x] **完成 `config.lcf` 的安全防护优化** —— 2026-09-25（DNS：A1～A7；WebRTC：B1～B2；结构问题：C1～C4）
- [ ] **确认 `GEOIP,CN,DIRECT,no-resolve` 被识别** —— 第 1 步 `httpbin.org` 命中 FINAL
- [ ] **确认两个新增域名规则集已加载** —— China_Domain 约 3689 条、Apple_Domain 约 1560 条
- [ ] **确认 `disable-stun` 按报文识别** —— 第 3 步 19302、3478 与 IP 形式均无 srflx
- [ ] **确认 `hijack-dns = *:53` 未误伤直连** —— 第 7 步
- [ ] **IPv6 泄露测试** —— 需在提供 IPv6 的网络下进行，如国内宽带或蜂窝数据
- [ ] **第 1～7 步（iPhone）**
- [ ] **Mac 端完整跑一遍** —— 第 1～7 步（如在 Mac 上使用 Loon）
- [ ] **根据第 5、6 步结果决定默认模式** —— 若 `PROTOCOL,STUN` 在通话模式下稳定生效，Loon 可评估将通话模式作为默认：既不泄露真实 IP，通话也正常

---

## 附录 分流规则匹配与 DNS 解析过程

> 本节说明 Loon 如何为一个请求选择规则、何时会发起本地 DNS 解析，以及 `GEOIP,CN,DIRECT,no-resolve` 为什么能防止 DNS 泄漏。
> 依据：官方文档 [nsloon.app/docs](https://nsloon.app/docs/intro)（规则、IP 规则、DNS、通用配置）与 [LoonManual](https://github.com/Loon0x00/LoonManual)。

### 1. Loon 平时并不做 DNS 解析（Fake-IP）

App 访问 `x.com` 时，先向系统查询它的 IP。Loon 不去真实解析，而是直接回给 App 一个假 IP（`198.18.x.x`），并记下「这个假 IP 对应 `x.com`」。App 拿假 IP 发起连接，Loon 截住连接，查出域名 `x.com`，再去匹配分流规则。

**到这一步，还没有任何 DNS 查询发出去。**

> `real-ip` 中的域名例外：它们不用假 IP，App 查询时 Loon 就会用加密 DNS 在本地真实解析。所以该列表里不应放境外域名（已删除 5 条 Google / golang 条目）。
>
> `hijack-dns = *:53` 让绕过系统、直接向 `8.8.8.8:53` 等发查询的 App 也拿到假 IP，同样进入上述流程。

### 2. 不同类型的规则需要的信息不同

| 规则类型                                         | 判断时需要什么   | 是否需要 DNS 解析                                         |
| ------------------------------------------------ | ---------------- | --------------------------------------------------------- |
| `DOMAIN` / `DOMAIN-SUFFIX` / `DOMAIN-KEYWORD`    | 只需要域名字符串 | **不需要**                                                |
| `PROTOCOL` / `DEST-PORT`                         | 协议、端口       | 不需要                                                    |
| `GEOIP` / `IP-CIDR` / `IP-CIDR6` / `IP-ASN`      | 需要真实 IP      | **需要**：域名请求必须先解析出真实 IP；带 `no-resolve` 则跳过 |
| `FINAL`                                          | 不需要任何信息   | 不需要                                                    |

一个**域名请求**只要走到不带 `no-resolve` 的 IP 类规则，Loon 就会用 `doh-server`（doh.pub 腾讯、alidns 阿里）在本地解析它。国内 DNS 服务商因此看到「这个用户在查某个境外域名」，这就是 **DNS 泄漏**。

### 3. 规则匹配优先级（Loon 3.0.3+）

```
目标为域名时：
  ① 域名类规则   本地 [Rule] → 插件 → 订阅 [Remote Rule]（各自从上到下）
  ② 均未命中 → 在本地解析 DNS → 匹配 IP 类规则（带 no-resolve 的跳过）
  ③ FINAL
其他规则（PROTOCOL、端口、逻辑规则等）按书写顺序，本地 > 插件 > 订阅
```

与 QuantumultX 的最大区别：**不按规则类型分档**，本地规则整体优先于订阅规则。

### 4. 用实际请求走一遍

#### 例 1　`www.zhihu.com`：规则集收录的国内域名

```
① DOMAIN-SUFFIX,zhihu.com（China_Domain.list）→ 命中 → DIRECT
```

直连需要真实 IP，Loon 用国内 DoH 解析，这是正常的：国内网站本就该用国内 DNS，可以分到就近 CDN。

#### 例 2　`x.com`：规则集收录的境外域名

```
① DOMAIN-SUFFIX,x.com（Twitter.list）→ 命中 → Twitter 策略组（代理）
```

走代理时，Loon 把域名 `x.com` 原样交给节点，**由节点在海外解析**，本地不做任何查询。

#### 例 3　`httpbin.org`：任何规则集都没收录的境外域名

**`GEOIP` 不带 `no-resolve` 时（原配置）：**

```
① 域名类全部未命中
② 本地解析 httpbin.org → 用 doh.pub / alidns   ← ⚠️ 泄漏发生在这里
   GEOIP,CN → 美国 IP，不命中
③ FINAL → 兜底线路（代理）→ 节点在海外再解析一次
```

腾讯或阿里的 DNS 已看到这个境外域名；而最终走的是代理，本地这次解析的结果**根本没用上**，白白泄漏了一次。

**加上 `no-resolve` 之后：**

```
① 域名类全部未命中
② GEOIP,CN,DIRECT,no-resolve → 目标是域名，跳过，不解析
③ FINAL → 兜底线路（代理）
```

本地不做任何查询。

#### 例 4　Telegram App 直连 `149.154.167.51`：纯 IP 请求

```
① 域名类：没有域名可比，全部跳过
② IP-CIDR,149.154.160.0/20,no-resolve（Telegram.list）→ 命中 → Telegram 策略组
```

本身就是 IP，无需解析；`no-resolve` 只跳过域名请求，纯 IP 请求照常按 IP 规则与 `GEOIP` 分流。

#### 例 5　`xiaodian-abc.cn`：没被收录的国内小网站（副作用）

```
原配置：② 解析出国内 IP → GEOIP,CN 命中 → DIRECT
现配置：③ FINAL → 兜底线路（默认代理）
```

会绕道代理，可能变慢，一般仍能打开。遇到时单独加一条 `DOMAIN-SUFFIX,xiaodian-abc.cn,DIRECT` 即可。

### 5. 同一原理的其他应用

- **泄漏检测站规则**写在本地即可，本地优先于 China_Domain.list，结果确定。
- **「通话模式配套」为什么既有 `PROTOCOL,STUN` 又有 `DOMAIN` 规则**：目标为域名时 Loon 先匹配域名类规则。`stun.chat.bilibili.com` 若只靠 `PROTOCOL,STUN`，可能先被 BiliBili.list 的 `DOMAIN-SUFFIX,bilibili.com` 接走，走直连并暴露真实 IP；本地的精确 `DOMAIN` 规则可确保它走代理。IP 形式的 STUN 没有域名，则由 `PROTOCOL,STUN` 接住。
- **已核实**：所有已启用的远程规则集与插件共 770 条 IP 类规则，全部带 `no-resolve`，本地 `GEOIP` 是唯一曾触发本地解析的规则。

### 6. 如何自行验证

在 Loon 的请求记录中查看每条请求命中的规则与策略：

| 看到的规则                          | 含义                                                                        |
| ----------------------------------- | --------------------------------------------------------------------------- |
| `DOMAIN-SUFFIX,zhihu.com` → DIRECT  | 被规则集收录，正常分流                                                      |
| `FINAL` → 兜底线路                  | 未被收录的域名，未做本地解析                                                |
| 域名请求显示命中 `GEOIP` 或 `IP-CIDR` | ⚠️ 域名走到了 IP 类规则并被解析，检查 `no-resolve` 是否生效（见已知问题 1） |
