# QuantumultX 安全防护测试指南

> 本指南用于验证 `config.conf` 优化后的 DNS / WebRTC 安全防护是否真正生效。
> 按顺序执行第 1～7 步，每一步都给出了操作方法、预期结果，以及不符合预期时的排查方向。
> 规则如何匹配、何时触发本地 DNS 解析，见 [附录 分流规则匹配与 DNS 解析过程](#附录-分流规则匹配与-dns-解析过程)。

|   #   | 测试项           | 验证目标                                           |
| :---: | ---------------- | -------------------------------------------------- |
|   1   | 规则与重写       | 分流规则、兜底规则、HTTPDNS 拦截重写是否按预期工作 |
|   2   | DNS 泄露         | 域名解析是否泄露给运营商或非预期的 DNS 服务商      |
|   3   | WebRTC 泄露      | 网页能否通过 STUN 探测到真实 IP                    |
|   4   | IPv6 泄露        | IPv6 流量是否绕过隧道直接出网                      |
|   5   | 通话功能         | 默认模式（丢弃 STUN）对各类通话的影响              |
|   6   | 通话模式（可选） | 切换到通话模式后，STUN 是否都走代理、不暴露真实 IP |
|   7   | 日常回归         | 分流是否正确，常用应用是否正常                     |

---

## 一、与 Shadowrocket 的关键差异

| 项目                  | Shadowrocket                                  | QuantumultX                                                                                                     |
| --------------------- | --------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| 加密 DNS              | `dns-server` + `fallback-dns-server`          | `[dns]` 的 `doh-server`；**没有加密备用 DNS**                                                                   |
| 明文 `server`         | —                                             | 设置了 `doh-server` 后，全局明文 `server` 与 system 均被忽略；绑定域名的 `server=/域名/IP` 仍走明文，已全部注释 |
| 防止 IP 规则触发解析  | `GEOIP,CN,DIRECT,no-resolve`                  | **不支持 `no-resolve`**，改用 `host-keyword, ., 兜底线路` 在域名阶段接住未收录域名                              |
| 规则优先级            | 从上到下                                      | 「分流匹配优化」开启时先按**类型**分档，同档内才看本地/远程，详见附录                                           |
| 不使用 Fake-IP 的域名 | `always-real-ip`                              | `dns_exclusion_list`（两者内容已同步）                                                                          |
| WebRTC / STUN         | `stun-response-ip` 返回假 IP                  | **没有该功能**，用 `udp_drop_list = 443, STUN, QUIC` 直接丢弃 STUN 包                                           |
| 节点不支持 UDP 时     | `udp-policy-not-supported-behaviour = REJECT` | `fallback_udp_policy = reject`                                                                                  |
| 硬编码 DNS            | `hijack-dns` 劫持                             | 无对应配置，官方未说明。推断发往 `8.8.8.8:53` 等的查询作为纯 IP 流量分流（境外 IP 走兜底线路）                  |
| HTTPDNS 拦截          | 模块，含分流规则                              | 远程重写 `QX-HTTPDNS.Block.conf`，**只有 URL 重写**；其中 `http://` 部分无需 MITM                               |
| 策略组类型            | `select` / `url-test`                         | `static` / `url-latency-benchmark`                                                                              |

> 💡 **WebRTC 测试的预期结果与 Shadowrocket 不同。** QX 直接丢弃 STUN 包，Trickle ICE 里应表现为**没有 srflx 候选**，而不是显示某个假 IP。

---

## 二、已知问题与限制

### 1. 配置依赖「分流匹配优化」保持开启

`host-keyword, ., 兜底线路` 只有在该开关开启时，才会排在所有远程 `host-suffix` 规则之后。**一旦关闭，它会截走全部域名请求，远程规则集全部失效。** 位置：QX 设置 → 其他设置。

### 2. 规则集会把检测站分到直连

QX 版 `China.list` 收录了 `browserleaks.com`、`myip.la`、`whatismyip.com`。已在 `[filter_local]` 加入 8 条 `host-suffix` 规则指向「兜底线路」，与 China.list 同档、本地优先。2026-09-25 第 1 步已确认生效。

### 3. 没有加密备用 DNS

DoH 不可用时（如需网页登录的公共 Wi-Fi、DoH 被屏蔽的网络）直连域名会解析失败，需临时关闭 QX 完成登录。在境外时 doh.pub、AliDNS 延迟较高，直连的国内网站首次打开可能稍慢。

### 4. 默认模式会影响部分通话

`udp_drop_list` 丢弃所有 STUN 包，ICE 连通检测与 UDP TURN 也会被一并丢弃。需要通话时切换到「通话模式」，见第 6 步。

### 5. 未收录的国内小站会走代理

这是 `host-keyword, .` 防 DNS 泄漏的代价。遇到时在 `[filter_local]` 为其单独加一条 `host-suffix, 域名, direct`。

### 6. 分流模式下存在无法根治的关联风险

网站可以在页面中嵌入一个走直连的资源，从服务端把真实 IP 与你的访问关联起来。这是分流代理的固有特性，只有全局代理能规避。

---

## 三、测试前准备

- [ ] 导入最新配置，**长按首页风车 › 更新**，让所有规则资源、重写资源重新下载
- [ ] 在规则资源列表中确认各资源均已加载成功（显示规则条数），**尤其是「数字加密」**
- [ ] 设置 › 其他设置 › **「分流匹配优化」为开启**
- [ ] 首页运行模式为 **规则分流**，而不是全部代理或全部直连
- [ ] **QX设置中「重写」总开关为开启**，且远程重写「HTTPDNS拦截器」为启用状态。总开关关闭时，所有重写（含 HTTPDNS 拦截、去广告）都不生效
- [ ] 选好节点，确认能正常上网；「兜底线路」当前选择的是代理，而不是 direct
- [ ] 记下真实 IP 备用：先关闭 QX，访问 `ip.sb` 记录 IP，再重新开启
- [ ] 进入首页「网络活动」，清空已有记录，方便后续查看

> ⚠️ 后续所有测试都要和这个真实 IP 做对比。检测页会显示真实 IP，截图外发前记得打码。

---

## 四、测试步骤

### 第 1 步 · 规则与重写生效情况

**操作** 　QX 没有「测试规则」功能，改为用 Safari 依次访问下列地址，再到首页「网络活动」查看每条请求下方的 `规则类型, 规则内容, 策略`。

**预期结果**

| 访问                                    | 预期命中规则                    | 预期策略     | 验证目的                                     |
| --------------------------------------- | ------------------------------- | ------------ | -------------------------------------------- |
| `https://www.baidu.com`                 | `HOST-SUFFIX, BAIDU.COM`        | DIRECT       | 远程 China.list 正常                         |
| `https://www.taobao.com`                | `HOST-SUFFIX, TAOBAO.COM`       | DIRECT       | 同上                                         |
| `https://x.com`                         | `HOST-SUFFIX, X.COM`            | Twitter      | 远程分类规则集正常                           |
| `https://telegram.org`                  | `HOST-SUFFIX, TELEGRAM.ORG`     | Telegram     | 同上                                         |
| `https://www.tiktok.com`                | `HOST-SUFFIX, TIKTOK.COM`       | TikTok       | 同上                                         |
| `https://browserleaks.com`              | `HOST-SUFFIX, BROWSERLEAKS.COM` | **兜底线路** | 本地检测站规则压过 China.list                |
| `http://httpbin.org`                    | `HOST-KEYWORD, .`               | 兜底线路     | 未收录域名被兜底规则接住，不做本地解析       |
| `http://neverssl.com`                   | `HOST-KEYWORD, .`               | 兜底线路     | 同上                                         |
| `http://119.29.29.29/d?dn=www.qq.com` ⚠️ | 被重写拒绝                      | reject       | HTTPDNS 拦截重写生效（明文 HTTP，无需 MITM） |

> 💡 `browserleaks.com` 若显示 DIRECT，说明命中的是 China.list 而非本地规则，第 2 步结果将无效。

> ⚠️ 测试 `http://119.29.29.29/d?dn=www.qq.com` 前，须在 QX 设置**开启「重写」总开关**，否则重写不生效，页面会直接返回一串 IP，被误判为拦截失败。

**不符合时**

- **任何域名请求显示 `GEOIP, CN` 或 `IP-CIDR, …`**：域名走到了 IP 类规则，发生了本地解析。检查「分流匹配优化」是否开启、`host-keyword, ., 兜底线路` 是否仍在。
- **所有域名都显示 `HOST-KEYWORD, .`**：「分流匹配优化」被关闭了，远程规则集全部失效。
- **`browserleaks.com` 为 DIRECT**：把 `[filter_local]` 中 8 条检测站规则从 `host-suffix` 改为 `host`（第 ① 档）后重测。
- **HTTPDNS 地址能返回 IP 列表**：先确认设置「重写」总开关已开启，再检查「HTTPDNS拦截器」是否启用、资源是否下载成功。

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

**不符合时** 　① 回到第 1 步，确认 `browserleaks.com` 为兜底线路、「分流匹配优化」已开启；② 在「网络活动」中查找对 `browserleaks` 相关域名的记录，看命中了哪条规则；③ 确认 `[dns]` 中没有被放开的 `server = /域名/IP` 行。

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
```

6. 再打开 https://browserleaks.com/webrtc ，查看「Public IP Address」一栏

**预期结果** 　`udp_drop_list` 含 `STUN`，三个地址均应**没有 srflx 候选**；browserleaks 的 WebRTC 页不应显示任何公网 IP。

| Trickle ICE 结果  | 判定                                      |
| ----------------- | ----------------------------------------- |
| 无 srflx，仅 host | ✅ STUN 被丢弃                             |
| 出现节点 IP       | ⚠️ STUN 未被丢弃，但该路径走了代理，不泄露 |
| 出现真实 IP       | ❌ 泄露                                    |

> 💡 host 候选应为 mDNS 的 `xxxx.local` 形式，不会暴露局域网 IP。

**不符合时** 　确认 `[general]` 中生效的是 `udp_drop_list = 443, STUN, QUIC`，而不是通话模式的 `udp_drop_list = QUIC`。

<br>

### 第 4 步 · IPv6 泄露测试

> ⚠️ 必须在**确实提供 IPv6 的网络**下测试，否则结果无意义。

**操作**

1. **先关闭 QX**，打开 https://test-ipv6.com ，确认能看到 IPv6 地址，以此证明该网络有 IPv6
2. 开启 QX，重新测试

**预期结果**

| 结果                              | 判定              |
| --------------------------------- | ----------------- |
| 无 IPv6 地址，或仅显示节点的 IPv6 | ✅ 通过            |
| 出现不属于节点的 IPv6 地址        | ❌ IPv6 绕过了隧道 |

> 💡 `[dns]` 启用了 `no-ipv6`，QX 不返回 AAAA 记录；「兼容性增强」保持关闭。

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

1. 在 `config.conf` 中切换到通话模式，**两处同时改**：
   - `[general]`：注释 `udp_drop_list = 443, STUN, QUIC`，放开 `# udp_drop_list = QUIC`
   - `[filter_local]`：放开「通话模式配套」下的 9 行 STUN 规则
2. 更新配置，重复第 3 步的 Trickle ICE 测试
3. 在「网络活动」中查看三个 STUN 域名命中的规则
4. 重复第 5 步中失败的通话项
5. **测试结束后改回默认模式**，并重新执行一次第 3 步，确认恢复为无 srflx

**预期结果**

| 检查项                   | 预期                                        |
| ------------------------ | ------------------------------------------- |
| 三个 STUN 地址的 srflx   | **均为节点 IP**                             |
| `stun.chat.bilibili.com` | 命中 `HOST, STUN.CHAT.BILIBILI.COM` → proxy |
| `stun.miwifi.com`        | 命中 `HOST, STUN.MIWIFI.COM` → proxy        |
| 第 5 步中失败的通话      | 恢复正常                                    |

**不符合时**

- **`stun.miwifi.com` 的 srflx 为真实 IP**：检查 `dns_exclusion_list` 中是否又出现了 `*.miwifi.com`。该通配符会让 `stun.miwifi.com` 直接拿到真实 IP，QX 无法按域名分流；现已收窄为 `miwifi.com, www.miwifi.com`。
- **某个 STUN 显示真实 IP**：说明它未被 `host` 规则覆盖，在「网络活动」查看命中了哪条规则。
- **通话仍失败**：检查所选节点是否支持 UDP 转发（`udp-relay=true`）；不支持时受 `fallback_udp_policy = reject` 影响会直接失败。

<br>

### 第 7 步 · 日常功能回归

| 测试项                                | 预期结果                                                              |
| ------------------------------------- | --------------------------------------------------------------------- |
| 淘宝、B站、微信、支付宝               | 正常且快，走直连                                                      |
| Google、YouTube、Twitter              | 正常，走代理                                                          |
| App Store、iCloud                     | 正常，走「苹果服务」（默认 direct）                                   |
| AI 服务（ChatGPT、Claude）            | 走「AI服务」策略组                                                    |
| TikTok、PayPal、加密货币交易所        | 走各自设定的策略组，账户无异常提示                                    |
| 随机打开几个冷门国内网站              | 能打开；若走了兜底线路导致变慢，按需单独加 direct 规则                |
| 公共 Wi-Fi 登录页                     | 打不开时临时关闭 QX 完成登录，这是没有加密备用 DNS 的已知代价         |
| 节点延迟测试                          | 各节点能正常显示延迟（测速地址已改为 `www.gstatic.com/generate_204`） |
| AirPlay 投屏、NAS、打印机等局域网设备 | 正常；私有网段已加入 `excluded_routes`，这类流量不再经过 QX           |
| 小米路由器管理页 `miwifi.com`         | 连接小米路由器 Wi-Fi 时能正常打开                                     |

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

**原理** 　Fake-IP 模式下，App 的 DNS 查询由 QX 直接返回假 IP，不产生上游查询；随后按分流规则决定走向——走代理时把**域名**交给节点解析，走直连时才用 `doh-server`（doh.pub、AliDNS）在本机解析。

**例外** 　以下情况本机仍会使用 `doh-server`：

- 域名请求走到了 IP 类规则（已由 `host-keyword, .` 防住）
- `dns_exclusion_list` 列表内的域名
- 节点域名自身的解析
- 运行模式切到「全部直连」，或「兜底线路」被切到 direct

### 本地规则只在「同一类型」内优先于远程规则

「分流匹配优化」开启时，QX 先按规则类型分档，**同一类型内**才是本地优先于远程。检测站的修正规则写成本地 `host-suffix`，与 China.list 中的 `host-suffix, browserleaks.com` 同档，因此本地生效；若写成 `host-keyword`，则会输给远程的 `host-suffix`。详见 [附录 分流规则匹配与 DNS 解析过程](#附录-分流规则匹配与-dns-解析过程)。

### 解析服务器的 IPv6 地址不是泄露

browserleaks 的 DNS 结果里出现 `2400:cb00::`、`2607:f8b0::`、`2620:171::` 等地址，那是**解析服务器自身的 IPv6**，与你设备的 IPv6 无关。

---

## 六、测试进度与结果

> 测试环境：2026-09-25 ｜ iPhone ｜ 当前网络不提供 IPv6 ｜ 所在地、本地网络、代理节点待补充

| 测试项                   | iPhone |  Mac  | 说明                                                                                                   |
| ------------------------ | :----: | :---: | ------------------------------------------------------------------------------------------------------ |
| 第 1 步 规则与重写       |   ✅    |   ⬜   | 各域名命中预期规则；`browserleaks.com` 命中本地规则 → 兜底线路；未收录域名 → `HOST-KEYWORD, .`；HTTPDNS 请求被重写拒绝 |
| 第 2 步 DNS 泄露         |   ✅    |   ⬜   | 顶部为节点 IP，解析服务器均在节点一侧，无腾讯、阿里，无运营商                                           |
| 第 3 步 WebRTC 泄露      |   ✅    |   ⬜   | 三个 STUN 地址均无 srflx 候选，STUN 已被丢弃                                                             |
| 第 4 步 IPv6 泄露        |   ⏸    |   ⏸   | 当前网络不提供 IPv6，结果无参考价值，需换网络补测                                                       |
| 第 5 步 通话功能         |   ⬜    |   ⬜   | 待后续跟进                                                                                             |
| 第 6 步 通话模式（可选） |   ⬜    |   ⬜   | 待后续跟进                                                                                             |
| 第 7 步 日常回归         |   ⬜    |   ⬜   | 待后续跟进                                                                                             |

<small>✅ 通过 ｜ ⏸ 待补测 ｜ ⬜ 待测</small>

---

## 七、待办清单

- [x] **第 1～3 步（iPhone）** —— 2026-09-25 全部符合预期
- [x] **确认 `browserleaks.com` 命中本地规则** —— 已确认，检测站规则保持 `host-suffix`
- [x] **收窄 `dns_exclusion_list` 中的 `*.miwifi.com`** —— 已改为 `miwifi.com, www.miwifi.com`，四个客户端同步
- [ ] **确认小米路由器管理页仍可访问** —— 连小米路由器的 Wi-Fi 时打开 `miwifi.com`
- [ ] **IPv6 泄露测试** —— 当前网络不提供 IPv6，需换到有 IPv6 的网络补测，如国内宽带或蜂窝数据
- [ ] **第 5～7 步（iPhone）** —— 通话功能、通话模式（可选）、日常回归
- [ ] **Mac 端完整跑一遍** —— 第 1～7 步（如在 Mac 上使用 QX）
- [ ] **根据第 5 步结果决定默认模式** —— 通话需求多时，可评估常开通话模式

---

## 附录 分流规则匹配与 DNS 解析过程

> 本节说明 QX 如何为一个请求选择规则、何时会发起本地 DNS 解析，以及 `[filter_local]` 中 `host-keyword, ., 兜底线路` 为什么能防止 DNS 泄漏。
> 依据：官方 [sample.conf](https://raw.githubusercontent.com/crossutility/Quantumult-X/master/sample.conf)、[QX Wiki Book](https://qx.atlucky.me/shi-yong-fang-fa/pei-zhi-wen-jian-xiang-jie/filter-fen-liu-gui-ze)、[Lucy's Tool](https://wiki.repcz.link/quantumultx/filter/)。

### 1. QX 平时并不做 DNS 解析（Fake-IP）

App 访问 `x.com` 时，先向系统查询它的 IP。QX 不去真实解析，而是直接回给 App 一个假 IP（`198.18.x.x`），并记下「这个假 IP 对应 `x.com`」。App 拿假 IP 发起连接，QX 截住连接，查出域名 `x.com`，再去匹配分流规则。

**到这一步，还没有任何 DNS 查询发出去。**

> `dns_exclusion_list` 中的域名例外：它们不用假 IP，App 查询时 QX 就会用 `doh-server` 在本地真实解析。所以该列表里不应放境外域名。

### 2. 不同类型的规则需要的信息不同

| 规则类型                                                  | 判断时需要什么   | 是否需要 DNS 解析                                     |
| --------------------------------------------------------- | ---------------- | ----------------------------------------------------- |
| `host` / `host-suffix` / `host-wildcard` / `host-keyword` | 只需要域名字符串 | **不需要**                                            |
| `user-agent`                                              | 请求头里的 UA    | 不需要（但只对明文 HTTP 或被 MITM 解密的 HTTPS 可见） |
| `geoip` / `ip-cidr` / `ip6-cidr` / `ip-asn`               | 需要真实 IP      | **需要**：域名请求必须先解析出真实 IP                 |
| `final`                                                   | 不需要任何信息   | 不需要                                                |

一个**域名请求**只要走到 IP 类规则，QX 就会用 `[dns]` 的 `doh-server`（doh.pub 腾讯、alidns 阿里）在本地解析它。国内 DNS 服务商因此看到「这个用户在查某个境外域名」，这就是 **DNS 泄漏**。

QX **不支持** `no-resolve` 参数，无法像 Shadowrocket 那样让 IP 类规则跳过域名请求。

### 3. 规则匹配优先级

#### 「分流匹配优化」开启时（App 默认开启，位置：设置 → 其他设置）

先按类型分档，档位高的先匹配，匹配上就停。**不同档位之间只看类型，与本地/远程、书写先后均无关**：

```
① host
② host-suffix         ← 远程规则集的绝大多数规则在这一档（China.list、Google.list……）
③ host-wildcard
④ host-keyword        ← host-keyword, ., 兜底线路 在这一档（域名类的最后一档）
─────────── 以上只看域名，不用解析；以下需要真实 IP 或 UA ───────────
⑤ user-agent          （资料对 ⑤ ⑥ 先后说法不一，见下方说明）
⑥ IP 类：ip-cidr = ip6-cidr = geoip = ip-asn   ← 域名请求走到这里就要做 DNS 解析
⑦ final               （写在哪里都一样，始终最后）
```

**同一档位内**，按加载顺序匹配：

```
远程且带 inserted-resource=true  >  本地 [filter_local]（从上到下）  >  远程 [filter_remote]（按列表顺序，列表内从上到下）
```

实例：
- 本地 `host-keyword, google, 谷歌服务` 输给远程 Google.list 的 `host-suffix, googleapis.com`（第 ④ 档 < 第 ② 档）。实测 `play.googleapis.com` 命中的是远程规则。
- 本地 `host-suffix, browserleaks.com, 兜底线路` 与 China.list 的同名规则同在第 ② 档，本地先加载，由本地生效。

> 关于 ⑤ ⑥ 的先后：多数资料写 user-agent 在 IP 类之前，个别资料相反，官方未说明。本配置中域名请求在第 ④ 档已全部分流完毕，⑤ ⑥ 只会遇到纯 IP 请求，先后基本无影响。

#### 「分流匹配优化」关闭时

不分档位，按加载顺序逐条匹配：本地（从上到下）> 远程（按列表顺序），但域名类规则始终先于 IP 类。

⚠️ 此时本地的 `host-keyword, ., 兜底线路` 会排在所有远程规则集之前，截走全部域名请求，**远程规则集全部失效**。本配置依赖该开关保持开启。

### 4. 用实际请求走一遍

#### 例 1　`www.baidu.com`：规则集收录的国内域名

```
② host-suffix, baidu.com（China.list）→ 命中 → DIRECT
```

在第 ② 档结束。直连需要真实 IP，QX 用国内 DoH 解析，这是正常的：国内网站本就该用国内 DNS，可以分到就近 CDN。

#### 例 2　`x.com`：规则集收录的境外域名

```
② host-suffix, x.com（Twitter.list）→ 命中 → Twitter 策略组（代理）
```

在第 ② 档结束。走代理时，QX 把域名 `x.com` 原样交给节点，**由节点在海外解析**，本地不做任何查询。

#### 例 3　`o30969.ingest.us.sentry.io`：任何规则集都没收录的境外域名

**不加 `host-keyword, .` 时：**

```
① ~ ⑤ 全部未命中
⑥ geoip, cn → 需要 IP → 用 doh.pub / alidns 解析 sentry.io   ← ⚠️ 泄漏发生在这里
          → 结果是美国 IP，不是 CN → 不命中
⑦ final → 兜底线路（代理）→ 节点在海外再解析一次
```

腾讯或阿里的 DNS 已看到这个境外域名；而最终走的是代理，由节点重新解析，本地这次解析的结果**根本没用上**，白白泄漏了一次。

**加上之后：**

```
① ~ ③ 未命中
④ host-keyword, .  →  域名包含 "." → 命中 → 兜底线路（代理）
```

在第 ④ 档结束，**走不到第 ⑥ 档的 geoip**，本地不做任何查询。实测「网络活动」中显示为 `HOST-KEYWORD, ., 兜底线路`。

#### 例 4　Telegram App 直连 `149.154.167.51`：纯 IP 请求

```
① ~ ④ 域名类：没有域名可比，全部跳过
⑥ ip-cidr（Telegram.list）→ 命中 → Telegram 策略组
```

本身就是 IP，无需解析，不受 `host-keyword, .` 影响，照常按 IP 规则分流。

#### 例 5　`xiaodian-abc.cn`：没被收录的国内小网站（副作用）

```
不加：⑥ geoip 解析出国内 IP → 命中 → DIRECT
加上：④ host-keyword, . → 兜底线路（默认代理）
```

会绕道代理，可能变慢，一般仍能打开。遇到时单独加一条 `host-suffix, xiaodian-abc.cn, direct` 即可。

### 5. 要点总结

| 问题                                  | 结论                                                                                                                                                                              |
| ------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `host-keyword, .` 为什么能防 DNS 泄漏 | 让未收录的域名在**不需要 DNS 的第 ④ 档**就分流完，走不到需要 DNS 的 geoip                                                                                                         |
| 为什么用 `.`                          | 所有域名都含「.」，它能接住任何域名                                                                                                                                               |
| 为什么不会抢走其他规则                | 它在域名类**最低档**，更具体的域名规则（不论本地、远程）都先匹配                                                                                                                  |
| 为什么 `final` 做不到                 | `final` 在第 ⑦ 档，请求到达时已经过第 ⑥ 档，DNS 查询早已发出。`final` 只决定线路，拦不住解析                                                                                      |
| 与 Shadowrocket 的对应                | 效果等同 `GEOIP,CN,DIRECT,no-resolve`：域名请求跳过 geoip，只有纯 IP 请求才按 geoip 分流                                                                                          |
| 前提条件                              | 「分流匹配优化」必须开启                                                                                                                                                          |
| 代价                                  | ① 未收录的国内小站走代理；② 远程规则集里的 USER-AGENT 规则（China.list 31 条、Apple.list 23 条等，排在第 ⑤ 档）对域名请求不再生效。此前这些规则也只对明文 HTTP 有效，实际损失很小 |

### 6. 同一原理的其他应用

- **泄漏检测站规则**必须写成 `host-suffix`（第 ② 档），才能与 China.list 同档、靠本地优先生效。
- **「通话模式配套」的 STUN 规则**必须写成 `host`（第 ① 档）。例如 `stun.chat.bilibili.com` 若只靠 `host-keyword, stun`（第 ④ 档），会先被 BiliBili.list 的 `host-suffix, bilibili.com`（第 ② 档）截走，走直连并暴露真实 IP。`stun.miwifi.com` 同理，会被 China.list 截走。

### 7. 如何自行验证

打开 QX 首页的「网络活动」，每条请求下方显示 `规则类型, 规则内容, 策略`：

| 看到的规则                                       | 含义                                                                                               |
| ------------------------------------------------ | -------------------------------------------------------------------------------------------------- |
| `HOST-SUFFIX, BAIDU.COM, DIRECT`                 | 被规则集收录，正常分流                                                                             |
| `HOST-KEYWORD, ., 兜底线路`                      | 未被收录的域名，由兜底规则接住，未做本地解析                                                       |
| 域名请求显示 `GEOIP, CN, DIRECT` 或 `IP-CIDR, …` | ⚠️ 域名走到了 IP 类规则，发生了本地解析。先检查「分流匹配优化」是否开启、`host-keyword, .` 是否仍在 |
