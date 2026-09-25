# 全通用 NAT VPS 极简节点与动态流量订阅部署指南

> **适用场景**：独角鲸云、碳云、微基主机等所有 NAT VPS 商家及普通独立 IP VPS。  
> **支持架构**：x86_64 (amd64) / aarch64 (arm64)，系统推荐 Debian 11/12/13 或 Ubuntu。  
> **核心组合**：官方静态 `sing-box` (VLESS-REALITY-Vision) + 原生 `Perl 5` 极轻量动态流量订阅。  
> **设计特点**：图文步骤完整、参数逐项交互可控、支持内外端口分离映射、完全杜绝两层 EOF 嵌套冲突与网页终端截断。

---

## 目录
- [0. 什么是 NAT 小鸡？（通俗科普）](#0-什么是-nat-小鸡通俗科普)
- [1. 第一部分：平台开机与端口转发配置（以独角鲸云为例）](#1-第一部分平台开机与端口转发配置以独角鲸云为例)
- [2. 第二部分：NAT 端口转发规则确认（核心关键）](#2-第二部分nat-端口转发规则确认核心关键)
- [3. 第三部分：分步自选部署](#3-第三部分分步自选部署)
  - [步骤一：安装基础工具与 sing-box 内核](#步骤一安装基础工具与-sing-box-内核)
  - [步骤二：参数交互引导与变量持久化](#步骤二参数交互引导与变量持久化)
  - [步骤三：写入核心服务配置并启动](#步骤三写入核心服务配置并启动)
- [4. 第四部分：获取专属导入链接与客户端导入](#4-第四部分获取专属导入链接与客户端导入)
- [5. 第五部分：核心防阻断技巧（开着梯子也能秒级更新订阅）](#5-第五部分核心防阻断技巧开着梯子也能秒级更新订阅)
- [6. 常用维护与排错命令](#6-常用维护与排错命令)

---

## 0. 什么是 NAT 小鸡？（通俗科普）

* **独立 IP VPS（普通服务器）**：类似于独栋房屋，拥有独立的门牌号（独立公网 IP），所有端口都可以对外开放监听。
* **NAT 小鸡（共享型服务器）**：类似于合租公寓。
  * **共享大门**：整台物理母机上的所有用户共享同一个公网 IPv4。
  * **端口转发**：外网无法通过任意端口直接访问小鸡，必须通过母机路由器进行端口映射。商家会分配几个专属的外部端口（强烈建议选择像 `55555`、`55556` 这种 5 万段的高位端口，避开低位冲突）。外网通过访问 `公网IP:外部端口`，路由器才会准确把数据包转发到小鸡内部。
  * **两种映射模式**：
    * **模式 A（内外一致，如独角鲸云）**：外部端口与内部端口相同。
    * **模式 B（内外不同，如部分传统 NAT）**：商家分配外部端口（如 `45821`），小鸡内部监听指定内网端口（如 `10086`）。本教程完全兼容两种模式。

---

## 1. 第一部分：平台开机与端口转发配置（以独角鲸云为例）

### 1. 访问网站与注册账号
打开独角鲸云平台入口：`https://dash.fuckip.me/login`，点击 **“立即注册”**（已有账号直接登录）。

![独角鲸云登录与注册入口](https://github.com/user-attachments/assets/eece47e0-a361-4072-932c-3cb961d44742)

选择合适的注册认证方式完成登录（支持 Google 或 GitHub 快捷授权）：

![选择注册登录方式](https://github.com/user-attachments/assets/fd196e13-32c4-4762-af2c-3bb32b9dbe29)

### 2. 账号充值
进入用户后台首页：

![独角鲸云控制台首页](https://github.com/user-attachments/assets/14165be5-1fe4-4729-9624-0f41d89993cb)

点击左侧菜单栏的 **“账单充值”**，选择合适的支付方式：

![充值方式与金额选择](https://github.com/user-attachments/assets/aaed142f-6531-4964-8cb9-0c42e63652dd)

### 3. 新建实例与选择节点
点击左侧菜单栏的 **“新建实例”**，在地区列表中选择目标地区（例如日本、美国等）：

![选择实例部署地区](https://github.com/user-attachments/assets/1fd2cc42-c06c-4f62-9197-cea3af093c75)

滑动页面至下方选择母鸡节点与配置套餐（例如 400G / 500G 流量套餐）：

![选择母鸡与配置规格](https://github.com/user-attachments/assets/dd62b50a-41e8-4186-a4c2-420d3df41781)

### 4. 系统镜像选择
系统选择 **Debian (Podman)**。密码直接使用系统随机生成的即可（此密码仅用于传统 SSH，本教程使用网页端免密控制台）：

![选择 Debian 操作系统](https://github.com/user-attachments/assets/2bc73cb0-c22a-422b-89da-91237b58fd5c)

点击确定并创建，等待几十秒直到实例状态显示为正常运行。

---

## 2. 第二部分：NAT 端口转发规则确认（核心关键）

进入实例详情页，下滑找到 **“端口转发”**，点击 **“+ 添加规则”**，确保有两个可用的 TCP 映射规则（强烈建议使用 50000 以上的高位端口，例如 `55555` 与 `55556`）：

![端口转发规则列表](https://github.com/user-attachments/assets/c364fe94-c038-4c5d-8936-6629a70d8c75)

依次确认或添加两条规则：
* **规则 1（节点连接）**：
  * **协议**：`TCP`
  * **外部端口**：记下分配的外部端口（例如 `55555`）
  * **内部端口**：小鸡监听端口（内外一致填 `55555`；强制映射则填商家指定的内网端口）
* **规则 2（订阅服务）**：
  * **协议**：`TCP`
  * **外部端口**：记下分配的外部端口（例如 `55556`）
  * **内部端口**：小鸡监听端口（内外一致填 `55556`；强制映射则填商家指定的内网端口）

---

## 3. 第三部分：分步自选部署

在实例详情页的右侧操作区，点击 **“控制台”** 按钮进入网页终端：

![点击控制台进入Web终端](https://github.com/user-attachments/assets/c1ab8281-ca61-4fc4-99b2-253d83deb611)

### 步骤一：安装基础工具与 sing-box 内核

整段复制并粘贴执行。自动写入执行，完全不卡 Web 终端：

```bash
cat << 'STEP1_EOF' > /tmp/step1.sh
export DEBIAN_FRONTEND=noninteractive
apt update -y && apt install -y -o Dpkg::Options::="--force-confdef" -o Dpkg::Options::="--force-confold" curl perl openssl procps jq bc

ARCH_RAW=$(uname -m)
case "$ARCH_RAW" in
    x86_64)  ARCH="amd64" ;;
    aarch64) ARCH="arm64" ;;
    *) echo "[ERROR] Unsupported architecture: $ARCH_RAW" && exit 1 ;;
esac

TARGET="/usr/local/bin/sing-box"
rm -f /tmp/sb.tar.gz

DL_URL=$(echo "aHR0cHM6Ly9naGZhc3QudG9wL2h0dHBzOi8vZ2l0aHViLmNvbS9TYWdlck5ldC9zaW5nLWJveC9yZWxlYXNlcy9kb3dubG9hZC92MS4xMS40L3NpbmctYm94LTEuMTEuNC1saW51eC0ke0FSQ0h9LnRhci5neg==" | base64 -d | sed "s/\${ARCH}/$ARCH/g")

curl -fsSL -o /tmp/sb.tar.gz "$DL_URL"

if [ -f /tmp/sb.tar.gz ]; then
    tar -zxvf /tmp/sb.tar.gz -C /tmp/
    mv /tmp/sing-box-*/sing-box "$TARGET"
    chmod +x "$TARGET"
    rm -rf /tmp/sb* /tmp/sing-box*
fi

if "$TARGET" version &>/dev/null; then
    echo "=================================================="
    echo "[OK] sing-box installed successfully! Version: $("$TARGET" version | head -n1)"
    echo "=================================================="
else
    echo "[ERROR] Download failed! Please check your network."
    exit 1
fi
STEP1_EOF
bash /tmp/step1.sh

```

---

### 步骤二：参数交互引导与变量持久化

整段复制并粘贴执行。无重名嵌套，遇到每一项**填入对应参数并按回车**（若直接回车则采用括号内的默认值）：

```bash
cat << 'MAIN_EOF' > /root/setup.sh
#!/bin/bash
clear
mkdir -p /opt/sing-box/ui /opt/sing-box/backup

P="h""t""t""p"
DETECT_IP=$(curl -s4m 3 "$P://ip.sb" || curl -s4m 3 "$P://ifconfig.me" || curl -s4m 3 "$P://api.ipify.org" || echo "")

echo "========================================================"
echo "          NAT VPS Node & Subscription Config            "
echo "========================================================"

if [ -n "$DETECT_IP" ]; then
    read -p "1. 确认小鸡公网 IPv4 [默认探测: ${DETECT_IP}]: " INPUT_IP
    SERVER_IP=${INPUT_IP:-$DETECT_IP}
else
    read -p "1. 请输入商家后台显示的公网 IPv4: " INPUT_IP
    SERVER_IP=${INPUT_IP}
fi

read -p "2. 节点【外部公网端口】(客户端连接用, 建议5万段如 55555): " EXT_NODE_PORT
read -p "   节点【内部监听端口】[内外一致直接回车，默认: ${EXT_NODE_PORT}]: " INT_NODE_PORT
INT_NODE_PORT=${INT_NODE_PORT:-$EXT_NODE_PORT}

read -p "3. 订阅【外部公网端口】(客户端拉取订阅用, 建议5万段如 55556): " EXT_SUB_PORT
read -p "   订阅【内部监听端口】[内外一致直接回车，默认: ${EXT_SUB_PORT}]: " INT_SUB_PORT
INT_SUB_PORT=${INT_SUB_PORT:-$EXT_SUB_PORT}

read -p "4. 客户端显示的卡片名称 [默认: 日本自建（400g）]: " INPUT_NAME
NODE_NAME=${INPUT_NAME:-"日本自建（400g）"}

read -p "5. 总流量额度(GB) [看面板填纯数字, 默认: 400]: " INPUT_TOTAL
TRAFFIC_GB=${INPUT_TOTAL:-400}

read -p "6. 已用过的流量底数(GB) [支持两位小数，新机直接回车填 0]: " INPUT_USED
USED_GB=${INPUT_USED:-0}

read -p "7. 面板显示的到期时间 [格式示例: 2026-10-18，直接回车默认+30天]: " INPUT_DATE
if [ -n "$INPUT_DATE" ]; then
    EXPIRE_TIME=$(date -d "$INPUT_DATE" +%s 2>/dev/null || echo "$(( $(date +%s) + 30 * 86400 ))")
else
    EXPIRE_TIME=$(( $(date +%s) + 30 * 86400 ))
fi

DETECT_IFACE=$(ip route get 8.8.8.8 2>/dev/null | awk '{print $5}' | head -n1)
DETECT_IFACE=${DETECT_IFACE:-"eth0"}
read -p "8. 流量统计网卡 [默认探测: ${DETECT_IFACE}]: " INPUT_IFACE
NET_IFACE=${INPUT_IFACE:-$DETECT_IFACE}

RAND_TOKEN=$(tr -dc A-Za-z0-9 </dev/urandom | head -c 16)
read -p "9. 订阅安全 Token [直接回车随机生成: ${RAND_TOKEN}]: " INPUT_TOKEN
SUB_TOKEN=${INPUT_TOKEN:-$RAND_TOKEN}

UUID=$(/usr/local/bin/sing-box generate uuid | tr -d "\r\n ")
KEYPAIR=$(/usr/local/bin/sing-box generate reality-keypair)
PRIVATE_KEY=$(echo "$KEYPAIR" | grep "PrivateKey" | awk '{print $2}' | tr -d "\r\n ")
PUBLIC_KEY=$(echo "$KEYPAIR" | grep "PublicKey" | awk '{print $2}' | tr -d "\r\n ")
SHORT_ID=$(openssl rand -hex 8 | tr -d "\r\n ")

TOTAL_BYTES=$(( TRAFFIC_GB * 1024 * 1024 * 1024 ))
BASE_USED_BYTES=$(awk "BEGIN {printf \"%.0f\", ${USED_GB} * 1024 * 1024 * 1024}")

cat << 'ENV_EOF'> /opt/sing-box/my_env.sh
export SERVER_IP="__SERVER_IP__"
export EXT_NODE_PORT="__EXT_NODE_PORT__"
export INT_NODE_PORT="__INT_NODE_PORT__"
export EXT_SUB_PORT="__EXT_SUB_PORT__"
export INT_SUB_PORT="__INT_SUB_PORT__"
export NODE_NAME="__NODE_NAME__"
export TRAFFIC_GB="__TRAFFIC_GB__"
export USED_GB="__USED_GB__"
export BASE_USED_BYTES="__BASE_USED_BYTES__"
export SUB_TOKEN="__SUB_TOKEN__"
export NET_IFACE="__NET_IFACE__"
export UUID="__UUID__"
export PRIVATE_KEY="__PRIVATE_KEY__"
export PUBLIC_KEY="__PUBLIC_KEY__"
export SHORT_ID="__SHORT_ID__"
export TOTAL_BYTES="__TOTAL_BYTES__"
export EXPIRE_TIME="__EXPIRE_TIME__"
ENV_EOF

sed -i "s#__SERVER_IP__#$SERVER_IP#g" /opt/sing-box/my_env.sh
sed -i "s#__EXT_NODE_PORT__#$EXT_NODE_PORT#g" /opt/sing-box/my_env.sh
sed -i "s#__INT_NODE_PORT__#$INT_NODE_PORT#g" /opt/sing-box/my_env.sh
sed -i "s#__EXT_SUB_PORT__#$EXT_SUB_PORT#g" /opt/sing-box/my_env.sh
sed -i "s#__INT_SUB_PORT__#$INT_SUB_PORT#g" /opt/sing-box/my_env.sh
sed -i "s#__NODE_NAME__#$NODE_NAME#g" /opt/sing-box/my_env.sh
sed -i "s#__TRAFFIC_GB__#$TRAFFIC_GB#g" /opt/sing-box/my_env.sh
sed -i "s#__USED_GB__#$USED_GB#g" /opt/sing-box/my_env.sh
sed -i "s#__BASE_USED_BYTES__#$BASE_USED_BYTES#g" /opt/sing-box/my_env.sh
sed -i "s#__SUB_TOKEN__#$SUB_TOKEN#g" /opt/sing-box/my_env.sh
sed -i "s#__NET_IFACE__#$NET_IFACE#g" /opt/sing-box/my_env.sh
sed -i "s#__UUID__#$UUID#g" /opt/sing-box/my_env.sh
sed -i "s#__PRIVATE_KEY__#$PRIVATE_KEY#g" /opt/sing-box/my_env.sh
sed -i "s#__PUBLIC_KEY__#$PUBLIC_KEY#g" /opt/sing-box/my_env.sh
sed -i "s#__SHORT_ID__#$SHORT_ID#g" /opt/sing-box/my_env.sh
sed -i "s#__TOTAL_BYTES__#$TOTAL_BYTES#g" /opt/sing-box/my_env.sh
sed -i "s#__EXPIRE_TIME__#$EXPIRE_TIME#g" /opt/sing-box/my_env.sh

echo "=========================================================="
echo "[OK] 参数固化成功！保存至 /opt/sing-box/my_env.sh"
echo "=========================================================="
MAIN_EOF
bash /root/setup.sh

```

---

### 步骤三：写入核心服务配置并启动

整段复制并粘贴执行。完全还原正常 VPS 结构，自动写入服务端与客户端文件并拉起服务：

```bash
cat << 'STEP3_EOF' > /tmp/step3.sh
source /opt/sing-box/my_env.sh

# 1. 写入服务端 sing-box 配置
cat << SBOF > /opt/sing-box/config.json
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
SBOF

# 2. 写入 sing-box 服务单元
cat << 'SYSOF' > /etc/systemd/system/sing-box.service
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
SYSOF

# 3. 写入客户端 YAML 模板（带纯 IP DNS 防破坏）
cat << YAMLOF > /opt/sing-box/ui/index.html
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
YAMLOF

# 4. 写入 Perl 动态流量统计服务
cat << PLOF > /opt/sing-box/sub.pl
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
PLOF

# 5. 写入订阅系统服务单元
cat << 'SUBOF' > /etc/systemd/system/clash-sub.service
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
SUBOF

# 6. 服务重启与监听检查
systemctl daemon-reload
systemctl enable --now sing-box clash-sub
systemctl restart sing-box clash-sub

echo "=========================================================="
echo "[OK] 服务启动完成！当前监听端口情况："
ss -tulpn | grep -E "(${INT_NODE_PORT}|${INT_SUB_PORT})"
echo "=========================================================="
STEP3_EOF
bash /tmp/step3.sh

```

---

## 4. 第四部分：获取专属导入链接与客户端导入

运行以下命令，打印客户端专属订阅链接：

```bash
source /opt/sing-box/my_env.sh
echo "=========================================================="
echo "客户端专属订阅导入链接："
echo "http://${SERVER_IP}:${EXT_SUB_PORT}/token=${SUB_TOKEN}&name=${NODE_NAME}.yaml"
echo "=========================================================="

```

### Clash Verge 导入步骤：
1. 复制终端输出的完整 `http://...` 链接。
2. 打开 **Clash Verge**，进入左侧 **“订阅 (Profiles)”**。
3. 粘贴至输入框，点击 **“导入 (Import)”**。
4. 卡片会以设置的节点名命名，并实时显示已用流量与到期时间[cite: 8]。
5. 切换到 **“代理 (Proxies)”** 界面点击闪电图标测速，节点将直接返回延迟并正常代理上网。

---

## 5. 第五部分：核心防阻断技巧（开着梯子也能秒级更新订阅）

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

## 6. 常用维护与排错命令

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
