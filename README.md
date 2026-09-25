# 全通用 NAT VPS 极简节点与动态流量订阅一键部署指南

> **适用场景**：独角鲸云、碳云、微基主机等所有 NAT VPS 商家及普通独立 IP VPS。  
> **支持架构**：x86_64 (amd64) / aarch64 (arm64)，系统推荐 Debian 11/12 或 Ubuntu 20.04/22.04。  
> **核心组合**：官方静态 `sing-box` (VLESS-REALITY-Vision) + 原生 `Perl 5` 极轻量动态流量订阅。  
> **核心优势**：
> 1. **全通用**：支持“内外端口一致”与“内外端口不同（强制映射）”的所有 NAT 商家。
> 2. **零转义事故**：DNS 纯 IP 化，完全规避浏览器翻译插件改写 Markdown 超链接引起的 YAML 语法崩溃。
> 3. **原子级单步生成**：不使用脆弱的二次文本替换，杜绝空白密钥或端口冲突。
> 4. **双向防阻断**：自带开着代理也能秒级更新订阅的规则设计。

---

## 目录
- [0. 什么是 NAT 小鸡？（通俗科普）](#0-什么是-nat-小鸡通俗科普)
- [1. 第一部分：平台注册、充值与开机（以独角鲸云为例）](#1-第一部分平台注册充值与开机以独角鲸云为例)
- [2. 第二部分：NAT 端口转发规则配置（核心关键）](#2-第二部分nat-端口转发规则配置核心关键)
- [3. 第三部分：进入控制台与全通用一键部署](#3-第三部分进入控制台与全通用一键部署)
  - [步骤一：安装核心依赖环境与 sing-box 校验](#步骤一安装核心依赖环境与-sing-box-校验)
  - [步骤二：全通用交互式部署脚本运行](#步骤二全通用交互式部署脚本运行)
- [4. 第四部分：查看客户端导入链接与 Clash Verge 导入](#4-第四部分查看客户端导入链接与-clash-verge-导入)
- [5. 第五部分：核心防阻断技巧（开着梯子也能秒级更新订阅）](#5-第五部分核心防阻断技巧开着梯子也能秒级更新订阅)
- [6. 常用维护与救砖命令](#6-常用维护与救砖命令)

---

## 0. 什么是 NAT 小鸡？（通俗科普）

* **独立 IP VPS（普通服务器）**：类似于独门独栋别墅，有独立的门牌号（独立公网 IP），所有端口都可以对外开放监听。
* **NAT 小鸡（共享型服务器）**：类似于合租大型公寓楼。
  * **共享大门**：整台物理母机上的所有合租用户共享同一个公网 IPv4。
  * **端口转发进屋**：外网无法通过任意端口找到你的小鸡，必须通过商家路由器进行“端口映射”。商家会分配给你的房间几个专属的外部端口（例如分配 `23456`、`34567`）。外网通过访问 `公网IP:外部端口`，路由器才会准确把流量转发给你的小鸡内部。
  * **两种映射模式**：
    * **模式 A（内外一致，如独角鲸云）**：外部端口和内部端口相同。
    * **模式 B（内外不同，如部分传统 NAT）**：商家分配外部端口（如 `45821`），小鸡内部监听指定内网端口（如 `10086`）。本教程全面支持这两种模式。

---

## 1. 第一部分：平台注册、充值与开机（以独角鲸云为例）

### 1. 访问网站与注册账号
打开独角鲸云平台入口：`https://dash.fuckip.me/login`，点击 **“立即注册”**（已有账号可直接登录）。

![独角鲸云登录与注册入口](https://github.com/user-attachments/assets/eece47e0-a361-4072-932c-3cb961d44742)

选择合适的注册认证方式完成登录（支持 Google 或 GitHub 快捷授权）：

![选择注册登录方式](https://github.com/user-attachments/assets/fd196e13-32c4-4762-af2c-3bb32b9dbe29)

### 2. 账号充值
进入用户后台首页：

![独角鲸云控制台首页](https://github.com/user-attachments/assets/14165be5-1fe4-4729-9624-0f41d89993cb)

点击左侧菜单栏的 **“账单充值”**，选择合适的支付方式（支持信用卡、支付宝、微信或加密货币）：

![充值方式与金额选择](https://github.com/user-attachments/assets/aaed142f-6531-4964-8cb9-0c42e63652dd)

### 3. 新建实例与选择节点
点击左侧菜单栏的 **“新建实例”**，在地区列表中选择需要的国家（例如日本、美国等）：

![选择实例部署地区](https://github.com/user-attachments/assets/1fd2cc42-c06c-4f62-9197-cea3af093c75)

滑动页面至下方选择母鸡节点与配置规格（例如 400G / 500G 流量套餐）：

![选择母鸡与配置规格](https://github.com/user-attachments/assets/dd62b50a-41e8-4186-a4c2-420d3df41781)

### 4. 系统镜像选择
系统选择 **Debian (Podman)**。密码直接使用系统随机生成的即可（此密码仅用于传统 SSH，本教程使用网页免密控制台，无需死记）：

![选择 Debian 操作系统](https://github.com/user-attachments/assets/2bc73cb0-c22a-422b-89da-91237b58fd5c)

点击确定并创建，等待几十秒直到实例状态显示为正常运行。

---

## 2. 第二部分：NAT 端口转发规则配置（核心关键）

由于是 NAT 架构，必须在商家后台把外网端口与小鸡内网端口打通。我们需要开放 **两个 TCP 规则**：
1. **节点服务端口**：运行 VLESS-REALITY 协议。
2. **订阅服务端口**：运行 Perl 动态流量统计与配置下发。

进入实例详情页，下滑找到 **“端口转发”**，点击 **“+ 添加规则”**：

![端口转发规则列表](https://github.com/user-attachments/assets/c364fe94-c038-4c5d-8936-6629a70d8c75)

请依次添加两条规则（建议内外端口保持一致，例如分配 `23456` 和 `34567`）：
* **规则 1（给节点连接使用）**：
  * **协议**：`TCP`
  * **外部端口**：记下分配的外部端口（例如 `23456`）
  * **内部端口**：小鸡监听端口（内外一致填 `23456`，强制映射则填商家指定的内网端口）
* **规则 2（给订阅下发使用）**：
  * **协议**：`TCP`
  * **外部端口**：记下分配的外部端口（例如 `34567`）
  * **内部端口**：小鸡监听端口（内外一致填 `34567`，强制映射则填商家指定的内网端口）

---

## 3. 第三部分：进入控制台与全通用一键部署

在实例详情页的右侧操作区，点击 **“控制台”** 按钮进入网页终端：

![点击控制台进入Web终端](https://github.com/user-attachments/assets/c1ab8281-ca61-4fc4-99b2-253d83deb611)

### 步骤一：安装核心依赖环境与 sing-box 校验

整段复制并粘贴执行以下命令，脚本会自动适配 amd64 / arm64 架构并进行双通道镜像下载与执行验证：

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

### 步骤二：全通用交互式部署脚本运行

整段复制并粘贴到终端回车。脚本会自动探测物理网卡与公网 IP，引导输入端口（支持内外一致或内外分离），自动生成 UUID、Reality 密钥，并原子化一次性生成所有服务端与客户端配置文件：

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

read -p "4. 客户端显示的卡片名称 [默认: 日本自建（400g）]: " INPUT_NAME
NODE_NAME=${INPUT_NAME:-"日本自建（400g）"}

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

## 4. 第四部分：查看客户端导入链接与 Clash Verge 导入

运行以下命令，打印客户端专属订阅链接：

```bash
source /opt/sing-box/my_env.sh
echo "=========================================================="
echo "客户端专属订阅导入链接："
echo "http://${SERVER_IP}:${EXT_SUB_PORT}/token=${SUB_TOKEN}&name=${NODE_NAME}.yaml"
echo "=========================================================="
```

### Clash Verge 导入方法：
1. 打开 **Clash Verge**，点击左侧 **“订阅 (Profiles)”**。
2. 将打印出来的完整链接粘贴进上方输入框。
3. 点击 **“导入 (Import)”**。卡片将自动以设定的节点名称命名，并精确显示带有两位小数的真实流量进度条。
4. 切换到 **“代理 (Proxies)”** 界面，点击闪电测速验证连通性。

---

## 5. 第五部分：核心防阻断技巧（开着梯子也能秒级更新订阅）

**小白常踩的坑**：很多用户在电脑开着其他梯子/系统代理时，点击订阅卡片的“更新”，会报错提示 `failed to fetch remote profile`。这是因为中间商业代理节点屏蔽了非标高位外部端口。

### 一劳永逸解法（无需每次手动开关梯子）：
1. 在 Clash Verge 中打开平时主力使用的那个订阅卡片，右键点击选择 **“编辑扩展配置 (Edit Rules / Script)”**。
2. 在规则列表（`rules:`）的最顶部添加一行公网 IP 直连规则：
   ```yaml
   rules:
     - IP-CIDR,你的小鸡公网IP/32,DIRECT,no-resolve
   ```
3. 保存并刷新应用。

**生效机制**：  
当你开着系统代理更新该订阅时，流量会自动绕开梯子，直接走本地物理宽带直连出站，彻底避开商业节点防火墙对高位端口的拦截，秒级拉取成功。

---

## 6. 常用维护与救砖命令

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
