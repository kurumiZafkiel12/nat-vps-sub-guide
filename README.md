# NAT小鸡极简自建节点与动态流量订阅指南

> **适用场景**：内存仅有 64MB ~ 512MB 的 NAT VPS、独立 IP VPS。  
> **核心组件**：官方静态编译 `sing-box` (VLESS-REALITY-Vision) + 原生 `Perl 5` 极轻量订阅守护进程。  
> **设计目标**：常驻内存小于 15MB、不爆内存宕机、读取系统内核真实流量扣额、自带到期时间与全套精细分流规则，并彻底解决**“开着梯子/全局代理时更新订阅报 failed to fetch remote profile”**的问题。

---

## 目录
- [0. 核心原理解析：为什么开着代理会更新失败？](#0-核心原理解析为什么开着代理会更新失败)
- [1. 部署前准备：记录你的小鸡参数](#1-部署前准备记录你的小鸡参数)
- [2. 步骤一：安装与验证基础环境](#2-步骤一安装与验证基础环境)
- [3. 步骤二：环境变量定义（配置参数注入）](#3-步骤二环境变量定义配置参数注入)
- [4. 步骤三：生成专属 REALITY 密钥与用户凭据](#4-步骤三生成专属-reality-密钥与用户凭据)
- [5. 步骤四：配置并启动 sing-box 服务端](#5-步骤四配置并启动-sing-box-服务端)
- [6. 步骤五：生成客户端分流配置文件 (LoyalSoldier)](#6-步骤五生成客户端分流配置文件-loyalsoldier)
- [7. 步骤六：部署原生 Perl 动态流量订阅服务端](#7-步骤六部署原生-perl-动态流量订阅服务端)
- [8. 步骤七：本地链路验证与客户端订阅导入](#8-步骤七本地链路验证与客户端订阅导入)
- [9. 核心实战技巧：如何实现开着代理也能秒级更新订阅？](#9-核心实战技巧如何实现开着代理也能秒级更新订阅)
- [10. 运维与灾备：配置备份与一键无损回滚](#10-运维与灾备配置备份与一键无损回滚)

---

## 0. 核心原理解析：为什么开着代理会更新失败？

在客户端（如 Clash Verge、Flclash、v2rayN）开启“系统代理”或“TUN 模式”时，本机所有向外的网络请求都会被代理核心接管，并转发至当前连接的代理节点。

此时拉取自建订阅经常失败，主要有两个原因：
1. **商业节点防火墙拦截高位端口**：绝大多数商业节点或机场节点的出站安全策略只放行 `80` (标准 HTTP) 和 `443` (标准 HTTPS)。NAT 小鸡分配给订阅服务的端口通常为高位端口（如 `59688`、`45001` 等），且传输的是纯明文 HTTP 流量，直接被商业节点的安全策略切断或丢弃，导致客户端报 `failed to fetch remote profile`。
2. **中间代理改写 HTTP 请求行**：许多代理服务器在转发 HTTP 请求时，会将相对路径（如 `GET /token=... HTTP/1.1`）改写为绝对 URI（如 `GET http://1.2.3.4:59688/token=... HTTP/1.1`）。如果服务端的脚本匹配规则过于严苛，就会直接判定为非法路径并关闭连接。

### 对应解决策略：
- **服务端放宽鉴权**：Perl 守护进程使用子字符串匹配算法，只要请求行中包含正确的 Token 字符串即放行，无论路径是否被中间代理改写都能正常返回。
- **客户端配置直连规则（推荐）**：在本地客户端的规则库中，将该小鸡的公网 IP 写入直连规则（`DIRECT`）。当更新订阅时，流量会绕开当前挂着的代理，走本地物理宽带直连出站，彻底避开高位端口拦截。

---

## 1. 部署前准备：记录你的小鸡参数

在动手前，新建一个文本记录以下参数，后续代码中的变量会用到：

| 参数项 | 说明 | 示例值 |
| :--- | :--- | :--- |
| **`SERVER_IP`** | 小鸡的公网 IPv4 | `66.154.108.15` |
| **`NODE_PORT`** | 节点映射端口 (VLESS-REALITY) | `59689` |
| **`SUB_PORT`** | 订阅映射端口 (HTTP 订阅) | `59688` |
| **`NODE_NAME`** | 客户端节点与卡片显示的名称 | `日本自建（400g）` |
| **`TRAFFIC_GB`** | 计划展示的总额度（单位：GB） | `400` |
| **`SUB_TOKEN`** | 防全网端口扫描探测的自定义密码串 | `jp_token_8899` |
| **`NET_IFACE`** | 流量统计网卡名称（运行 `ip -br link` 查看） | `eth0` |

---

## 2. 步骤一：安装与验证基础环境

登录 VPS 终端，运行以下命令安装基础组件并获取官方静态 `sing-box`：

```bash
# 1. 更新软件包列表并安装必要基础工具
apt update && apt install -y curl perl openssl

# 2. 检查并安装官方静态编译的 sing-box (若已安装则跳过)
if ! command -v sing-box &> /dev/null; then
    echo "正在安装官方静态 sing-box..."
    ARCH=$(uname -m)
    if [ "$ARCH" = "x86_64" ]; then
        SB_ARCH="amd64"
    elif [ "$ARCH" = "aarch64" ]; then
        SB_ARCH="arm64"
    else
        SB_ARCH="amd64"
    fi
    curl -Lo /usr/local/bin/sing-box [https://github.com/SagerNet/sing-box/releases/download/v1.11.4/sing-box-1.11.4-linux-$](https://github.com/SagerNet/sing-box/releases/download/v1.11.4/sing-box-1.11.4-linux-$){SB_ARCH}.tar.gz
    tar -zxvf /usr/local/bin/sing-box -C /tmp/
    mv /tmp/sing-box-*/sing-box /usr/local/bin/sing-box
    chmod +x /usr/local/bin/sing-box
    rm -rf /tmp/sing-box*
fi

# 3. 验证程序版本
sing-box version
perl -v | head -n 2
```

---

## 3. 步骤二：环境变量定义（配置参数注入）

将你在“步骤 1”中确定的实际参数填入下方的变量定义中，然后在终端中整段粘贴运行。后续步骤会自动调用这些变量，无需手动修改配置文件。

```bash
# =================【用户参数自定义区域】=================
SERVER_IP="66.154.108.15"            # 替换为你小鸡的公网 IPv4
NODE_PORT="59689"                    # 替换为你小鸡的节点外部端口
SUB_PORT="59688"                     # 替换为你小鸡的订阅外部端口
NODE_NAME="日本自建（400g）"         # 替换为你想要显示的节点名称
TRAFFIC_GB=400                       # 替换为总配额 (单位: GB)
SUB_TOKEN="jp_token_8899"            # 替换为你自定义的防扫 Token
NET_IFACE="eth0"                     # 统计流量的网卡名 (通过 ip -br link 查看)
# ========================================================

# 自动换算总字节数与到期时间戳 (默认设为当前时间往后推 30 天)
TOTAL_BYTES=$(( TRAFFIC_GB * 1024 * 1024 * 1024 ))
EXPIRE_TIME=$(( $(date +%s) + 30 * 86400 ))

# 建立程序与备份目录
mkdir -p /opt/sing-box/ui /opt/sing-box/backup
```

---

## 4. 步骤三：生成专属 REALITY 密钥与用户凭据

执行以下命令，脚本会自动生成 `UUID`、REALITY 密钥对及 `Short ID`，并将它们暂存在当前终端的环境变量中：

```bash
# 1. 生成并捕获 UUID
UUID=$(sing-box generate uuid)

# 2. 生成并解析 REALITY 密钥对
KEYPAIR=$(sing-box generate reality-keypair)
PRIVATE_KEY=$(echo "$KEYPAIR" | grep "PrivateKey" | awk '{print $2}')
PUBLIC_KEY=$(echo "$KEYPAIR" | grep "PublicKey" | awk '{print $2}')

# 3. 生成 8 字节十六进制短 ID
SHORT_ID=$(openssl rand -hex 8)

# 4. 在终端展示生成的凭据 (建议截图或保存在本地备忘录中)
echo "================ 你的专属安全凭证 ================"
echo "UUID        : ${UUID}"
echo "PrivateKey  : ${PRIVATE_KEY}"
echo "PublicKey   : ${PUBLIC_KEY}"
echo "Short ID    : ${SHORT_ID}"
echo "=================================================="
```

---

## 5. 步骤四：配置并启动 sing-box 服务端

执行以下命令，将自动引用上述变量生成 `sing-box` 配置文件，并配置 `systemd` 服务开机自启：

```bash
# 1. 写入 sing-box 服务端核心配置
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

# 2. 写入 systemd 守护进程文件
cat <<'EOF' > /etc/systemd/system/sing-box.service
[Unit]
Description=sing-box service
After=network.target nss-lookup.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/sing-box run -c /opt/sing-box/config.json
Restart=always
RestartSec=3s
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF

# 3. 加载并启动 sing-box
systemctl daemon-reload
systemctl enable --now sing-box
systemctl restart sing-box

# 4. 验证运行状态 (应显示 active running)
systemctl status sing-box --no-pager
```

---

## 6. 步骤五：生成客户端分流配置文件 (LoyalSoldier)

此步骤生成供客户端拉取的完整 Clash 规则配置文件，并保存至 `/opt/sing-box/ui/index.html`。该配置集成了 **LoyalSoldier** 规则集，包含国内直连、海外代理、微软/苹果服务分流及 AI 平台独立策略：

```bash
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
```

---

## 7. 步骤六：部署原生 Perl 动态流量订阅服务端

执行以下命令，部署一个常驻内存仅约 1.5MB 的纯原生 Perl 5 HTTP 订阅守护进程。它会实时抓取 Linux 内核文件 `/proc/net/dev` 中的字节数，并在客户端请求时附带 `Subscription-Userinfo` 头部信息下发：

```bash
# 1. 写入 Perl 订阅服务端脚本
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

# 2. 写入 systemd 守护配置
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

# 3. 启动订阅守护进程
systemctl daemon-reload
systemctl enable --now clash-sub
systemctl restart clash-sub

# 4. 验证运行状态
systemctl status clash-sub --no-pager
```

---

## 8. 步骤七：本地链路验证与客户端订阅导入

### 1. 服务端本地验证
在终端执行以下命令，验证端口监听与订阅返回：

```bash
# 验证端口监听状态
ss -tulpn | grep -E "(${NODE_PORT}|${SUB_PORT})"

# 本地模拟拉取测试 (正常应返回 HTTP/1.1 200 OK 且附带 Subscription-Userinfo)
curl -i "[http://127.0.0.1](http://127.0.0.1):${SUB_PORT}/token=${SUB_TOKEN}" | head -n 8

echo ""
echo "=========================================================="
echo "配置成功！客户端订阅链接为："
echo "http://${SERVER_IP}:${SUB_PORT}/token=${SUB_TOKEN}&name=${NODE_NAME}.yaml"
echo "=========================================================="
```

### 2. 客户端导入操作
1. 打开客户端（如 Clash Verge Rev、Flclash 等）。
2. 在“订阅链接”输入框中粘贴上面打印的完整 URL：
   ```text
   http://你的公网IP:你的订阅端口/token=你的Token&name=你的节点名称.yaml
   ```
3. 点击“导入”。客户端将自动锁定卡片标题，并显示实时的流量额度进度条与到期时间。

---

## 9. 核心实战技巧：如何实现开着代理也能秒级更新订阅？

如果你平时开着全局代理或商业 VPN，直接在客户端点击“更新订阅”时，可能会遇到 `failed to fetch remote profile`。这是因为中间代理拦截了小鸡的高位端口。

### 一劳永逸的解决方案（无需关闭代理）：
在本地客户端中，把该小鸡的公网 IP 加入到直连规则中。以 Clash Verge Rev 为例：
1. 点击客户端左侧的 **订阅 (Profiles)**。
2. 找到你平时主力使用的代理订阅卡片，右键选择 **编辑扩展配置 / 脚本 (Edit Rules / Script)**，或进入全局规则配置。
3. 在 `rules:` 列表的最顶部，添加一条 IP-CIDR 直连规则：
   ```yaml
   rules:
     - IP-CIDR,你的小鸡公网IP/32,DIRECT,no-resolve
   ```
4. 保存配置并重新应用。

**生效机制**：  
当你开着系统代理更新该订阅时，客户端会优先匹配此规则，使发往该小鸡公网 IP 的请求直接走你本机的物理网络出站，不再经过商业节点转发，从而彻底规避非标端口阻断。

---

## 10. 运维与灾备：配置备份与一键无损回滚

### 1. 建立初始状态备份
在 VPS 终端执行以下命令，保存当前正常运行的所有配置：

```bash
mkdir -p /opt/sing-box/backup
cp -a /opt/sing-box/config.json /opt/sing-box/backup/config.json.bak
cp -a /opt/sing-box/sub.pl /opt/sing-box/backup/sub.pl.bak
cp -a /opt/sing-box/ui/index.html /opt/sing-box/backup/index.html.bak
```

### 2. 故障一键还原
如果后续误修改了任何配置导致服务异常，执行以下命令即可恢复初始状态：

```bash
cp /opt/sing-box/backup/config.json.bak /opt/sing-box/config.json
cp /opt/sing-box/backup/sub.pl.bak /opt/sing-box/sub.pl
cp /opt/sing-box/backup/index.html.bak /opt/sing-box/ui/index.html
systemctl restart sing-box clash-sub
```
