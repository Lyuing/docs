# VPN 代理转发部署分享文档：3x-ui + Xray + VLESS-REALITY（首尔节点）

> 服务器：腾讯云 首尔 `43.164.130.**` · Ubuntu 26.04 LTS · 2 核 2G · 部署日期：2026-10-16
> 客户端目标：Clash Verge Rev（Windows/macOS/Android 通用 Mihomo 内核）
> ⚠️ 敏感信息（面板密码、UUID、私钥）不在本文档明文扩散，统一存于服务器 `~/vpn-3xui/CREDENTIALS.txt` / `SUBSCRIPTION.txt`（权限 600），分享时请脱敏。

---

## 一、前期方案对比分析

### 1.1 需求拆解

| 需求 | 含义 |
|---|---|
| 稳定 | 成熟内核、社区活跃、被封概率低、长期维护 |
| 延迟低 | 协议握手开销小、无多余 TLS 证书交互、首尔到国内物理延迟本身 ~30-60ms |
| 支持 Clash Verge | 节点/订阅能被 Mihomo 内核解析（VLESS/Trojan/Hysteria2 等） |
| UI 界面操作 | 有 Web 面板增删用户、看流量、生成二维码/订阅，而不是纯改配置文件 |
| 远程订阅 | 面板自带订阅服务，一条 URL 在 Clash Verge 里自动更新 |

### 1.2 主流方案对比

| 方案 | 内核/协议 | 延迟 | 抗封锁 | Clash Verge | UI 面板 | 订阅 | 运维成本 | 结论 |
|---|---|---|---|---|---|---|---|---|
| **WireGuard** | 内核 UDP VPN | ★★★★★ 最低 | ★★ UDP 特征明显、易被 QoS/封 UDP | △ 需手动转配置、无原生订阅 | 无官方（需 wg-easy） | △ | 中 | 适合组网，不适合作为翻墙主力 |
| **OpenVPN** | TCP/UDP TLS VPN | ★★★ 握手重、单线程 | ★★ 特征成熟易识别 | ✗ 基本不支持 | 有但笨重 | ✗ | 高 | 老牌企业 VPN，已过时 |
| **Shadowsocks (ss-rust)** | SOCKS 加密代理 | ★★★★ | ★★★ 无 TLS 伪装易被主动探测 | ✓ 原生支持 | 无官方（需另搭） | △ 需订阅转换 | 低 | 轻量，但裸奔风险高 |
| **Trojan** | TLS 443 伪装 | ★★★★ | ★★★★ 需真域名+证书 | ✓ | 无官方 | △ | 中（证书续期） | 可用，但证书维护麻烦 |
| **V2Ray / VMess** | VMess+WS+TLS | ★★★ 协议头开销大 | ★★★ 已被大量识别 | ✓ | 有（v2-ui 已停更） | ✓ | 中 | 已被 VLESS 取代 |
| **Xray / VLESS+REALITY+Vision** ✅ | TCP 443，借真站 TLS 指纹 | ★★★★★ Vision 减少解密开销、无 TLS 握手自建 | ★★★★★ REALITY 借 `www.samsung.com` 等真站证书，无需自有域名/证书、最难识别 | ✓ Mihomo 原生支持 | **3x-ui** 成熟面板 | ✓ 原生通用/Clash/JSON 三种订阅 | 低 | **本次选型** |
| **Hysteria2** | QUIC/UDP + Brutal CC | ★★★★★ 弱网/丢包时最低 | ★★★ UDP 大流量易被 QoS | ✓ | 可集成进 3x-ui | ✓ | 低 | 优秀的**备用**协议（本次预留 844*/UDP，后续可在面板一键加） |
| **TUIC v5** | QUIC | ★★★★ | ★★★ 同 UDP 问题 | ✓ | 面板支持有限 | △ | 中 | 小众，生态不如 Hysteria2 |
| **sing-box 一站式** | 多协议内核 | ★★★★ | ★★★★ | ✓（自身即客户端） | 无成熟官方 UI（需 yacd 等） | △ | 高（JSON 手写） | 内核强，但 UI/订阅不如 3x-ui 开箱即用 |

### 1.3 最终选型

```
3x-ui (面板/UI/订阅)  +  Xray-core (内核)  +  VLESS + REALITY + xtls-rprx-vision (主协议)
伪装目标 dest: www.samsung.com:443（首尔 IP 访问三星最自然、TLS 1.3、Xray 官方候选列表内）
预留: Hysteria2 @ 844*/UDP 作为弱网备用（面板里 5 分钟可加，本次未启用避免 UDP 被 QoS 干扰判断）
```

理由：① 不需要买域名/申请证书，部署 10 分钟完成；② REALITY 是目前抗主动探测最强的方案之一；③ 3x-ui 同时输出通用订阅 + Clash 订阅 + JSON 订阅，Clash Verge 粘贴一条 URL 即可；④ Docker 部署，升级 `docker compose pull && up -d` 一条命令。

---

## 二、中间每个指令（完整复现）

### 2.0 环境勘察

```bash
cat /etc/os-release; uname -a; whoami
nproc; free -h; df -h
curl -s ipinfo.io/43.164.130.**      # 确认机房：Seoul / Tencent AS132203
ss -tlnp                             # 确认仅 22 端口在监听
sudo -n true && echo "sudo OK"       # 确认免密 sudo
```

### 2.1 安装 Docker（Ubuntu 26.04 官方源自带 docker.io，无需外加 Docker 源）

```bash
sudo DEBIAN_FRONTEND=noninteractive apt-get update -qq
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y -qq docker.io docker-compose-v2 curl openssl sqlite3 jq
sudo systemctl enable --now docker
sudo usermod -aG docker ubuntu
docker --version                     # Docker 29.1.3
sudo docker compose version          # Compose 2.40.3
sudo docker run --rm hello-world     # 验证拉镜像/运行正常
```

### 2.2 部署 3x-ui（Docker Compose，host 网络）

```bash
mkdir -p ~/vpn-3xui/db ~/vpn-3xui/cert
cat > ~/vpn-3xui/docker-compose.yml <<'YAML'
services:
  3x-ui:
    image: ghcr.io/mhsanaei/3x-ui:latest
    container_name: 3x-ui
    hostname: vpn-seoul
    volumes:
      - ./db:/etc/x-ui
      - ./cert:/root/cert
    environment:
      XRAY_VMESS_AEAD_FORCED: "false"
      XUI_ENABLE_FAIL2BAN: "true"
      TZ: Asia/Shanghai
    tty: true
    network_mode: host          # host 模式省去每个入站端口逐个映射，后续加协议不用改 compose
    restart: unless-stopped
YAML
cd ~/vpn-3xui && sudo docker compose pull
sudo docker compose up -d
sudo docker logs 3x-ui          # 应看到 Web server :205* / Sub :209* / Xray 26.10.10 started
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:205*/   # 200
```

### 2.3 初始化面板：改默认账号 / 端口 / 随机路径

```bash
PANEL_USER="vpnadmin"
PANEL_PASS=$(openssl rand -base64 18 | tr -d '/+=' | head -c 20)
PANEL_PATH=$(openssl rand -hex 8)
PANEL_PORT=543**
# 写入 ~/vpn-3xui/CREDENTIALS.txt 并 chmod 600（见服务器文件，本文不贴明文密码）

sudo docker exec 3x-ui /app/x-ui setting -username "$PANEL_USER" -password "$PANEL_PASS" \
  -port $PANEL_PORT -webBasePath "/$PANEL_PATH/"
sudo docker exec 3x-ui /app/x-ui setting -show
cd ~/vpn-3xui && sudo docker compose restart
curl -s -o /dev/null -w "%{http_code}\n" "http://127.0.0.1:543**/$PANEL_PATH/"   # 200
```

### 2.4 防火墙 UFW（云安全组见 3.3，两层都要开）

```bash
sudo ufw --force reset
sudo ufw default deny incoming; sudo ufw default allow outgoing
sudo ufw allow 22/tcp comment 'SSH'
sudo ufw allow 543**/tcp comment '3x-ui Panel'
sudo ufw allow 209*/tcp comment '3x-ui Sub'
sudo ufw allow 443/tcp comment 'VLESS Reality'
sudo ufw allow 844*/udp comment 'Hysteria2 backup'
sudo ufw --force enable; sudo ufw status numbered
```

### 2.5 通过 API 创建 VLESS+REALITY 入站（等价于 UI 点选，见第三节截图式步骤）

```bash
# 1) 生成 API Token（注意输出第二行才是 token，见 FAQ-6）
sudo docker exec 3x-ui /app/x-ui setting -getApiToken -tokenName install -tokenScope admin
API_TOKEN=xxxxxxxx   # 保存到 ~/vpn-3xui/.api_token (600)

# 2) 生成 Reality 密钥对 / UUID / ShortId
sudo docker exec 3x-ui /app/bin/xray-linux-amd64 x25519   # PrivateKey / Password(PublicKey)
cat /proc/sys/kernel/random/uuid
openssl rand -hex 8                                       # shortId

# 3) POST 创建入站（关键字段）
curl -X POST "http://127.0.0.1:543**/<PANEL_PATH>panel/api/inbounds/add" \
  -H "Authorization: Bearer $API_TOKEN" -H "Content-Type: application/json" -d '{
  "remark":"Seoul-VLESS-Reality","enable":true,"port":443,"protocol":"vless",
  "settings":"{\"clients\":[{\"id\":\"<UUID>\",\"flow\":\"xtls-rprx-vision\",\"email\":\"seoul-user1\",\"subId\":\"<SUB_ID>\"}],\"decryption\":\"none\"}",
  "streamSettings":"{\"network\":\"tcp\",\"security\":\"reality\",\"realitySettings\":{\"dest\":\"www.samsung.com:443\",\"serverNames\":[\"www.samsung.com\"],\"privateKey\":\"<PRIV>\",\"shortIds\":[\"<SHORT_ID>\"],\"settings\":{\"publicKey\":\"<PUB>\",\"fingerprint\":\"chrome\"}},\"tcpSettings\":{\"header\":{\"type\":\"none\"}}}",
  "sniffing":"{\"enabled\":true,\"destOverride\":[\"http\",\"tls\",\"quic\",\"fakedns\"]}"
}'
# 返回 {"success":true,"msg":"Inbound has been successfully created."}

# 4) 启用 Clash / JSON 订阅（默认关！见 FAQ-1）
#    POST /panel/api/setting/all 取全量 -> 改 subClashEnable=true, subJsonEnable=true,
#    subClashAutoDetect=true, subDomain=43.164.130.** -> POST /panel/api/setting/update
cd ~/vpn-3xui && sudo docker compose restart   # 更新 Reality dest 后必须重启才重生成 Xray config
```

### 2.6 验证（两段都实测通过）

```bash
# A) Xray 原生客户端：SOCKS 10808 -> VLESS -> 服务器 127.0.0.1:443
sudo docker cp 3x-ui:/app/bin/xray-linux-amd64 /tmp/xray-test/xray
/tmp/xray-test/xray run -c /tmp/xray-test/client.json &
curl -x socks5h://127.0.0.1:10808 https://ipinfo.io/ip        # 43.164.130.**
curl -o /dev/null -w "%{http_code} %{time_total}s\n" -x socks5h://127.0.0.1:10808 https://www.google.com/generate_204
# 结果: 204, 0.124s（直连 0.100s，协议开销 ~24ms，本地回环测试）

# B) Mihomo（Clash Verge 内核）加载 Clash 订阅 YAML
#    注：服务器本机用公网 IP 会 hairpin 超时（FAQ-5），本地把 server 改 127.0.0.1 验证逻辑
/tmp/xray-test/mihomo -f clash-local.yaml &
curl -x http://127.0.0.1:7890 https://ipinfo.io/ip             # 43.164.130.**
curl -o /dev/null -w "%{http_code} %{time_total}s\n" -x http://127.0.0.1:7890 https://www.google.com/generate_204
# 结果: 204, 0.141s ✅ Clash 配置可用
```

---

## 三、UI 界面的搭建与操作

### 3.1 登录

```
地址: http://43.164.130.**:543**/<随机路径>/     # 完整地址见服务器 SUBSCRIPTION.txt
账号: vpnadmin / 见 CREDENTIALS.txt
```
> 登录后建议：设置 → 安全 → 开启两步验证（2FA）；面板仅 HTTP，后续有域名后应在设置 → 证书里配 TLS，或把 543** 在云安全组里限制为仅自己 IP。

### 3.2 UI 里对应的入站（与 2.5 API 等价，图形化复核）

1. 左侧 **入站列表 → 添加入站**：备注 `Seoul-VLESS-Reality`、协议 `vless`、端口 `443`。
2. 客户端：邮箱 `seoul-user1`、流控 `xtls-rprx-vision`、订阅 ID 自动生成（即 `subId`）。
3. 传输：`tcp`；安全：`Reality`；目标 `www.samsung.com:443`；SNI `www.samsung.com`；
   点 **Get New Cert / 生成密钥对** 自动生成 X25519（⚠️ 确认公钥已填入客户端设置，见 FAQ-2）；ShortID 随机生成。
4. Sniffing 开启。保存 → 重启 Xray（右上角）。

### 3.3 云厂商安全组（UI 之外最容易漏的一步！）

腾讯云控制台 → 云服务器 → 安全组 → 入站规则，逐条放行：

| 端口 | 协议 | 来源 | 用途 |
|---|---|---|---|
| 443 | TCP | 0.0.0.0/0 | VLESS 节点（核心） |
| 209* | TCP | 0.0.0.0/0 | 远程订阅拉取 |
| 543** | TCP | **仅自己的公网 IP/32** | 面板（勿对全网开放） |
| 844* | UDP | 0.0.0.0/0 | Hysteria2 备用（启用时才需要） |
| 22 | TCP | 仅自己 IP/32 为佳 | SSH |

### 3.4 Clash Verge Rev 导入订阅（远程订阅）

1. 打开 Clash Verge Rev → **订阅** → 新建 → 粘贴 Clash 订阅 URL（见 `SUBSCRIPTION.txt` 的「Clash订阅」一行，形如 `http://43.164.130.**:209*/<随机路径>/<subId>`）。
2. 保存后点 **更新**，选中该配置 → 代理页选择 `Seoul-VLESS-Reality-seoul-user1`（或 PROXY 组）。
3. 设置 → 开启 **系统代理**（或 TUN 模式）；延迟测试应显示首尔节点 ~40-80ms（视本地运营商）。
4. 后续在面板增删用户/改节点，Clash Verge 里点一次订阅更新即同步，无需重新分发文件。

### 3.5 面板常用运维（UI）

- **入站列表**：看每用户上下行流量、到期时间、在线 IP；可禁用/重置流量。
- **添加用户**：同一入站里加客户端即可（多设备共用一条订阅 URL，靠 subId 聚合）。
- **设置 → 订阅设置**：通用/Clash/JSON 三个随机路径都在这里，可一键重置路径防泄露。
- **系统状态**：CPU/内存/Xray 版本，一键重启 Xray / 面板。

---

## 四、常见问题模块（本次真实遇到的错误记录）

### FAQ-1：订阅 `/sub/<id>` 能打开，但 Clash 路径 404
- **现象**：`GET /dfs1.../<subId>` 返回 `404 page not found`。
- **原因**：3x-ui v3 默认 `subClashEnable=false`、`subJsonEnable=false`（且订阅路径是每台机器随机生成的 `/sdlbj.../` 这类路径，不是固定 `/sub/`、`/clash/`）。
- **解决**：面板 设置→订阅 打开 Clash/JSON 开关；或 API `POST /panel/api/setting/update` 改上述字段后重启。订阅真实路径可查 DB：`sqlite3 db/x-ui.db "SELECT key,value FROM settings WHERE key LIKE 'sub%Path';"`

### FAQ-2：分享链接 / Clash YAML 缺 `pbk`（public-key），客户端连不上 ★最坑
- **现象**：生成的 VLESS 链接只有 `sid/sni/spx`，没有 `pbk=`；Clash 的 `reality-opts` 里也只有 `short-id`。
- **原因**：新版 3x-ui 的分享生成器只从 `realitySettings.settings.publicKey` 读公钥（源码 `internal/sub/service.go: searchKey(realitySettings,"publicKey")`），而服务端 Xray 只需要 `privateKey`——只填私钥时服务器能跑，但下发的客户端配置残缺。
- **解决**：创建入站时在 `realitySettings` 里同时加 `"settings":{"publicKey":"<公钥>","fingerprint":"chrome"}`，然后重启面板重生成配置。

### FAQ-3：本地测订阅时节点地址是 `localhost`
- **现象**：`curl 127.0.0.1:209*/...` 解码后是 `vless://...@localhost:443`。
- **原因**：入站 `shareAddrStrategy=node` 时按请求 Host 头生成地址。
- **解决**：正常现象。用公网域名/IP 访问即为公网地址；本地调试加 `-H "Host: 43.164.130.**:209*"`；也可在面板把分享地址策略改自定义并填服务器 IP。

### FAQ-4：Xray 警告 `Choosing "www.microsoft.com" as the target will increase the likelihood of your server's IP being blocked by the GFW`
- **原因**：微软站是被用滥的 Reality 目标，Xray 26 直接发警告。
- **解决**：换成 `www.samsung.com:443`（已换；满足 TLS1.3、且与首尔机房地理一致），改后必须重启面板，确认 `docker exec 3x-ui grep samsung /app/bin/config.json` 生效。

### FAQ-5：服务器本机用公网 IP 测 Mihomo 超时 `dial tcp 43.164.130.**:443: i/o timeout`
- **原因**：腾讯云不支持 NAT 回环（hairpin），本机绕公网 IP 回自己不通；不代表外部客户端不通。
- **解决**：本机验证改连 `127.0.0.1:443`（已实测 204/0.14s）；外部可用性以真机 Clash Verge 拉订阅测延迟为准。同时确认云安全组 443 已放行（FAQ-7）。

### FAQ-6：API Token / 设置接口的两个小坑
- `x-ui setting -getApiToken` 输出是多行文本，第二行 `apiToken: xxx` 才是 token，直接 `tail -1` 会拼错前缀导致 401/空响应。
- `GET /panel/api/setting/all` 返回空，必须用 `POST`。鉴权头：`Authorization: Bearer <token>` 且要带面板随机 `webBasePath` 前缀。

### FAQ-7：外网打不开面板/订阅，但服务器本地 curl 200
- **排查顺序**：① `ss -tlnp` 看 443/209*/543** 是否监听 → ② `sudo ufw status` 本机防火墙 → ③ **腾讯云安全组**（最常见，系统内查不到）→ ④ 云厂商是否要求备案/封了 443（换 844* 等高位端口再试）。

### FAQ-8：Clash Verge 导入订阅后延迟 -1 / 超时
- 依次查：订阅选的是 **Clash 专用路径**而非通用路径；节点 `servername=SNI` 与 dest 一致；系统时间误差 < 90 秒（REALITY 对时间敏感）；本地运营商是否屏蔽 443 出站（换 Hysteria2 443/UDP 或加备用端口）。

### FAQ-9：改了入站 dest/密钥后客户端还连旧配置
- 面板更新入站后 Xray 的 `bin/config.json` 不一定立即重写，执行 `docker compose restart` 或面板右上角重启 Xray，再让 Clash Verge 更新订阅。

### FAQ-10：原 IP `43.164.130.**` 被封，换到 `43.164.140.**` 后改端口（2026-10-27）
- **现象**：`43.164.130.**` 所在网段被封，节点不可用。同机房新地址 `43.164.140.**`（仍在腾讯云首尔 `43.164.**.**/16`）当时还能用。随后把面板、订阅换到新端口，但腾讯云实例属性里的防火墙仍显示最初那套规则。
- **原因**：
  1. 封的是网段，不是某一个端口配错。同网段换 IP 只能临时恢复，挡不住下一次按前缀封禁。
  2. `www.samsung.com` 在 Xray 默认 Reality 候选列表里，和大量节点重复，容易被一起记账（FAQ-4 里换成三星只是避开微软，并不是终点）。
  3. 机器里的 ufw 和腾讯云控制台防火墙是两层。ufw 只过滤已经进到这台虚拟机的包；控制台规则在腾讯云网络上，SSH 进系统改不到。所以控制台不会跟着 ufw 变化。外网要通，两边都得放行。
- **已做**（面板和订阅仍对公网监听，只换端口，没有改成 `127.0.0.1`）：
  1. Reality 目标由 `www.samsung.com:443` 改为 `www.hanwha.com:443`（首尔访问约 14ms，TLS 1.3 + H2，证书是该站自己的）。密钥、443、`xtls-rprx-vision` 未动。本机经新目标访问 Google `generate_204` 返回 204。
  2. 面板 `543**` → `241**`（`0.0.0.0`）。订阅 `209*` → `583**`（继续监听所有网卡）。
  3. 本机 ufw 现放行：`22/tcp`、`443/tcp`、`241**/tcp`、`583**/tcp`。`543**` 与未使用的 `844*/udp` 已从 ufw 去掉，机器上也不再监听。
  4. `CREDENTIALS.txt`、`SUBSCRIPTION.txt` 中的旧 IP 已改为 `43.164.140.**`，端口改为上面的新端口。三个订阅地址本机请求返回 200。
- **还要在腾讯云控制台做**（不改的话外网仍进不了新端口）：入站添加 `241**/tcp`、`583**/tcp`；`543**/tcp`、`209*/tcp`、`844*/udp` 可删。控制台规则要在网页里改，改完才会显示新端口。
- **客户端**：旧订阅地址还是 `:209*`，SNI 还是 `www.samsung.com`，对旧地址点「更新」无效。用 `SUBSCRIPTION.txt` 里的新地址重新导入。改 dest 或端口后若仍连不上，按 FAQ-9 重启 Xray 再拉一次订阅。

---

## 五、运维速查

```bash
cd ~/vpn-3xui
sudo docker compose ps                  # 状态
sudo docker logs -f 3x-ui               # 日志
sudo docker compose pull && sudo docker compose up -d   # 升级
sudo docker exec 3x-ui /app/x-ui setting -show           # 面板设置
cat ~/vpn-3xui/SUBSCRIPTION.txt         # 订阅地址（勿外传）
# 备份：打包 db 目录即可（x-ui.db 是全部配置）
tar czf ~/3xui-backup-$(date +%F).tgz -C ~/vpn-3xui db docker-compose.yml CREDENTIALS.txt
```

## 六、后续可扩展（未做，面板内几分钟可加）

1. **Hysteria2 备用节点**（844*/UDP）：弱网/晚高峰丢包时延迟显著优于 TCP，需自签或真证书。
2. **域名 + HTTPS 面板**：解析域名到本机后在面板申请证书，避免面板 HTTP 明文与订阅被劫持。
3. **多用户与限额**：入站内继续加客户端，设流量上限/到期，适合分享给朋友（各自 subId 独立订阅）。
4. **WARP 出站分流**：面板出站里加 Cloudflare WARP，解锁部分流媒体/ChatGPT 的机房 IP 限制。
