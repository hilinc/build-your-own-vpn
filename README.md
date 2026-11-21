# Build Your Own VPN：高性价比双协议自建代理节点完全教程

本教程整理了一套经过晚高峰实战验证的自建代理节点方案，适合希望以较低成本（约 \$5–\$7/月）获得**独享 IP、高稳定性、晚高峰流畅 4K** 体验的个人用户。

---

## 架构概述

单台入门级 Linux VPS（1 vCPU / 1GB RAM）同时部署两套互补协议，兼顾晚高峰高速与弱 UDP 环境下的高兼容性：

| 角色 | 协议 | 监听端口（示例，可自定义） | 适用场景 |
| :--- | :--- | :--- | :--- |
| **主用协议** | **Hysteria2**（基于 QUIC/UDP + 端口跳跃） | UDP `8443`（+ `30000-45000` 端口跳跃） | 晚高峰高丢包/高延迟线路下抗拥塞、跑满带宽 |
| **备用协议** | **Xray VLESS + Reality**（`xtls-rprx-vision`） | TCP `443` | 公司/校园网等限制 UDP 或 QoS 严厉的网络环境 |

> **为什么需要双协议？**  
> 入门级海外 VPS 在晚高峰容易出现回程拥塞与高丢包，纯 TCP 协议（即便开启 BBR）下载速度也可能骤降至几 Mbps；而 Hysteria2 基于 UDP 拥塞控制，在同样丢包链路下仍可保持高吞吐。同时保留 VLESS + Reality 作为 TCP 备选，可在 UDP 受限环境下无缝切换。

---

## 第一步：VPS 选购与线路验收

### 1. 选购核心要点

- **计费方式**：优先选择**按小时计费**的海外云厂商，开到绕路严重或高丢包 IP 时可直接销毁重建，试错成本极低。
- **流量计费口径**：注意区分**单向计费**（只算入/出最大值，如 2TB 实际可用接近 2TB）与**双向合并计费**（入+出合并计算，代理流量一进一出，实际可用减半）。
- **地域与系统**：优先选择离本地物理距离近的机房（如亚太或美西）；操作系统推荐 **Ubuntu 24.04 LTS x64**。
- **关闭非必要附加项**：创建实例时取消勾选自动备份（Automatic Backups）等额外付费项。

### 2. 入门方案参考（以亚太地区为例）

| 方案类型 | 常见月价 | 流量口径 | 特点 |
| :--- | :--- | :--- | :--- |
| 按小时计费型海外 VPS | \$5–\$6/月 | 1TB–2TB（单向） | 换 IP 成本极低，稳定性好，适合入门首选 |
| 大厂轻量应用服务器 | \$3.5–\$7/月 | 双向合并或峰值限速 | 需注意双向流量折半或持续满速动态限速策略 |
| 优化线路年付型 VPS | \$25–\$80/年 | 直连优化线路 | 晚高峰直连延迟低，但常缺货且年付难退款 |

### 3. 开机线路初检

拿到服务器 IP 后，**先测连通性与丢包，再部署服务**：

```bash
# 本地终端测试 30 次 ping，观察丢包率与抖动
ping -c 30 <YOUR_SERVER_IP>
```

- **验收标准**：丢包率应接近 **0%**。若刚开机就出现明显丢包（如 >5%），建议直接销毁实例并更换机房或重新创建，直到拿到丢包为 0% 的 IP。

---

## 第二步：系统初始化与内核网络调优

使用 `root` SSH 登录服务器后，同步系统时间并开启 BBR、调大内核 UDP/TCP 缓冲区：

```bash
# 1. 确保系统时间准确（VLESS Reality 对时间偏差敏感）
timedatectl set-ntp true

# 2. 开启 BBR 并调大网络缓冲区（为 Hysteria2 UDP 大吞吐做准备）
cat << 'EOF' > /etc/sysctl.d/99-bbr.conf
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
EOF

sysctl --system

# 3. 验证 BBR 是否生效（应输出 bbr）
sysctl net.ipv4.tcp_congestion_control
```

---

## 第三步：部署主用协议 —— Hysteria2（UDP 端口跳跃）

### 1. 安装并生成配置

以下脚本会自动安装 Hysteria2、签发自签名证书、生成随机强密码，并启用 HTTP/3 反向代理伪装（监听端口与伪装域名均为示例，可按需替换）：

```bash
apt update && apt install -y curl openssl
curl -fsSL https://get.hy2.sh/ | bash

mkdir -p /etc/hysteria
openssl ecparam -genkey -name prime256v1 -out /etc/hysteria/server.key
openssl req -new -x509 -days 36500 -key /etc/hysteria/server.key -out /etc/hysteria/server.crt -subj "/CN=www.microsoft.com"

HY2_PASS=$(openssl rand -base64 16)

cat << EOF > /etc/hysteria/config.yaml
listen: :8443

tls:
  cert: /etc/hysteria/server.crt
  key: /etc/hysteria/server.key

auth:
  type: password
  password: $HY2_PASS

masquerade:
  type: proxy
  proxy:
    url: https://www.microsoft.com/
    rewriteHost: true
EOF

chown -R hysteria:hysteria /etc/hysteria
chmod 600 /etc/hysteria/server.key
systemctl enable --now hysteria-server.service

echo "=== 请保存 Hysteria2 连接密码 ==="
echo "HY2_PASSWORD: $HY2_PASS"
```

### 2. 配置防火墙与 UDP 端口跳跃（示例：`30000-45000 -> 8443`）

端口跳跃可让客户端定期切换目标 UDP 端口，服务端通过 `iptables` NAT 规则统一转发至监听端口，有效缓解运营商对单一 UDP 端口的限速（端口范围可自由调整）：

```bash
# 自动获取默认公网网卡名
IFACE=$(ip route show default | awk '/default/ {print $5; exit}')

# 放行 TCP 443（供备用协议）与 UDP 8443（供 Hysteria2）
iptables -I INPUT -p tcp --dport 443 -j ACCEPT
iptables -I INPUT -p udp --dport 8443 -j ACCEPT

# 将 UDP 30000-45000 流量转发至 8443
iptables -t nat -A PREROUTING -i "$IFACE" -p udp --dport 30000:45000 -j DNAT --to-destination :8443

# 持久化保存防火墙规则（重启不丢失）
DEBIAN_FRONTEND=noninteractive apt install -y iptables-persistent
netfilter-persistent save
```

> **提示**：如果你使用的云厂商自带网页端安全组（Security Group / Firewall），请同时在云控制台放行 `TCP 443`、`UDP 8443` 与 `UDP 30000-45000`。

### 3. 关于 Salamander 混淆（可选，默认不建议开启）

- 仅当本地运营商对标准 QUIC 流量做针对性限速时，再考虑开启 Salamander 混淆。
- 开启后服务端会失去 HTTP/3 伪装能力，且部分运营商会直接限速无法识别的未知 UDP 流量。
- 如需开启：在 `/etc/hysteria/config.yaml` 末尾追加 `obfs` 段落并重启服务；如需回滚关闭（**前提：`obfs:` 位于文件最末尾**），可执行：
  ```bash
  sed -i '/^obfs:/,$d' /etc/hysteria/config.yaml
  systemctl restart hysteria-server.service
  ```
  > **注意**：关闭服务端混淆后，务必同步删除客户端配置中的 `obfs` 与 `obfs-password` 字段，否则两端协议不匹配会导致一直 `timeout`。

---

## 第四步：部署备用协议 —— Xray VLESS + Reality（TCP 443）

VLESS + Reality 监听 **TCP 443**，与 Hysteria2 互不冲突。

### 1. 安装并生成配置

```bash
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install

UUID=$(xray uuid)
KEYS=$(xray x25519)
PRIV_KEY=$(echo "$KEYS" | awk '/Private/ {print $NF}')
PUB_KEY=$(echo "$KEYS" | awk '/Public|Password/ {print $NF}')
SHORT_ID=$(openssl rand -hex 8)

cat << EOF > /usr/local/etc/xray/config.json
{
  "log": { "loglevel": "warning" },
  "inbounds": [
    {
      "listen": "0.0.0.0",
      "port": 443,
      "protocol": "vless",
      "settings": {
        "clients": [ { "id": "$UUID", "flow": "xtls-rprx-vision" } ],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "show": false,
          "dest": "www.samsung.com:443",
          "xver": 0,
          "serverNames": [ "www.samsung.com" ],
          "privateKey": "$PRIV_KEY",
          "shortIds": [ "$SHORT_ID" ]
        }
      },
      "sniffing": {
        "enabled": true,
        "destOverride": [ "http", "tls", "quic" ]
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "settings": { "domainStrategy": "UseIPv4" },
      "tag": "direct"
    }
  ]
}
EOF

systemctl restart xray && systemctl enable xray

echo "=== 请保存以下客户端连接参数（切勿泄露） ==="
echo "UUID:       $UUID"
echo "PUBLIC_KEY: $PUB_KEY"
echo "SHORT_ID:   $SHORT_ID"
echo "SNI:        www.samsung.com"
```

### 2. 配置细节说明

- **伪装目标域名（`dest` / `serverNames`）**：选择与 VPS 同地区或网络延迟低、支持 TLS 1.3 + H2 且不被封锁的常见大站域名（上方 `www.samsung.com` 仅为示例，请自行替换为合适的目标站点，并避免使用会被 Xray 警告的常见域名）。
- **强制 IPv4 出站（`domainStrategy: UseIPv4`）**：避免部分 VPS 未配置好 IPv6 导致连接卡顿或超时。
- **密钥区分**：服务端配置填 `PrivateKey`，客户端配置只填 `PublicKey`（新版 `xray x25519` 输出中可能显示为 `Password`，上述脚本已兼容提取）。

---

## 第五步：客户端配置与多端同步

### 1. 桌面端（Clash Verge / Mihomo）配置模板

新建一个 `.yaml` 文件，将尖括号中的占位符替换为你的实际参数后导入客户端：

```yaml
proxies:
  - name: "MyVPS-Hy2"
    type: hysteria2
    server: <YOUR_SERVER_IP>
    port: 8443
    ports: "30000-45000"
    hop-interval: 45
    password: "<HY2_PASSWORD>"
    up: "30 Mbps"
    down: "100 Mbps"   # 建议设置为略低于你本地宽带的实际下行带宽
    sni: www.microsoft.com
    skip-cert-verify: true

  - name: "MyVPS-Reality"
    type: vless
    server: <YOUR_SERVER_IP>
    port: 443
    uuid: <UUID>
    network: tcp
    tls: true
    udp: true
    flow: xtls-rprx-vision
    servername: www.samsung.com
    reality-opts:
      public-key: <PUBLIC_KEY>
      short-id: "<SHORT_ID>"
      support-x25519mlkem768: true
    client-fingerprint: chrome

proxy-groups:
  - name: "🚀 节点选择"
    type: select
    proxies:
      - "MyVPS-Hy2"
      - "MyVPS-Reality"
      - DIRECT

  - name: "🐟 漏网之鱼"
    type: select
    proxies:
      - "🚀 节点选择"
      - DIRECT

rules:
  - IP-CIDR,<YOUR_SERVER_IP>/32,DIRECT   # 关键：防止 TUN 模式下 SSH 形成路由环路
  - GEOSITE,private,DIRECT
  - GEOIP,private,DIRECT,no-resolve
  - GEOSITE,category-ads-all,REJECT      # 可选：去广告，如遇国内 App 误杀可注释
  - DOMAIN-SUFFIX,cloudflare.com,🚀 节点选择
  - GEOSITE,openai,🚀 节点选择
  - GEOSITE,anthropic,🚀 节点选择
  - GEOSITE,google,🚀 节点选择
  - GEOSITE,github,🚀 节点选择
  - GEOSITE,telegram,🚀 节点选择
  - GEOSITE,youtube,🚀 节点选择
  - GEOSITE,cn,DIRECT
  - GEOIP,cn,DIRECT,no-resolve
  - MATCH,🐟 漏网之鱼
```

> **安全提醒**：包含真实 IP、密码、UUID 的配置文件属于敏感凭证，**切勿提交到任何公开 Git 仓库**。

### 2. 多台电脑同步方案

- **自动更新订阅（推荐）**：在 GitHub 创建一个 **Secret Gist**（务必选择 Secret，不要选 Public），粘贴你的 YAML 配置后点击 `Raw`，将 URL 中 `/raw/` 与文件名之间的 commit hash 删去（形如 `https://gist.githubusercontent.com/<user>/<gist_id>/raw/<filename>.yaml`），在各台设备的 Clash Verge 中作为远程订阅导入。后续只需修改 Gist，各设备点刷新即可同步。
- **离线传输**：直接通过 AirDrop / 局域网传输 `.yaml` 文件，以本地配置（Local）方式导入。

### 3. 移动端（iOS Shadowrocket）导入要点

- **方式 A（订阅导入）**：直接添加上述 Secret Gist 的 Raw URL（类型为 Subscribe）。
- **方式 B（链接/二维码导入）**：按以下格式拼接分享链接（密码中如有特殊字符需 URL 编码），用 `qrencode` 生成二维码扫码导入：
  - **Hysteria2**：
    ```text
    hysteria2://<URL_ENCODED_HY2_PASSWORD>@<YOUR_SERVER_IP>:8443?sni=www.microsoft.com&insecure=1&mport=30000-45000&upmbps=30&downmbps=100#MyVPS-Hy2
    ```
  - **VLESS + Reality**：
    ```text
    vless://<UUID>@<YOUR_SERVER_IP>:443?encryption=none&flow=xtls-rprx-vision&security=reality&sni=www.samsung.com&fp=chrome&pbk=<PUBLIC_KEY>&sid=<SHORT_ID>&type=tcp#MyVPS-Reality
    ```
- **导入后必检项**：
  1. 点开 Hysteria2 节点详情，确认 **Allow Insecure（允许不安全）** 已开启、**端口跳跃（Port Hopping / mport）** 为 `30000-45000`。
  2. 将 Shadowrocket 的 **全局路由** 设为 **配置（Config）**，并配合白名单规则文件（如社区维护的 Shadowrocket 白名单 `.conf`），避免国内 App 流量白白消耗 VPS 月流量配额。

---

## 第六步：流量监控与日常运维

### 1. 安装 `vnstat` 监控月流量

```bash
apt update && apt install -y vnstat
systemctl enable --now vnstat

vnstat -m    # 查看按月统计（留出 10% 余量，避免超额扣费）
vnstat -d    # 查看每日流量
vnstat -l    # 查看实时网速
```

### 2. 常用运维验证命令

```bash
# 查看双服务运行状态
systemctl status hysteria-server xray

# 实时查看日志（排错最有效）
journalctl -u hysteria-server -f
journalctl -u xray -f

# 验证端口跳跃 NAT 规则是否命中（客户端联网后 pkts/bytes 应持续递增）
iptables -t nat -L PREROUTING -n -v
```

---

## 常见问题与排错指南（FAQ）

### 1. 开启代理 TUN 模式后，SSH 无法连接服务器（`Connection closed by ... port 22`）
- **原因**：TUN 模式接管了系统全局流量，`MATCH` 规则把 SSH 流量路由进了尚未配置好的代理节点，形成死循环。
- **解决**：
  1. 临时关闭客户端的 TUN 模式（或通过云厂商网页端 Web Console 登录服务器）。
  2. 在客户端 `rules:` 最顶部加入 `- IP-CIDR,<YOUR_SERVER_IP>/32,DIRECT`。

### 2. VLESS + Reality 一直提示 `timeout`
按以下顺序逐项排查：
1. **防火墙拦截**：确认 `iptables` 与云厂商安全组均已放行 `TCP 443`。
2. **系统时间偏差**：在服务端执行 `timedatectl set-ntp true`。
3. **新版 Xray 后量子握手兼容性**：Xray **v26.9.8+** 服务端强制要求客户端在 TLS ClientHello 中携带 `X25519MLKEM768`，否则服务端日志会报 `REALITY: processed invalid connection`。
   - **解法 A（推荐）**：升级客户端内核（Mihomo `>= v1.19.34`），并在 `reality-opts:` 下添加 `support-x25519mlkem768: true`。
   - **解法 B（兼容老旧移动端）**：将服务端 Xray 降级至 `v26.6.27`：
     ```bash
     bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install --version v26.6.27
     ```

### 3. 测速速度很快但延迟异常，或服务端几乎没有流量
- **原因**：客户端规则中缺少 `MATCH` 兜底规则，导致测速网站（如 `speed.cloudflare.com`）直接走了 `DIRECT`（本地直连）。
- **解决**：确认规则末尾存在 `MATCH` 兜底规则，并将测速域名加入代理规则，或在测速时临时切换为「全局（Global）」模式验证。

### 4. YAML 导入客户端报语法错误
- **原因**：YAML 对缩进极其敏感，`- name:` 下方属性与 `- ` 后的 `name` 必须对齐（通常为 2 空格缩进），且不可混用 Tab。
- **解决**：严格对照上方模板缩进，检查是否存在多余空格。
