# NAT小鸡极简自建节点与动态流量订阅指南（小白交互问答版）

> **适用场景**：独角鲸云或其他商家的 NAT VPS、独立 IP 小鸡（64MB~512MB 内存）。  
> **核心组合**：官方静态 `sing-box` (VLESS-REALITY) + 原生 `Perl 5` 极轻量动态流量订阅。  
> **设计特点**：终端问答式输入、自动探测 IP 与网卡、自动生成密钥、持久化保存变量、自带防梯子阻断。

---

## 目录
- [0. 什么是 NAT 小鸡？（通俗科普）](#0-什么是-nat-小鸡通俗科普)
- [1. 核心原理解析：为什么开着梯子会更新失败？](#1-核心原理解析为什么开着梯子会更新失败)
- [2. 准备工作：在商家后台看哪两个端口？（以独角鲸云为例）](#2-准备工作在商家后台看哪两个端口以独角鲸云为例)
- [3. 步骤一：安装基础依赖（一次性复制）](#3-步骤一安装基础依赖一次性复制)
- [4. 步骤二：交互式参数配置（终端弹窗自动建变量）](#4-步骤二交互式参数配置终端弹窗自动建变量)
- [5. 步骤三：一键部署核心节点与订阅服务](#5-步骤三一键部署核心节点与订阅服务)
- [6. 步骤四：查看你的专属导入链接与客户端导入](#6-步骤四查看你的专属导入链接与客户端导入)
- [7. 核心防阻断技巧：开着梯子也能秒级更新订阅](#7-核心防阻断技巧开着梯子也能秒级更新订阅)
- [8. 日常维护：随时查看或调用已保存的变量](#8-日常维护随时查看或调用已保存的变量)

---

## 0. 什么是 NAT 小鸡？（通俗科普）

很多新手第一次接触 NAT 小鸡会觉得陌生，用一个简单的比喻就能彻底明白：

* **独立 IP 的 VPS（普通服务器）**：相当于你独自租了一整套单门独户的别墅。大门直接对着外面的马路（公网），门牌号（公网 IP）全属于你一个人，你想开哪个窗户、走哪个门（开放 1~65535 任意端口）都可以直接连通。
* **NAT 小鸡（共享型服务器）**：相当于几百个人**合租在同一栋大型公寓大楼**里。
  * **共享大门**：整栋楼只有一个对外的总门牌号（所有合租人共享同一个公网 IP）。
  * **按房间分发端口**：邮递员送包裹不可能直接送到你房间。商家（房东）在后台通过路由器做“端口映射”，分配给你的小鸡几个固定的外部端口（例如分配给你 `59688` 和 `59689`）。外网必须通过 `总门牌号:指定外网端口`，才能准确转交给你房间内部的程序。
  * **优势与劣势**：**性价比极高、便宜**，但必须在商家后台查看并映射外部端口，不能像普通 VPS 那样随意监听 `443` 或 `80`。

---

## 1. 核心原理解析：为什么开着梯子会更新失败？

在客户端（Clash Verge、Flclash 等）开启了全局代理或系统代理后，拉取自建订阅经常红字报错 `failed to fetch remote profile`：

1. **商业代理/机场防火墙拦截高位端口**：  
   你当前挂着的商业梯子节点，出站防火墙通常只允许访问标准的 `80`（HTTP）或 `443`（HTTPS）。而 NAT 小鸡的订阅端口一般是几万的高位端口（如 `59688`），走的又是纯明文 HTTP，直接被中间节点切断丢弃。
2. **请求路径被中间代理改写**：  
   部分代理软件在转发明文时，会把请求路径从相对路径（`/token=...`）改写为绝对 URI（`http://ip:port/token=...`），服务端的正则如果太严谨就会直接拒连。

### 本教程的破解对策：
* **服务端自适应模糊匹配**：只要请求里带有你的密码 Token，管它中间代理怎么改写一律放行。
* **客户端规则分流（最推荐）**：在客户端规则里把小鸡的 IP 加入直连（`DIRECT`），更新订阅时直接走你家里的宽带出站，彻底避开商业节点的拦截。

---

## 2. 准备工作：在商家后台看哪两个端口？（以独角鲸云为例）

进入独角鲸云控制台 -> 打开你的实例详情 -> 找到 **NAT 端口转发 / 端口映射**：

| 映射用途 | 内部端口 (填给服务器) | 协议 | 外部端口 (商家分配给你的) |
| :--- | :--- | :--- | :--- |
| **节点端口** | `59689` | TCP | 商家分配的外部端口（例如 `59689`） |
| **订阅端口** | `59688` | TCP | 商家分配的外部端口（例如 `59688`） |

> **提示**：记下商家分给你的这两个**外部端口**，等一下终端会弹出来让你输入。

---

## 3. 步骤一：安装基础依赖（一次性复制）

登录 VPS 终端，直接复制执行：

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

## 4. 步骤二：交互式参数配置（终端弹窗自动建变量）

在终端中**直接整段粘贴运行**下面这段命令。  
它会自动抓取本机公网 IP、自动识别网卡、自动生成密码密钥。对于无法自动获取的信息，**终端会逐条停下来让你输入**，回车即确认，并自动存入文件供后续随时调用：

```bash
mkdir -p /opt/sing-box/ui /opt/sing-box/backup

# 1. 自动探测公网 IPv4
DETECT_IP=$(curl -s4m 5 [https://api.ipify.org](https://api.ipify.org) || curl -s4m 5 [https://icanhazip.com](https://icanhazip.com) || echo "")
echo "--------------------------------------------------------"
read -p "1. 确认公网 IP [默认探测为: ${DETECT_IP}]: " INPUT_IP
SERVER_IP=${INPUT_IP:-$DETECT_IP}

# 2. 交互输入节点与订阅端口
read -p "2. 请输入商家分配给节点的外部端口 (VLESS, 如 59689): " NODE_PORT
read -p "3. 请输入商家分配给订阅的外部端口 (HTTP, 如 59688): " SUB_PORT

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

# 6. 自动生成安全密钥对与换算
UUID=$(sing-box generate uuid)
KEYPAIR=$(sing-box generate reality-keypair)
PRIVATE_KEY=$(echo "$KEYPAIR" | grep "PrivateKey" | awk '{print $2}')
PUBLIC_KEY=$(echo "$KEYPAIR" | grep "PublicKey" | awk '{print $2}')
SHORT_ID=$(openssl rand -hex 8)

TOTAL_BYTES=$(( TRAFFIC_GB * 1024 * 1024 * 1024 ))
EXPIRE_TIME=$(( $(date +%s) + EXPIRE_DAYS * 86400 ))

# 7. 持久化存储到环境变量脚本中，方便以后随时一键载入
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

## 5. 步骤三：一键部署核心节点与订阅服务

参数生成后，直接整段复制执行以下代码。它会自动调用刚才生成的全部变量，直接把 sing-box、LoyalSoldier 精细分流 YAML、原生 Perl 动态流量服务配置好并启动：

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

# 4. 写入原生 Perl 动态流量订阅服务端
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

## 6. 步骤四：查看你的专属导入链接与客户端导入

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
1. 打开 **Clash Verge** -> 点击左侧 **“订阅”**。
2. 将打印出来的链接粘贴进上方输入框。
3. 点击 **“导入”**。卡片将自动以你的节点名称命名，并显示带有真实消耗的流量进度条。

---

## 7. 核心防阻断技巧：开着梯子也能秒级更新订阅

如果你电脑上正开着其他商业梯子，点击“更新配置”时出现 `failed to fetch remote profile` 错误：

### 一劳永逸解法（免去频繁开关梯子）：
1. 在 Clash Verge 中打开你平时主力使用的那个订阅卡片，右键点击选择 **“编辑扩展配置 (Edit Rules / Script)”**。
2. 在规则列表（`rules:`）的最顶部添加一行公网 IP 直连规则：
   ```yaml
   rules:
     - IP-CIDR,你的小鸡公网IP/32,DIRECT,no-resolve
   ```
3. 保存并刷新。

**效果**：  
当你开着系统代理更新该订阅时，流量会自动绕开梯子，直接走你本机的宽带直连出站，彻底避开商业节点防火墙对高位端口的拦截，秒级拉取成功。

---

## 8. 日常维护：随时查看或调用已保存的变量

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
