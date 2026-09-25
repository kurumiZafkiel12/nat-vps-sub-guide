# 全通用 NAT VPS 极简节点与动态流量订阅一键部署指南

> **适用场景**：所有 NAT VPS（独角鲸云、碳云、微基主机、各类廉价 NAT 小鸡）及普通独立 IP VPS。  
> **支持架构**：x86_64 (amd64) / aarch64 (arm64)，系统推荐 Debian 11/12 或 Ubuntu 20.04/22.04。  
> **核心组件**：官方静态 `sing-box` (VLESS-REALITY-Vision) + 原生 `Perl 5` 超轻量动态流量订阅。  
> **核心优势**：
> 1. **全通用**：支持“内外端口一致”与“内外端口不同（强制映射）”的所有 NAT 商家。
> 2. **零转义事故**：DNS 纯 IP 化，完全规避浏览器翻译插件改写 Markdown 超链接引起的 YAML 语法崩溃。
> 3. **原子级单步生成**：不使用脆弱的二次文本替换，杜绝空白密钥或端口冲突。
> 4. **双向防阻断**：自带开着代理也能秒级更新订阅的规则设计。

---

## 目录
- [0. 核心原理解析：NAT 端口映射模型](#0-核心原理解析nat-端口映射模型)
- [1. 第一步：在商家后台配置/查看端口转发](#1-第一步在商家后台配置查看端口转发)
- [2. 第二步：环境初始化与 sing-box 校验](#2-第二步环境初始化与-sing-box-校验)
- [3. 第三步：全通用交互式部署脚本](#3-第三步全通用交互式部署脚本)
- [4. 第四步：获取专属订阅并在客户端导入](#4-第四步获取专属订阅并在客户端导入)
- [5. 核心避坑：开着梯子更新订阅失败的解法](#5-核心避坑开着梯子更新订阅失败的解法)
- [6. 维护与救砖命令](#6-维护与救砖命令)

---

## 0. 核心原理解析：NAT 端口映射模型

NAT VPS 由于全母鸡共享同一个公网 IPv4，商家会通过网关进行端口映射。通常有两种模式：
* **模式 A（内外一致，如独角鲸云）**：你申请外部端口 `23456`，小鸡内部也直接监听 `23456`。
* **模式 B（内外不同，如多数传统 NAT 面板）**：商家分配给你一个外部端口（如 `45821`），但要求你小鸡内部必须监听指定的内网端口（如 `10086` 或 `10000~10020` 范围）。

本教程脚本全面兼容这两种模式，在交互输入时根据后台提示填入即可。

我们需要在小鸡上打通**两个服务**：
1. **节点服务**：运行 VLESS-REALITY 协议。
2. **订阅服务**：运行 Perl 动态流量统计与配置下发。

---

## 1. 第一步：在商家后台配置/查看端口转发

登录你的 NAT VPS 后台，找到 **端口转发 / Port Forwarding** 列表，确保有两个可用的 TCP 映射规则：

| 规则用途 | 外部端口 (公网访问用) | 内部端口 (小鸡自身监听用) | 协议 |
| :--- | :--- | :--- | :--- |
| **节点转发** | 记下商家分配的外部端口 A | 记下对应的内部端口 A | **TCP** |
| **订阅转发** | 记下商家分配的外部端口 B | 记下对应的内部端口 B | **TCP** |

> **注**：如果商家允许内外端口自定义（如独角鲸云），直接让外部端口等于内部端口（例如外部 `23456`，内部填 `23456`）。

---

## 2. 第二步：环境初始化与 sing-box 校验

登录小鸡终端（SSH 或网页控制台），整段复制执行以下命令。该脚本会自动检测 CPU 架构（amd64 / arm64），并通过国内与国际双通道拉取官方静态二进制包：

```bash
apt update && apt install -y curl perl openssl procps jq bc

# 自动匹配架构并下载 sing-box 静态包
ARCH_RAW=$(uname -m)
case "$ARCH_RAW" in
    x86_64)  ARCH="amd64" ;;
    aarch64) ARCH="arm64" ;;
    armv7*)  ARCH="armv7" ;;
    *) echo "未受支持的架构: $ARCH_RAW" && exit 1 ;;
esac

SB_VER="1.11.4"
TARGET="/usr/local/bin/sing-box"

rm -f /tmp/sb.tar.gz
curl -fsSL -o /tmp/sb.tar.gz "[https://ghfast.top/https://github.com/SagerNet/sing-box/releases/download/v$](https://ghfast.top/https://github.com/SagerNet/sing-box/releases/download/v$){SB_VER}/sing-box-${SB_VER}-linux-${ARCH}.tar.gz" || \
curl -fsSL -o /tmp/sb.tar.gz "[https://github.com/SagerNet/sing-box/releases/download/v$](https://github.com/SagerNet/sing-box/releases/download/v$){SB_VER}/sing-box-${SB_VER}-linux-${ARCH}.tar.gz"

if [ -f /tmp/sb.tar.gz ]; then
    tar -zxvf /tmp/sb.tar.gz -C /tmp/
    mv /tmp/sing-box-*/sing-box "$TARGET"
    chmod +x "$TARGET"
    rm -rf /tmp/sb* /tmp/sing-box*
fi

# 核心可执行验证
if "$TARGET" version &>/dev/null; then
    echo "=================================================="
    echo "sing-box 初始化成功，版本：" $("$TARGET" version | head -n1)
    echo "=================================================="
else
    echo "ERROR: sing-box 下载异常，请检查网络连通性后重试！"
    exit 1
fi
```

---

## 3. 第三步：全通用交互式部署脚本

整段复制并粘贴到终端回车。脚本会自动探测物理网卡与公网 IP，引导输入端口（支持内外一致或内外分离），自动生成 UUID、Reality 密钥，并原子化一次性生成所有配置文件：

```bash
cat <<'SH_EOF' > /root/universal_deploy.sh
#!/bin/bash
set -e

mkdir -p /opt/sing-box/ui /opt/sing-box/backup

echo "========================================================"
echo "          全通用 NAT VPS 节点与动态订阅部署引导          "
echo "========================================================"

# 防插件篡改探测公网 IP
P="h""t""t""p"
DETECT_IP=$(curl -s4m 3 "$P://ip.sb" || curl -s4m 3 "$P://ifconfig.me" || curl -s4m 3 "$P://api.ipify.org" || echo "")

read -p "1. 确认小鸡公网 IPv4 [探测结果: ${DETECT_IP}]: " INPUT_IP
SERVER_IP=${INPUT_IP:-$DETECT_IP}

# --- 节点端口配置 ---
read -p "2. 节点【外部公网端口】(客户端连接用): " EXT_NODE_PORT
read -p "   节点【内部监听端口】[若内外一致直接回车，默认: ${EXT_NODE_PORT}]: " INT_NODE_PORT
INT_NODE_PORT=${INT_NODE_PORT:-$EXT_NODE_PORT}

# --- 订阅端口配置 ---
read -p "3. 订阅【外部公网端口】(客户端拉取订阅用): " EXT_SUB_PORT
read -p "   订阅【内部监听端口】[若内外一致直接回车，默认: ${EXT_SUB_PORT}]: " INT_SUB_PORT
INT_SUB_PORT=${INT_SUB_PORT:-$EXT_SUB_PORT}

read -p "4. 客户端显示的卡片名称 [默认: 自建NAT节点]: " INPUT_NAME
NODE_NAME=${INPUT_NAME:-"自建NAT节点"}

read -p "5. 总流量额度(GB) [纯数字, 默认: 400]: " INPUT_TOTAL
TRAFFIC_GB=${INPUT_TOTAL:-400}

read -p "6. 已用流量底数(GB) [支持两位小数如 12.35，新机直接填 0]: " INPUT_USED
USED_GB=${INPUT_USED:-0}

read -p "7. 到期时间 [格式示例 2026-10-18，直接回车默认+30天]: " INPUT_DATE
if [ -n "$INPUT_DATE" ]; then
    EXPIRE_TIME=$(date -d "$INPUT_DATE" +%s 2>/dev/null || echo "$(( $(date +%s) + 30 * 86400 ))")
else
    EXPIRE_TIME=$(( $(date +%s) + 30 * 86400 ))
fi

DETECT_IFACE=$(ip route get 8.8.8.8 2>/dev/null | awk '{print $5}' | head -n1)
DETECT_IFACE=${DETECT_IFACE:-"eth0"}
read -p "8. 统计网卡名称 [默认探测: ${DETECT_IFACE}]: " INPUT_IFACE
NET_IFACE=${INPUT_IFACE:-$DETECT_IFACE}

RAND_TOKEN=$(tr -dc A-Za-z0-9 </dev/urandom | head -c 16)
read -p "9. 订阅安全 Token [直接回车随机: ${RAND_TOKEN}]: " INPUT_TOKEN
SUB_TOKEN=${INPUT_TOKEN:-$RAND_TOKEN}

# 现场生成密钥对
UUID=$(/usr/local/bin/sing-box generate uuid | tr -d '\r\n ')
KEYPAIR=$(/usr/local/bin/sing-box generate reality-keypair)
PRIVATE_KEY=$(echo "$KEYPAIR" | grep "PrivateKey" | awk '{print $2}' | tr -d '\r\n ')
PUBLIC_KEY=$(echo "$KEYPAIR" | grep "PublicKey" | awk '{print $2}' | tr -d '\r\n ')
SHORT_ID=$(openssl rand -hex 8 | tr -d '\r\n ')

TOTAL_BYTES=$(( TRAFFIC_GB * 1024 * 1024 * 1024 ))
BASE_USED_BYTES=$(awk "BEGIN {printf \"%.0f\", ${USED_GB} * 1024 * 1024 * 1024}")

# 1. 固化持久化变量
cat <<EOF> /opt/sing-box/my_env.sh
export SERVER_IP="${SERVER_IP}"
export EXT_NODE_PORT="${EXT_NODE_PORT}"
export INT_NODE_PORT="${INT_NODE_PORT}"
export EXT_SUB_PORT="${EXT_SUB_PORT}"
export INT_SUB_PORT="${INT_SUB_PORT}"
export NODE_NAME="${NODE_NAME}"
export SUB_TOKEN="${SUB_TOKEN}"
EOF

# 2. 写入服务端 sing-box 配置（严格监听内网端口与 IPv4 地址 0.0.0.0）
cat <<EOF> /opt/sing-box/config.json
{
  "log": {
    "level": "warn"
  },
  "inbounds": [
    {
      "type": "vless",
      "tag": "vless-in",
      "listen": "0.0.0.0",
      "listen_port": ${INT_NODE_PORT},
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

# 3. 写入客户端纯净 YAML（客户端连接使用公网外部端口，DNS 全面纯 IP 化）
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
    - 8.8.8.8
    - 1.1.1.1
  fallback-filter:
    geoip: true
    geoip-code: CN

proxies:
  - name: "${NODE_NAME}"
    type: vless
    server: ${SERVER_IP}
    port: ${EXT_NODE_PORT}
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
    proxies:
      - "${NODE_NAME}"
      - DIRECT
  - name: "国外媒体"
    type: select
    proxies:
      - "${NODE_NAME}"
      - "国外流量"
  - name: "AI平台"
    type: select
    proxies:
      - "${NODE_NAME}"
      - "国外流量"
  - name: "微软服务"
    type: select
    proxies:
      - DIRECT
      - "${NODE_NAME}"
  - name: "苹果服务"
    type: select
    proxies:
      - DIRECT
      - "${NODE_NAME}"
  - name: "漏网之鱼"
    type: select
    proxies:
      - "国外流量"
      - DIRECT

rules:
  - DOMAIN-SUFFIX,openai.com,AI平台
  - DOMAIN-SUFFIX,chatgpt.com,AI平台
  - DOMAIN-SUFFIX,anthropic.com,AI平台
  - DOMAIN-SUFFIX,claude.ai,AI平台
  - DOMAIN-SUFFIX,youtube.com,国外媒体
  - DOMAIN-SUFFIX,googlevideo.com,国外媒体
  - GEOIP,CN,DIRECT
  - MATCH,漏网之鱼
EOF

# 4. 写入 Perl 动态流量统计服务（监听小鸡内部订阅端口）
cat <<EOF> /opt/sing-box/sub.pl
use strict;
use warnings;
use IO::Socket::INET;

my \$SECRET_TOKEN = "${SUB_TOKEN}";
my \$IFACE = "${NET_IFACE}";
my \$BASE_USED = ${BASE_USED_BYTES};
my \$PORT = ${INT_SUB_PORT};
my \$NODE_NAME = "${NODE_NAME}";
my \$TOTAL_BYTES = ${TOTAL_BYTES};
my \$EXPIRE_TIME = ${EXPIRE_TIME};

my \$NAME_ENCODED = \$NODE_NAME;
\$NAME_ENCODED =~ s/([^a-zA-Z0-9_.~-])/sprintf("%%%02X", ord(\$1))/eg;

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
    my \$total_rx = \$rx_curr + \$BASE_USED;
    return (\$total_rx, \$tx_curr);
}

my \$file = '/opt/sing-box/ui/index.html';
open my \$fh, '<', \$file or die "Cannot open \$file: \$!";
my \$body = do { local \$/; <\$fh> };
close \$fh;

my \$len = length(\$body);

my \$server = IO::Socket::INET->new(
    LocalAddr => '0.0.0.0',
    LocalPort => \$PORT,
    Proto     => 'tcp',
    Listen    => 20,
    ReuseAddr => 1
) or die "Cannot bind to port \$PORT: \$!";

while (my \$client = \$server->accept()) {
    my \$req_line = <\$client> || "";
    if (index(\$req_line, \$SECRET_TOKEN) != -1) {
        my (\$rx, \$tx) = get_network_traffic();
        my \$resp = "HTTP/1.1 200 OK\\r\\n" .
                   "Content-Type: text/yaml; charset=utf-8\\r\\n" .
                   "Content-Disposition: attachment; filename=\\"config.yaml\\"; filename*=UTF-8''\${NAME_ENCODED}.yaml\\r\\n" .
                   "Content-Length: \$len\\r\\n" .
                   "Subscription-Userinfo: upload=\$tx; download=\$rx; total=\${TOTAL_BYTES}; expire=\${EXPIRE_TIME}\\r\\n" .
                   "Connection: close\\r\\n\\r\\n" .
                   \$body;
        print \$client \$resp;
    }
    close \$client;
}
EOF

# 5. 配置 systemd 系统服务
cat <<'EOF' > /etc/systemd/system/sing-box.service
[Unit]
Description=sing-box service
After=network.target

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

# 语法测试与加载启动
/usr/local/bin/sing-box check -c /opt/sing-box/config.json
systemctl daemon-reload
systemctl enable --now sing-box clash-sub
systemctl restart sing-box clash-sub

echo "--------------------------------------------------------"
echo "部署完成！内部监听状态验证："
ss -tulpn | grep -E "(${INT_NODE_PORT}|${INT_SUB_PORT})"
echo "--------------------------------------------------------"
SH_EOF

bash /root/universal_deploy.sh
```

---

## 4. 第四部分：获取专属订阅并在客户端导入

执行完成后，在控制台运行以下命令输出客户端专属链接：

```bash
source /opt/sing-box/my_env.sh
echo "=========================================================="
echo "客户端专属订阅导入链接："
echo "http://${SERVER_IP}:${EXT_SUB_PORT}/token=${SUB_TOKEN}&name=${NODE_NAME}.yaml"
echo "=========================================================="
```

### 客户端导入与使用（以 Clash Verge 为例）：
1. 复制终端输出的 `http://${SERVER_IP}:${EXT_SUB_PORT}/...` 完整链接。
2. 打开 **Clash Verge**，进入 **“订阅 (Profiles)”**。
3. 粘贴到输入框中，点击 **“导入 (Import)”**。
4. 订阅卡片将自动显示节点名称、到期时间和动态真实流量进度条。
5. 切换到 **“代理 (Proxies)”** 界面，点击闪电测速即可正常显示绿色延迟。

---

## 5. 核心避坑：开着梯子更新订阅失败的解法

在电脑已经开启了系统代理（或开着其他机场梯子）的情况下，点击自建订阅的“更新”，很多客户端会报错 `failed to fetch remote profile`。这是因为中间商业代理节点为了安全，通常会拦截非标准的随机高位外部端口。

### 一劳永逸解法（无需每次手动开关梯子）：
1. 在 Clash Verge 中打开你平时主力使用的那个订阅卡片，右键点击选择 **“编辑扩展配置 (Edit Rules / Script)”**。
2. 在规则列表（`rules:`）的最顶部添加一行公网 IP 直连规则：
   ```yaml
   rules:
     - IP-CIDR,你的NAT小鸡公网IP/32,DIRECT,no-resolve
   ```
3. 保存并刷新应用。

此时只要拉取或更新该小鸡的订阅，流量便会自动走本地宽带直连出站，彻底避开商业代理节点的端口拦截。

---

## 6. 维护与救砖命令

* **查看当前小鸡保存的配置参数**：
  ```bash
  cat /opt/sing-box/my_env.sh
  ```
* **一键重启两项服务**：
  ```bash
  systemctl restart sing-box clash-sub
  ```
* **服务运行状态与端口自检**：
  ```bash
  systemctl status sing-box --no-pager
  ss -tulpn | grep -E "(sing-box|perl)"
  ```
