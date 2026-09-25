# NAT小鸡极简自建节点与动态流量订阅指南（以独角鲸云为例）

> **适用场景**：独角鲸云或其他商家的 NAT VPS、独立 IP 小鸡（64MB~512MB 内存）。  
> **核心组合**：官方静态 `sing-box` (VLESS-REALITY-Vision) + 原生 `Perl 5` 极轻量动态流量订阅。  
> **设计特点**：图文对照操作、终端问答式输入、自动探测 IP 与网卡、自动生成密钥、持久化保存变量、彻底解决“开着梯子更新订阅报错 `failed to fetch remote profile`”。

---

## 目录
- [0. 什么是 NAT 小鸡？（通俗科普）](#0-什么是-nat-小鸡通俗科普)
- [1. 第一部分：独角鲸云注册、充值与开机（图文步骤）](#1-第一部分独角鲸云注册充值与开机图文步骤)
- [2. 第二部分：NAT 端口转发规则配置（核心关键）](#2-第二部分nat-端口转发规则配置核心关键)
- [3. 第三部分：进入控制台与交互式部署服务](#3-第三部分进入控制台与交互式部署服务)
  - [步骤一：安装核心依赖环境](#步骤一安装核心依赖环境)
  - [步骤二：参数交互输入与变量持久化创建](#步骤二参数交互输入与变量持久化创建)
  - [步骤三：一键部署核心节点与订阅服务](#步骤三一键部署核心节点与订阅服务)
- [4. 第四部分：查看客户端导入链接与 Clash Verge 导入](#4-第四部分查看客户端导入链接与-clash-verge-导入)
- [5. 第五部分：核心防阻断技巧（开着梯子也能秒级更新订阅）](#5-第五部分核心防阻断技巧开着梯子也能秒级更新订阅)
- [6. 常用维护与一键救砖还原](#6-常用维护与一键救砖还原)

---

## 0. 什么是 NAT 小鸡？（通俗科普）

* **独立 IP VPS（普通服务器）**：类似于独门独栋别墅，有独立的门牌号（独立公网 IP），所有端口都可以对外开放监听。
* **NAT 小鸡（共享型服务器）**：类似于合租大型公寓楼。
  * **共享大门**：整台物理母机上的所有合租用户共享同一个公网 IPv4。
  * **端口转发进屋**：外网无法通过任意端口找到你的小鸡，必须通过商家路由器进行“端口映射”。商家会分配给你的房间几个专属的外部端口（例如分配 `59688`、`59689`）。外网通过访问 `公网IP:外部端口`，路由器才会准确把流量转发给你的小鸡内部。
  * **优势**：价格极其亲民、配置精巧、性价比高。

---

## 1. 第一部分：独角鲸云注册、充值与开机（图文步骤）

### 1. 访问网站与注册账号
打开独角鲸云平台入口：`https://dash.fuckip.me/login`，点击 **“立即注册”**（已有账号可直接登录）。

![独角鲸云登录与注册入口](images/01-login-register.png)

选择合适的注册认证方式完成登录（支持 Google 或 GitHub 快捷授权）：

![选择注册登录方式](images/02-register-methods.png)

### 2. 账号充值
进入用户后台首页：

![独角鲸云控制台首页](images/03-dashboard-home.png)

点击左侧菜单栏的 **“账单充值”**，选择合适的支付方式（支持信用卡、支付宝、微信或加密货币）：

![充值方式与金额选择](images/04-recharge-methods.png)

### 3. 新建实例与选择节点
点击左侧菜单栏的 **“新建实例”**，在地区列表中选择你需要的国家（例如日本、美国等）：

![选择实例部署地区](images/05-create-region.png)

滑动页面至下方选择母鸡节点与配置套餐（例如 400G / 500G 流量套餐）：

![选择母鸡与配置规格](images/06-select-node.png)

### 4. 系统镜像选择
系统选择 **Debian (Podman)**。密码直接使用系统随机生成的即可（此密码仅用于传统 SSH，本教程使用网页端免密控制台，无需死记）：

![选择 Debian 操作系统](images/07-os-debian.png)

点击确定并创建，等待几十秒直到实例状态显示为正常运行。

---

## 2. 第二部分：NAT 端口转发规则配置（核心关键）

由于是 NAT 架构，我们必须在商家后台把外网端口与小鸡内网端口打通。我们需要开放 **两个 TCP 规则**：
1. **节点服务端口**：运行 VLESS-REALITY 协议。
2. **订阅服务端口**：运行 Perl 动态流量统计与配置下发。

进入实例详情页，下滑找到 **“端口转发”**，点击 **“+ 添加规则”**：

![端口转发规则列表](images/08-port-forward.png)

请依次添加两条规则：
* **规则 1（给节点连接使用）**：
  * **协议**：`TCP`
  * **内部端口**：`59689`
  * **外部端口**：填入系统分配或自选的外部端口（例如 `59689`）
* **规则 2（给订阅下发使用）**：
  * **协议**：`TCP`
  * **内部端口**：`59688`
  * **外部端口**：填入系统分配或自选的外部端口（例如 `59688`）

> **记下这两个外部端口**：在后续的终端交互中会要求输入。

---

## 3. 第三部分：进入控制台与交互式部署服务

在实例详情页的右侧操作区，点击 **“控制台”** 按钮进入网页终端：

![点击控制台进入Web终端](images/09-web-console.png)

进入黑色终端窗口后，按以下步骤依次执行命令：

### 步骤一：安装核心依赖环境
直接复制并粘贴执行以下命令，安装依赖工具及官方静态 `sing-box`：

```bash
apt update && apt install -y curl perl openssl

if ! command -v sing-box &> /dev/null; then
    echo "正在自动下载官方静态 sing-box..."
    ARCH=$(uname -m)
    [ "$ARCH" = "x86_64" ] && SB_ARCH="amd64" || SB_ARCH="arm64"
    curl -Lo /usr/local/bin/sing-box [https://github.com/SagerNet/sing-box/releases/download/v1.11.4/sing-box-1.11.4-linux-$](https://github.com/SagerNet/sing-box/releases/download/v1.11.4/sing-box-1.11.4-linux-$){SB_ARCH}.tar.gz
    tar -zxvf /usr/local/bin/sing-box -C /tmp/
    mv /tmp/sing-box-*/sing-box /usr/local/bin/sing-box
    chmod +x /usr/local/bin/sing-box
    rm -rf /tmp/sing-box*
fi

sing-box version
perl -v | head -n 2
```

---

### 步骤二：参数交互输入与变量持久化创建

整段复制并粘贴到控制台回车。脚本会自动探测公网 IP 与真实网卡名、自动生成 UUID、Reality 密钥对与防扫 Token。遇到需要确认的项，**终端会逐条停下来让你输入**（若直接按回车则采用默认值）：

```bash
mkdir -p /opt/sing-box/ui /opt/sing-box/backup

# 1. 自动探测公网 IPv4
DETECT_IP=$(curl -s4m 5 [https://api.ipify.org](https://api.ipify.org) || curl -s4m 5 [https://icanhazip.com](https://icanhazip.com) || echo "")
echo "--------------------------------------------------------"
read -p "1. 确认公网 IP [默认探测为: ${DETECT_IP}]: " INPUT_IP
SERVER_IP=${INPUT_IP:-$DETECT_IP}

# 2. 交互输入节点与订阅端口
read -p "2. 请输入独角鲸云分配给节点的外部端口 (VLESS, 如 59689): " NODE_PORT
read -p "3. 请输入独角鲸云分配给订阅的外部端口 (HTTP, 如 59688): " SUB_PORT

# 3. 交互输入卡片名称与额度
read -p "4. 请输入客户端卡片与节点名称 [默认: 日本自建（400g）]: " INPUT_NAME
NODE_NAME=${INPUT_NAME:-"日本自建（400g）"}

read -p "5. 请输入总流量额度 (单位GB，纯数字) [默认: 400]: " INPUT_GB
TRAFFIC_GB=${INPUT_GB:-400}

read -p "6. 请输入到期天数 (多少天后过期) [默认: 30]: " INPUT_DAYS
EXPIRE_DAYS=${INPUT_DAYS:-30}

# 4. 自动探测真实物理网卡 (避开 lo)
DETECT_IFACE=$(ip route get 8.8.8.8 2>/dev/null | awk '{print $5}' | head -n1)
DETECT_IFACE=${DETECT_IFACE:-"eth0"}
read -p "7. 确认流量统计网卡名 [默认探测为: ${DETECT_IFACE}]: " INPUT_IFACE
NET_IFACE=${INPUT_IFACE:-$DETECT_IFACE}

# 5. 自动生成防扫描随机 Token
RAND_TOKEN=$(tr -dc A-Za-z0-9 </dev/urandom | head -c 16)
read -p "8. 确认订阅安全 Token [默认随机生成: ${RAND_TOKEN}]: " INPUT_TOKEN
SUB_TOKEN=${INPUT_TOKEN:-$RAND_TOKEN}

# 6. 自动生成安全密钥对与字节换算
UUID=$(sing-box generate uuid)
KEYPAIR=$(sing-box generate reality-keypair)
PRIVATE_KEY=$(echo "$KEYPAIR" | grep "PrivateKey" | awk '{print $2}')
PUBLIC_KEY=$(echo "$KEYPAIR" | grep "PublicKey" | awk '{print $2}')
SHORT_ID=$(openssl rand -hex 8)

TOTAL_BYTES=$(( TRAFFIC_GB * 1024 * 1024 * 1024 ))
EXPIRE_TIME=$(( $(date +%s) + EXPIRE_DAYS * 86400 ))

# 7. 持久化存储到 /opt/sing-box/my_env.sh 方便后续调用
cat <<EOF> /opt/sing-box/my_env.sh
export SERVER_IP="${SERVER_IP}"
export NODE_PORT="${NODE_PORT}"
export SUB_PORT="${SUB_PORT}"
export NODE_NAME="${NODE_NAME}"
export TRAFFIC_GB="${TRAFFIC_GB}"
export SUB_TOKEN="${SUB_TOKEN}"
export NET_IFACE="${NET_IFACE}"
export UUID="${UUID}"
export PRIVATE_KEY="${PRIVATE_KEY}"
export PUBLIC_KEY="${PUBLIC_KEY}"
export SHORT_ID="${SHORT_ID}"
export TOTAL_BYTES="${TOTAL_BYTES}"
export EXPIRE_TIME="${EXPIRE_TIME}"
EOF

source /opt/sing-box/my_env.sh

echo "--------------------------------------------------------"
echo "[+] 变量自动创建成功，已持久化保存到 /opt/sing-box/my_env.sh！"
```

---

### 步骤三：一键部署核心节点与订阅服务

整段复制并粘贴执行。代码会自动引用前面生成的变量，配置 sing-box、LoyalSoldier 规则集 YAML，并拉起 Perl 原生动态流量订阅服务：

```bash
source /opt/sing-box/my_env.sh

# 1. 写入 sing-box 节点配置
cat <<EOF> /opt/sing-box/config.json
{
  "log": { "level": "warn" },
  "inbounds": [
    {
      "type": "vless",
      "tag": "vless-in",
      "listen": "::",
      "listen_port": ${NODE_PORT},
      "users": [{ "uuid": "${UUID}", "flow": "xtls-rprx-vision" }],
      "tls": {
        "enabled": true,
        "server_name": "gateway.icloud.com",
        "reality": {
          "enabled": true,
          "handshake": { "server": "gateway.icloud.com", "server_port": 443 },
          "private_key": "${PRIVATE_KEY}",
          "short_id": ["${SHORT_ID}"]
        }
      }
    }
  ],
  "outbounds": [{ "type": "direct", "tag": "direct" }]
}
EOF

# 2. 写入 sing-box 服务单元并启动
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

systemctl daemon-reload
systemctl enable --now sing-box
systemctl restart sing-box

# 3. 写入 LoyalSoldier 精细分流客户端 YAML
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
  reject: { type: http, behavior: domain, url: "[https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/reject.txt](https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/reject.txt)", path: ./ruleset/reject.yaml, interval: 86400 }
  icloud: { type: http, behavior: domain, url: "[https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/icloud.txt](https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/icloud.txt)", path: ./ruleset/icloud.yaml, interval: 86400 }
  apple: { type: http, behavior: domain, url: "[https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/apple.txt](https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/apple.txt)", path: ./ruleset/apple.yaml, interval: 86400 }
  google: { type: http, behavior: domain, url: "[https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/google.txt](https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/google.txt)", path: ./ruleset/google.yaml, interval: 86400 }
  proxy: { type: http, behavior: domain, url: "[https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/proxy.txt](https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/proxy.txt)", path: ./ruleset/proxy.yaml, interval: 86400 }
  direct: { type: http, behavior: domain, url: "[https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/direct.txt](https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/direct.txt)", path: ./ruleset/direct.yaml, interval: 86400 }
  gfw: { type: http, behavior: domain, url: "[https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/gfw.txt](https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/gfw.txt)", path: ./ruleset/gfw.yaml, interval: 86400 }
  tld-not-cn: { type: http, behavior: domain, url: "[https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/tld-not-cn.txt](https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/tld-not-cn.txt)", path: ./ruleset/tld-not-cn.yaml, interval: 86400 }
  telegramcidr: { type: http, behavior: ipcidr, url: "[https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/telegramcidr.txt](https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/telegramcidr.txt)", path: ./ruleset/telegramcidr.yaml, interval: 86400 }
  cncidr: { type: http, behavior: ipcidr, url: "[https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/cncidr.txt](https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/cncidr.txt)", path: ./ruleset/cncidr.yaml, interval: 86400 }
  lancidr: { type: http, behavior: ipcidr, url: "[https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/lancidr.txt](https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/lancidr.txt)", path: ./ruleset/lancidr.yaml, interval: 86400 }

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

# 4. 写入原生 Perl 动态流量订阅服务端 (自动读取 /proc/net/dev)
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

# 5. 写入订阅服务并启动
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

systemctl daemon-reload
systemctl enable --now clash-sub
systemctl restart clash-sub
```

---

## 4. 第四部分：查看客户端导入链接与 Clash Verge 导入

运行以下命令，系统会自动打印出验证状态和专属链接：

```bash
source /opt/sing-box/my_env.sh

# 检查端口监听
ss -tulpn | grep -E "(${NODE_PORT}|${SUB_PORT})"

echo ""
echo "=========================================================="
echo "小鸡配置完成！你的客户端专属导入链接为："
echo "http://${SERVER_IP}:${SUB_PORT}/token=${SUB_TOKEN}&name=${NODE_NAME}.yaml"
echo "=========================================================="
```

### 客户端导入方法：
1. 打开 **Clash Verge**，点击左侧 **“订阅 (Profiles)”**。
2. 将打印出来的链接粘贴进上方输入框。
3. 点击 **“导入 (Import)”**。卡片将自动以设定的节点名称命名，并显示带有真实消耗的流量进度条。

---

## 5. 第五部分：核心防阻断技巧（开着梯子也能秒级更新订阅）

**小白常踩的坑**：很多用户在电脑开着其他梯子/系统代理时，点击订阅卡片的“更新”，会报错提示 `failed to fetch remote profile`。这是因为中间代理屏蔽了非标高位端口（如 `59688`）。

### 一劳永逸解法（无需每次手动开关梯子）：
1. 在 Clash Verge 中打开你平时主力使用的那个订阅卡片，右键点击选择 **“编辑扩展配置 (Edit Rules / Script)”**。
2. 在规则列表（`rules:`）的最顶部添加一行公网 IP 直连规则：
   ```yaml
   rules:
     - IP-CIDR,你的小鸡公网IP/32,DIRECT,no-resolve
   ```
3. 保存并刷新应用。

**生效机制**：  
当你开着系统代理更新该订阅时，流量会自动绕开梯子，直接走本地物理宽带直连出站，彻底避开商业节点防火墙对高位端口的拦截，秒级拉取成功。

---

## 6. 常用维护与一键救砖还原

本脚本将你的参数永久保存在了 `/opt/sing-box/my_env.sh` 中。

* **查看你之前设置的全部参数**：
  ```bash
  cat /opt/sing-box/my_env.sh
  ```
* **一键重启所有服务**：
  ```bash
  systemctl restart sing-box clash-sub
  ```
* **一键无损救砖备份与还原**：
  ```bash
  # 备份当前状态
  cp -a /opt/sing-box/config.json /opt/sing-box/backup/
  cp -a /opt/sing-box/sub.pl /opt/sing-box/backup/
  cp -a /opt/sing-box/ui/index.html /opt/sing-box/backup/

  # 误修改后一键还原
  cp /opt/sing-box/backup/* /opt/sing-box/
  systemctl restart sing-box clash-sub
  ```
