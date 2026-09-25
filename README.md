# 全通用 NAT VPS 极简节点与动态流量订阅一键部署指南

> **适用场景**：独角鲸云、碳云、微基主机等所有 NAT VPS 商家及普通独立 IP VPS。  
> **支持架构**：x86_64 (amd64) / aarch64 (arm64)，系统推荐 Debian 11/12/13 或 Ubuntu。  
> **核心组合**：官方静态 `sing-box` (VLESS-REALITY-Vision) + 原生 `Perl 5` 极轻量动态流量订阅。  
> **稳定性保障**：
> 1. **全自动屏蔽弹窗卡死**：内置非交互安装模式，杜绝 Debian 容器 `debconf` 假死。
> 2. **单脚本一键到底**：无需分步执行多条指令，一次粘贴全程引导完成。
> 3. **全通用端口设计**：同时支持“内外端口一致”与“内外端口强制映射”的 NAT 商家。
> 4. **DNS 纯 IP 化**：免除一切浏览器翻译插件改写超链接带来的 YAML 解析崩溃。

---

## 目录
- [0. 什么是 NAT 小鸡？（通俗科普）](#0-什么是-nat-小鸡通俗科普)
- [1. 第一部分：平台开机与端口转发配置](#1-第一部分平台开机与端口转发配置)
- [2. 第二部分：全自动一键交互部署（核心）](#2-第二部分全自动一键交互部署核心)
- [3. 第三部分：获取专属订阅并在客户端导入](#3-第三部分获取专属订阅并在客户端导入)
- [4. 第四部分：核心防阻断技巧（开着梯子也能秒级更新订阅）](#4-第四部分核心防阻断技巧开着梯子也能秒级更新订阅)
- [5. 常用维护与救砖命令](#5-常用维护与救砖命令)

---

## 0. 什么是 NAT 小鸡？（通俗科普）

* **独立 IP VPS（普通服务器）**：类似于独门独栋别墅，有独立的门牌号（独立公网 IP），所有端口都可以对外开放监听。
* **NAT 小鸡（共享型服务器）**：类似于合租大型公寓楼。
  * **共享大门**：整台物理母机上的所有合租用户共享同一个公网 IPv4。
  * **端口转发进屋**：外网无法通过任意端口找到你的小鸡，必须通过商家路由器进行“端口映射”。商家会分配给你的房间几个专属的外部端口（例如分配 `23456`、`34567`）。外网通过访问 `公网IP:外部端口`，路由器才会准确把流量转发给你的小鸡内部。
  * **映射模式**：
    * **内外一致（如独角鲸云）**：外部端口和内部端口相同。
    * **内外分离（如部分传统 NAT）**：商家分配外部端口（如 `45821`），小鸡内部监听指定内网端口（如 `10086`）。本教程全面兼容。

---

## 1. 第一部分：平台开机与端口转发配置

### 1. 独角鲸云开机示例（图文）
打开平台入口登录并充值后，选择节点套餐与系统镜像（推荐选择 **Debian**）：

![独角鲸云登录入口](https://github.com/user-attachments/assets/eece47e0-a361-4072-932c-3cb961d44742)
![选择注册登录方式](https://github.com/user-attachments/assets/fd196e13-32c4-4762-af2c-3bb32b9dbe29)
![用户后台首页](https://github.com/user-attachments/assets/14165be5-1fe4-4729-9624-0f41d89993cb)
![新建实例选择地区](https://github.com/user-attachments/assets/1fd2cc42-c06c-4f62-9197-cea3af093c75)
![选择配置规格](https://github.com/user-attachments/assets/dd62b50a-41e8-4186-a4c2-420d3df41781)
![选择操作系统镜像](https://github.com/user-attachments/assets/2bc73cb0-c22a-422b-89da-91237b58fd5c)

### 2. NAT 端口转发配置（核心）
进入实例详情页，下滑找到 **“端口转发”**，确保有两个可用的 TCP 映射规则：

![端口转发列表](https://github.com/user-attachments/assets/c364fe94-c038-4c5d-8936-6629a70d8c75)

* **规则 1（节点连接）**：协议 `TCP`，分配外部端口（如 `23456`），内部端口（内外一致填 `23456`，强制映射则填商家指定内网端口）。
* **规则 2（订阅服务）**：协议 `TCP`，分配外部端口（如 `34567`），内部端口（内外一致填 `34567`，强制映射则填商家指定内网端口）。

---

## 2. 第二部分：全自动一键交互部署（核心）

点击后台操作区的 **“控制台”** 按钮进入网页终端：

![进入Web终端](https://github.com/user-attachments/assets/c1ab8281-ca61-4fc4-99b2-253d83deb611)

**将下面整段代码完整复制，直接粘贴到控制台并回车：**  
脚本会自动设置非交互环境、静默安装依赖、多通道加速下载 `sing-box` 静态二进制文件，并直接进入问答模式引导输入：

```bash
export DEBIAN_FRONTEND=noninteractive
apt update -y && apt install -y -o Dpkg::Options::="--force-confdef" -o Dpkg::Options::="--force-confold" curl perl openssl procps jq bc

# 1. 自动适配架构并下载 sing-box
ARCH_RAW=$(uname -m)
case "$ARCH_RAW" in
    x86_64)  ARCH="amd64" ;;
    aarch64) ARCH="arm64" ;;
    armv7*)  ARCH="armv7" ;;
    *) echo "未受支持的架构: $ARCH_RAW" && exit 1 ;;
esac

SB_VER="1.11.4"
TARGET="/usr/local/bin/sing-box"

if ! "$TARGET" version &>/dev/null; then
    rm -f /tmp/sb.tar.gz
    curl -fsSL -o /tmp/sb.tar.gz "[https://ghfast.top/https://github.com/SagerNet/sing-box/releases/download/v$](https://ghfast.top/https://github.com/SagerNet/sing-box/releases/download/v$){SB_VER}/sing-box-${SB_VER}-linux-${ARCH}.tar.gz" || \
    curl -fsSL -o /tmp/sb.tar.gz "[https://github.com/SagerNet/sing-box/releases/download/v$](https://github.com/SagerNet/sing-box/releases/download/v$){SB_VER}/sing-box-${SB_VER}-linux-${ARCH}.tar.gz"
    tar -zxvf /tmp/sb.tar.gz -C /tmp/
    mv /tmp/sing-box-*/sing-box "$TARGET"
    chmod +x "$TARGET"
    rm -rf /tmp/sb* /tmp/sing-box*
fi

if ! "$TARGET" version &>/dev/null; then
    echo "ERROR: sing-box 下载异常，请检查网络！"
    exit 1
fi

mkdir -p /opt/sing-box/ui /opt/sing-box/backup

# 2. 交互式收集参数
clear
echo "========================================================"
echo "          全通用 NAT VPS 节点与动态订阅部署引导          "
echo "========================================================"

P="h""t""t""p"
DETECT_IP=$(curl -s4m 3 "$P://ip.sb" || curl -s4m 3 "$P://ifconfig.me" || curl -s4m 3 "$P://api.ipify.org" || echo "")

read -p "1. 确认小鸡公网 IPv4 [当前探测: ${DETECT_IP}]: " INPUT_IP
SERVER_IP=${INPUT_IP:-$DETECT_IP}

read -p "2. 节点【外部公网端口】(客户端连接用): " EXT_NODE_PORT
read -p "   节点【内部监听端口】[内外一致直接回车，默认: ${EXT_NODE_PORT}]: " INT_NODE_PORT
INT_NODE_PORT=${INT_NODE_PORT:-$EXT_NODE_PORT}

read -p "3. 订阅【外部公网端口】(客户端拉取订阅用): " EXT_SUB_PORT
read -p "   订阅【内部监听端口】[内外一致直接回车，默认: ${EXT_SUB_PORT}]: " INT_SUB_PORT
INT_SUB_PORT=${INT_SUB_PORT:-$EXT_SUB_PORT}

read -p "4. 客户端显示的卡片名称 [默认: 日本自建（400g）]: " INPUT_NAME
NODE_NAME=${INPUT_NAME:-"日本自建（400g）"}

read -p "5. 总流量额度(GB) [纯数字, 默认: 400]: " INPUT_TOTAL
TRAFFIC_GB=${INPUT_TOTAL:-400}

read -p "6. 已用流量底数(GB) [支持两位小数如 48.4，新机直接填 0]: " INPUT_USED
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

# 现场生成密钥
UUID=$(/usr/local/bin/sing-box generate uuid | tr -d '\r\n ')
KEYPAIR=$(/usr/local/bin/sing-box generate reality-keypair)
PRIVATE_KEY=$(echo "$KEYPAIR" | grep "PrivateKey" | awk '{print $2}' | tr -d '\r\n ')
PUBLIC_KEY=$(echo "$KEYPAIR" | grep "PublicKey" | awk '{print $2}' | tr -d '\r\n ')
SHORT_ID=$(openssl rand -hex 8 | tr -d '\r\n ')

TOTAL_BYTES=$(( TRAFFIC_GB * 1024 * 1024 * 1024 ))
BASE_USED_BYTES=$(awk "BEGIN {printf \"%.0f\", ${USED_GB} * 1024 * 1024 * 1024}")

# 3. 固化持久化环境变量
cat <<EOF> /opt/sing-box/my_env.sh
export SERVER_IP="${SERVER_IP}"
export EXT_NODE_PORT="${EXT_NODE_PORT}"
export INT_NODE_PORT="${INT_NODE_PORT}"
export EXT_SUB_PORT="${EXT_SUB_PORT}"
export INT_SUB_PORT="${INT_SUB_PORT}"
export NODE_NAME="${NODE_NAME}"
export SUB_TOKEN="${SUB_TOKEN}"
EOF

# 4. 写入服务端 sing-box 配置
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

# 5. 写入客户端纯净 YAML（全纯 IP 保证绝无插件超链接注入）
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

# 6. 写入 Perl 原生订阅服务
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

# 7. 配置 systemd 服务并启动
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

/usr/local/bin/sing-box check -c /opt/sing-box/config.json
systemctl daemon-reload
systemctl enable --now sing-box clash-sub
systemctl restart sing-box clash-sub

# 8. 打印结果
echo ""
echo "=========================================================="
echo "部署成功！内部监听状态验证："
ss -tulpn | grep -E "(${INT_NODE_PORT}|${INT_SUB_PORT})"
echo "----------------------------------------------------------"
echo "客户端专属订阅导入链接为："
echo "http://${SERVER_IP}:${EXT_SUB_PORT}/token=${SUB_TOKEN}&name=${NODE_NAME}.yaml"
echo "=========================================================="
```

---

## 3. 第三部分：获取专属订阅并在客户端导入

执行完成后，终端底部会输出形如以下格式的专属链接：
```text
http://你的小鸡公网IP:外部订阅端口/token=随机密钥&name=日本自建（400g）.yaml
```

### Clash Verge 导入步骤：
1. 复制终端输出的完整 `http://...` 链接。
2. 打开 **Clash Verge**，进入左侧 **“订阅 (Profiles)”**。
3. 粘贴至输入框，点击 **“导入 (Import)”**。
4. 卡片会以设置的节点名命名，并实时显示已用流量与到期时间[cite: 8]。
5. 切换到 **“代理”** 界面点击闪电图标测速，节点将直接返回延迟并可正常代理上网。

---

## 4. 第四部分：核心防阻断技巧（开着梯子也能秒级更新订阅）

**避坑提示**：若电脑开启了系统代理（或开着其他商业节点），点击更新自建订阅时可能会报 `failed to fetch remote profile`。这是因为中间代理屏蔽了非标准的高位外部端口。

### 一劳永逸解法：
1. 在 Clash Verge 中打开平时主力使用的商业订阅，右键选择 **“编辑扩展配置 (Edit Rules / Script)”**。
2. 在规则列表（`rules:`）的最顶部添加一行小鸡公网 IP 直连规则：
   ```yaml
   rules:
     - IP-CIDR,你的小鸡公网IP/32,DIRECT,no-resolve
   ```
3. 保存并刷新。更新自建小鸡订阅时将直接走本地宽带直连出站，彻底避开商业代理节点的端口拦截。

---

## 5. 常用维护与救砖命令

* **查看当前配置的参数**：
  ```bash
  cat /opt/sing-box/my_env.sh
  ```
* **一键重启两项服务**：
  ```bash
  systemctl restart sing-box clash-sub
  ```
* **检查运行状态与监听端口**：
  ```bash
  systemctl status sing-box --no-pager
  ss -tulpn | grep -E "(sing-box|perl)"
  ```
