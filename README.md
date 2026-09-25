# NAT小鸡搭建教程（开着梯子导入，更新不了）

> 针对 **64MB ~ 512MB 极小内存 NAT / 独享小鸡** 打造的纯原生、零额外依赖一键自建与订阅方案。  
> 采用 **官方静态编译 sing-box (VLESS-REALITY)** + **系统原生 Perl 5 极轻量动态流量订阅**。  
> **拒绝死机**：常驻内存仅约 15MB，杜绝 OOM。  
> **直击痛点**：彻底解决“开着梯子/代理更新订阅报错 `failed to fetch remote profile`”以及“高位端口被拦截”。

---

## 核心痛点解析：为什么“开着梯子导入更新不了”？

1. **商业代理/机场防火墙拦截高位非标端口**：  
   当你开着全局代理或 TUN 模式时，请求会通过节点出站。商业节点防火墙通常只允许向外发起 `80` 或 `443` 的 HTTP/HTTPS 请求。而 NAT 小鸡的订阅端口一般是几万的高位端口（如 `59688`、`45001`），且走的是纯文本 HTTP，直接会被中间代理丢包或切断连接，导致客户端提示 `failed to fetch remote profile`。
2. **中间代理重写请求行**：  
   很多代理服务在转发时会将 `/token=xxx` 改写为绝对 URI，导致严格匹配的轻量 Web 守护直接拒连。

### 优雅解决方案：
- **服务端放宽匹配**：脚本使用全路径模糊匹配，无论中间代理是否改写 URI，只要携带正确 Token 一律正常吐出配置。
- **客户端一劳永逸直连（免关梯子）**：在本地代理软件规则中把小鸡 IP 设为 `DIRECT` 直连，以后无论梯子开不开，更新订阅都会秒拉取。

---

## 全自动化一键部署脚本（终端直接粘贴运行）

登录新小鸡的 root 终端，直接**完整复制**并**粘贴执行**下面这一整段脚本：

```bash
bash -c "$(cat <<'SCRIPT_EOF'
#!/usr/bin/env bash
set -e

clear
echo "=========================================================="
echo "    NAT小鸡 VLESS-REALITY + 原生Perl订阅 一键安装脚本     "
echo "=========================================================="

# 1. 自动探测公网 IPv4
AUTO_IP=$(curl -s4m 5 [https://api.ipify.org](https://api.ipify.org) || curl -s4m 5 [https://icanhazip.com](https://icanhazip.com) || echo "")
read -p "请输入本机公网 IPv4 地址 [默认: ${AUTO_IP}]: " INPUT_IP
SERVER_IP=${INPUT_IP:-$AUTO_IP}
if [ -z "$SERVER_IP" ]; then
    echo "[-] 错误: 未能获取到公网 IP，请手动输入！"
    exit 1
fi

# 2. 交互输入端口与信息
read -p "请输入节点映射端口 (VLESS-REALITY, 如 59689): " NODE_PORT
read -p "请输入订阅映射端口 (HTTP订阅, 如 59688): " SUB_PORT
read -p "请输入计划展示的总额度 (单位GB, 如 400): " TRAFFIC_GB
read -p "请输入节点与订阅卡片显示名称 [默认: 日本自建（400g）]: " INPUT_NAME
NODE_NAME=${INPUT_NAME:-"日本自建（400g）"}

RAND_TOKEN=$(tr -dc A-Za-z0-9 </dev/urandom | head -c 16)
read -p "请输入订阅防扫描 Token [默认随机生成: ${RAND_TOKEN}]: " INPUT_TOKEN
SUB_TOKEN=${INPUT_TOKEN:-$RAND_TOKEN}

# 3. 自动探测主网卡
DEFAULT_IFACE=$(ip route get 8.8.8.8 2>/dev/null | awk '{print $5}' | head -n1)
DEFAULT_IFACE=${DEFAULT_IFACE:-"eth0"}
read -p "请输入流量统计网卡名称 [默认探测: ${DEFAULT_IFACE}]: " INPUT_IFACE
NET_IFACE=${INPUT_IFACE:-$DEFAULT_IFACE}

echo ""
echo "[+] 正在自动生成安全凭证 (UUID、REALITY密钥对、ShortID)..."

# 4. 自动生成密钥
UUID=$(sing-box generate uuid)
KEYPAIR=$(sing-box generate reality-keypair)
PRIVATE_KEY=$(echo "$KEYPAIR" | grep "PrivateKey" | awk '{print $2}')
PUBLIC_KEY=$(echo "$KEYPAIR" | grep "PublicKey" | awk '{print $2}')
SHORT_ID=$(openssl rand -hex 8)

TOTAL_BYTES=$(awk -v gb="$TRAFFIC_GB" 'BEGIN {printf "%.0f", gb * 1024 * 1024 * 1024}')
EXPIRE_TIME=$(($(date +%s) + 30 * 86400))

# 建立专属目录并做运行备份
mkdir -p /opt/sing-box/ui /opt/sing-box/backup

# 5. 写入 sing-box 服务端配置
cat <<EOF> /opt/sing-box/config.json
{
  "log": {
    "level": "warn"
  },
  "inbounds": [
    {
      "type": "vless",
      "tag": "vless-in",
      "listen": "::",
      "listen_port": ${NODE_PORT},
      "users": [
        {
          "uuid": "${UUID}",
          "flow": "xtls-rprx-vision"
        }
      ],
      "tls": {
        "enabled": true,
        "server_name": "gateway.icloud.com",
        "reality": {
          "enabled": true,
          "handshake": {
            "server": "gateway.icloud.com",
            "server_port": 443
          },
          "private_key": "${PRIVATE_KEY}",
          "short_id": [
            "${SHORT_ID}"
          ]
        }
      }
    }
  ],
  "outbounds": [
    {
      "type": "direct",
      "tag": "direct"
    }
  ]
}
EOF

# 6. 写入客户端分流 YAML 模板 (集成 LoyalSoldier 精细规则)
cat <<EOF> /opt/sing-box/ui/index.html
port: 7890
socks-port: 7891
allow-lan: false
mode: rule
log-level: info

dns:
  enable: true
  listen: 0.0.0.0:1053
  ipv6: false
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16
  nameserver:
    - 223.5.5.5
    - 119.29.29.29
  fallback:
    - [https://dns.google/dns-query](https://dns.google/dns-query)
    - [https://1.1.1.1/dns-query](https://1.1.1.1/dns-query)
  fallback-filter:
    geoip: true
    geoip-code: CN

proxies:
  - name: "${NODE_NAME}"
    type: vless
    server: ${SERVER_IP}
    port: ${NODE_PORT}
    uuid: ${UUID}
    network: tcp
    tls: true
    udp: false
    flow: xtls-rprx-vision
    servername: gateway.icloud.com
    reality-opts:
      public-key: ${PUBLIC_KEY}
      short-id: ${SHORT_ID}
    client-fingerprint: chrome

proxy-groups:
  - name: "国外流量"
    type: select
    proxies: ["${NODE_NAME}", DIRECT]
  - name: "国外媒体"
    type: select
    proxies: ["${NODE_NAME}", "国外流量"]
  - name: "AI平台"
    type: select
    proxies: ["${NODE_NAME}", "国外流量"]
  - name: "微软服务"
    type: select
    proxies: [DIRECT, "${NODE_NAME}"]
  - name: "苹果服务"
    type: select
    proxies: [DIRECT, "${NODE_NAME}"]
  - name: "漏网之鱼"
    type: select
    proxies: ["国外流量", DIRECT]

rule-providers:
  reject:
    type: http
    behavior: domain
    url: "[https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/reject.txt](https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/reject.txt)"
    path: ./ruleset/reject.yaml
    interval: 86400
  icloud:
    type: http
    behavior: domain
    url: "[https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/icloud.txt](https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/icloud.txt)"
    path: ./ruleset/icloud.yaml
    interval: 86400
  apple:
    type: http
    behavior: domain
    url: "[https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/apple.txt](https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/apple.txt)"
    path: ./ruleset/apple.yaml
    interval: 86400
  google:
    type: http
    behavior: domain
    url: "[https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/google.txt](https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/google.txt)"
    path: ./ruleset/google.yaml
    interval: 86400
  proxy:
    type: http
    behavior: domain
    url: "[https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/proxy.txt](https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/proxy.txt)"
    path: ./ruleset/proxy.yaml
    interval: 86400
  direct:
    type: http
    behavior: domain
    url: "[https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/direct.txt](https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/direct.txt)"
    path: ./ruleset/direct.yaml
    interval: 86400
  gfw:
    type: http
    behavior: domain
    url: "[https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/gfw.txt](https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/gfw.txt)"
    path: ./ruleset/gfw.yaml
    interval: 86400
  tld-not-cn:
    type: http
    behavior: domain
    url: "[https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/tld-not-cn.txt](https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/tld-not-cn.txt)"
    path: ./ruleset/tld-not-cn.yaml
    interval: 86400
  telegramcidr:
    type: http
    behavior: ipcidr
    url: "[https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/telegramcidr.txt](https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/telegramcidr.txt)"
    path: ./ruleset/telegramcidr.yaml
    interval: 86400
  cncidr:
    type: http
    behavior: ipcidr
    url: "[https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/cncidr.txt](https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/cncidr.txt)"
    path: ./ruleset/cncidr.yaml
    interval: 86400
  lancidr:
    type: http
    behavior: ipcidr
    url: "[https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/lancidr.txt](https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/lancidr.txt)"
    path: ./ruleset/lancidr.yaml
    interval: 86400

rules:
  - RULE-SET,lancidr,DIRECT,no-resolve
  - RULE-SET,reject,REJECT
  - DOMAIN-SUFFIX,openai.com,AI平台
  - DOMAIN-SUFFIX,chatgpt.com,AI平台
  - DOMAIN-SUFFIX,anthropic.com,AI平台
  - DOMAIN-SUFFIX,claude.ai,AI平台
  - DOMAIN-SUFFIX,youtube.com,国外媒体
  - DOMAIN-SUFFIX,googlevideo.com,国外媒体
  - DOMAIN-SUFFIX,netflix.com,国外媒体
  - RULE-SET,icloud,苹果服务
  - RULE-SET,apple,苹果服务
  - DOMAIN-SUFFIX,microsoft.com,微软服务
  - DOMAIN-SUFFIX,windowsupdate.com,微软服务
  - RULE-SET,google,国外流量
  - RULE-SET,proxy,国外流量
  - RULE-SET,gfw,国外流量
  - RULE-SET,tld-not-cn,国外流量
  - RULE-SET,telegramcidr,国外流量,no-resolve
  - RULE-SET,direct,DIRECT
  - RULE-SET,cncidr,DIRECT
  - GEOIP,CN,DIRECT
  - MATCH,漏网之鱼
EOF

# 7. 写入原生 Perl 订阅服务端
cat <<EOF> /opt/sing-box/sub.pl
use strict;
use warnings;
use IO::Socket::INET;

my \$SECRET_TOKEN = "${SUB_TOKEN}";
my \$IFACE = "${NET_IFACE}";

sub get_network_traffic {
    my (\$rx_curr, \$tx_curr) = (0, 0);
    if (open my \$fh, '<', '/proc/net/dev') {
        while (my \$line = <\$fh>) {
            if (\$line =~ /^\\s*\$IFACE:\\s*(\\d+)(?:\\s+\\d+){7}\\s+(\\d+)/) {
                \$rx_curr = \$1;
                \$tx_curr = \$2;
                last;
            }
        }
        close \$fh;
    }
    return (\$rx_curr, \$tx_curr);
}

my \$file = '/opt/sing-box/ui/index.html';
open my \$fh, '<', \$file or die "Cannot open \$file: \$!";
my \$body = do { local \$/; <\$fh> };
close \$fh;

my \$len = length(\$body);

my \$server = IO::Socket::INET->new(
    LocalPort => ${SUB_PORT},
    Proto     => 'tcp',
    Listen    => 20,
    ReuseAddr => 1
) or die "Cannot bind to port ${SUB_PORT}: \$!";

while (my \$client = \$server->accept()) {
    my \$req_line = <\$client> || "";
    if (index(\$req_line, \$SECRET_TOKEN) != -1) {
        my (\$rx, \$tx) = get_network_traffic();
        my \$resp = "HTTP/1.1 200 OK\\r\\n" .
                   "Content-Type: text/yaml; charset=utf-8\\r\\n" .
                   "Content-Disposition: attachment; filename=\\"${NODE_NAME}.yaml\\"; filename*=UTF-8''${NODE_NAME}.yaml\\r\\n" .
                   "Content-Length: \$len\\r\\n" .
                   "Subscription-Userinfo: upload=\$tx; download=\$rx; total=${TOTAL_BYTES}; expire=${EXPIRE_TIME}\\r\\n" .
                   "Connection: close\\r\\n\\r\\n" .
                   \$body;
        print \$client \$resp;
    }
    close \$client;
}
EOF

# 8. 写入 Systemd 服务
cat <<'EOF' > /etc/systemd/system/clash-sub.service
[Unit]
Description=Clash Subscription Server
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/perl /opt/sing-box/sub.pl
Restart=always
RestartSec=2s

[Install]
WantedBy=multi-user.target
EOF

# 备份初始可用状态
cp /opt/sing-box/config.json /opt/sing-box/backup/config.json.bak
cp /opt/sing-box/sub.pl /opt/sing-box/backup/sub.pl.bak
cp /opt/sing-box/ui/index.html /opt/sing-box/backup/index.html.bak

# 启动并使能服务
systemctl daemon-reload
systemctl enable --now sing-box clash-sub
systemctl restart sing-box clash-sub

clear
echo "=========================================================="
echo "               小鸡部署成功！配置信息如下                 "
echo "=========================================================="
echo "节点名称:  ${NODE_NAME}"
echo "公网 IP :  ${SERVER_IP}"
echo "节点端口:  ${NODE_PORT} (VLESS-REALITY)"
echo "订阅端口:  ${SUB_PORT} (原生 Perl 动态流量)"
echo "已设额度:  ${TRAFFIC_GB} GB"
echo ""
echo "专属客户端导入链接 (复制直接导入 Clash / Clash Verge):"
echo "http://${SERVER_IP}:${SUB_PORT}/token=${SUB_TOKEN}&name=${NODE_NAME}.yaml"
echo ""
echo "VLESS 单节点直链 (URI):"
echo "vless://${UUID}@${SERVER_IP}:${NODE_PORT}?security=reality&encryption=none&pbk=${PUBLIC_KEY}&headerType=none&fp=chrome&type=tcp&flow=xtls-rprx-vision&sni=gateway.icloud.com&sid=${SHORT_ID}#${NODE_NAME}"
echo "=========================================================="
echo "【避坑提示】如果开着梯子更新报错 failed to fetch remote profile："
echo "只要在代理客户端的规则中，把 ${SERVER_IP} 设为 DIRECT（直连），"
echo "以后开着梯子也能秒拉取更新，免去开关梯子的烦恼！"
echo "=========================================================="
SCRIPT_EOF
)"
