# 全通用 NAT VPS 极简节点与动态流量订阅部署指南

> 本项目用于在 NAT VPS 环境中快速部署个人节点服务。  
> 本教程以 **独角鲸云 NAT VPS** 为例，同时兼容碳云、微基主机及各类共享 IPv4 的 Podman / Docker 容器与 Debian 系统 VPS。

**核心组件：**
* `sing-box` 官方静态内核
* `VLESS-REALITY-Vision` 核心协议
* 原生 Perl 5 多协议动态流量订阅服务
* `Clash Meta (Mihomo)` 工业级全分流规则集

**项目目标：**
```
购买 NAT VPS
     ↓
确认两条端口映射
     ↓
执行一条命令自动安装
     ↓
参数交互与规则固化
     ↓
多协议动态订阅开箱即用
     ↓
客户端直接使用
```

---

# 一、NAT VPS 简介与自动化设计

## 1.1 什么是 NAT VPS

**普通 VPS：**  
拥有独立公网 IPv4 地址。

```
客户端 -----> 公网 IP:443 -----> 服务器服务端口
```

**NAT VPS：**  
多个用户共享同一个公网 IPv4，外部访问通过宿主机的 NAT 网关进行端口映射。

```
Clash 客户端
     │
     ▼
公网 IP:59688 (公网映射端口)
     │
     ▼
NAT 网关 (母鸡)
     │
     ▼
VPS:30000 (内部监听端口)
     │
     ▼
sing-box 核心进程
```

例如独角鲸云：
* 外部公网端口：`59688`（客户端连接填此端口）
* 内部监听端口：`30000`（sing-box 实际监听端口）
* NAT 自动转发：`59688` -> `30000`

---

## 1.2 NAT VPS 部署特点

* **优点**：成本极低、资源开销小、适合个人长期运行。
* **核心注意点**：**公网映射端口 ≠ VPS 内部监听端口**。部署时必须严格将两者解耦绑定。

---

## 1.3 本项目自动化设计

传统手动部署容易在端口、UUID、密钥及分流规则配置上出错。本项目设计为：

```
启动脚本
   ↓
自动检测 Debian 版本与 CPU 架构
   ↓
自动探测公网 IPv4
   ↓
自动生成标准 UUID 与 Reality 密钥对
   ↓
用户仅需确认外部与内部映射端口
   ↓
自动生成服务端配置与 Clash 规则
   ↓
拉起后台服务并提供开箱即用订阅
```

---

## 1.4 环境要求

* **推荐系统**：Debian 12 / Debian (Podman)
* **支持系统**：Debian 11+
* **硬件架构**：`x86_64 (amd64)` / `aarch64 (arm64)`

---

# 二、独角鲸云 NAT VPS 准备与端口映射

## 2.1 登录注册入口
访问商家后台，完成登录或注册。

![登录注册入口](https://github.com/user-attachments/assets/eece47e0-a361-4072-932c-3cb961d44742)

---

## 2.2 认证方式
按提示完成账户认证。

![认证方式](https://github.com/user-attachments/assets/fd196e13-32c4-4762-af2c-3bb32b9dbe29)

---

## 2.3 控制台首页
登录后进入控制台，在此管理已有实例、账单与网络映射。

![后台首页](https://github.com/user-attachments/assets/14165be5-1fe4-4729-9624-0f41d89993cb)

---

## 2.4 账单充值
保证账户有足够的余额维持实例正常运行。

![账单充值](https://github.com/user-attachments/assets/aaed142f-6531-4964-8cb9-0c42e63652dd)

---

## 2.5 新建实例与机房选区
点击「新建实例」，选择靠近你的机房区域（如日本、香港、美国等）。

![新建实例与选区](https://github.com/user-attachments/assets/1fd2cc42-c06c-4f62-9197-cea3af093c75)

---

## 2.6 套餐与系统选择
根据需求选择配置，系统推荐选择 **Debian (Podman)** 或 Debian 12。

![选配置套餐](https://github.com/user-attachments/assets/dd62b50a-41e8-4186-a4c2-420d3df41781)

![镜像选择系统](https://github.com/user-attachments/assets/2bc73cb0-c22a-422b-89da-91237b58fd5c)

---

## 2.7 端口转发规划 (核心)

NAT VPS 必须配置两条 **TCP** 端口转发规则。由于 10000~30000 号段容易受到骨干网干扰，**强烈建议在后台申请 50000 以上的高位端口**：

1. **节点服务端口**：
   * 外部公网端口：例如 `59688`（客户端连接使用）
   * 内部服务端口：例如 `30000`（sing-box 监听端口）
2. **订阅服务端口**：
   * 外部公网端口：例如 `59689`（客户端拉取订阅使用）
   * 内部服务端口：例如 `8080`（Perl 订阅服务监听端口）

![端口转发规则](https://github.com/user-attachments/assets/c364fe94-c038-4c5d-8936-6629a70d8c75)

---

## 2.8 登录 Web 控制台
实例开通后，点击「控制台」调出网页终端。所有后续指令均已进行分段隔离，防止因网络波动出现粘贴截断。

![Web 控制台入口](https://github.com/user-attachments/assets/c1ab8281-ca61-4fc4-99b2-253d83deb611)

---

# 三、步骤一：环境依赖与 sing-box 内核安装

整段复制并粘贴到终端执行。脚本自动识别系统架构，下载官方 sing-box 静态二进制包（URL 采用 Base64 防浏览器翻译插件注入）：

```bash
cat << 'STEP1_EOF' > /tmp/step1.sh
#!/bin/bash
set -e
export DEBIAN_FRONTEND=noninteractive

echo "=================================================="
echo "          [1/5] 安装基础依赖与内核环境            "
echo "=================================================="

# 1. 安装基础工具链
apt update -y && apt install -y -o Dpkg::Options::="--force-confdef" -o Dpkg::Options::="--force-confold" \
    curl wget tar perl ca-certificates openssl jq procps bc

# 2. 架构识别
ARCH_RAW=$(uname -m)
case "$ARCH_RAW" in
    x86_64)  ARCH="amd64" ;;
    aarch64|arm64) ARCH="arm64" ;;
    *) echo "[!] 不支持的硬件架构: $ARCH_RAW" && exit 1 ;;
esac
echo "[*] 系统硬件架构: $ARCH"

# 3. 目录初始化
mkdir -p /opt/sing-box/config /opt/sing-box/ui /opt/sing-box/env /tmp/sb_dl
TARGET="/opt/sing-box/sing-box"
rm -f /tmp/sb_dl/sing-box.tar.gz

# 4. Base64 编码防篡改下载官方 sing-box 内核
B64_URL=$(echo "aHR0cHM6Ly9naGZhc3QudG9wL2h0dHBzOi8vZ2l0aHViLmNvbS9TYWdlck5ldC9zaW5nLWJveC9yZWxlYXNlcy9kb3dubG9hZC92MS4xMS40L3NpbmctYm94LTEuMTEuNC1saW51eC0ke0FSQ0h9LnRhci5neg==" | base64 -d | sed "s/\${ARCH}/$ARCH/g")

echo "[*] 正在拉取官方 sing-box 静态核心..."
curl -fsSL -o /tmp/sb_dl/sing-box.tar.gz "$B64_URL"

tar -zxvf /tmp/sb_dl/sing-box.tar.gz -C /tmp/sb_dl/
mv $(find /tmp/sb_dl -name sing-box -type f | head -1) "$TARGET"
chmod +x "$TARGET"
ln -sf "$TARGET" /usr/local/bin/sing-box
rm -rf /tmp/sb_dl

if "$TARGET" version &>/dev/null; then
    echo "=================================================="
    echo "[+] sing-box 安装成功！版本: $("$TARGET" version | head -n1)"
    echo "=================================================="
else
    echo "[!] 内核验证失败，请排查网络环境。"
    exit 1
fi
STEP1_EOF
bash /tmp/step1.sh
```

---

# 四、步骤二：参数交互引导与配置固化

整段复制并粘贴执行。自动引导参数填写，分离收集内外端口，生成服务端 `config.json`（带独立 DNS 解析，杜绝容器超时），并写入包含全套 17 组 Loyalsoldier 纯文本格式规则以及 Fake-IP 过滤保护的客户端 YAML 工业模板：

```bash
cat << 'STEP2_EOF' > /tmp/step2.sh
#!/bin/bash
set -e
CONF_DIR="/opt/sing-box"
ENV_FILE="$CONF_DIR/env/node.env"
mkdir -p "$CONF_DIR/config" "$CONF_DIR/ui" "$CONF_DIR/env"

P="h""t""t""p"
DETECT_IP=$(curl -s4m 3 "$P://ip.sb" || curl -s4m 3 "$P://ifconfig.me" || curl -s4m 3 "$P://api.ipify.org" || echo "")

echo "=================================================="
echo "          [2/5] NAT VPS 参数交互与配置生成        "
echo "=================================================="

if [ -n "$DETECT_IP" ]; then
    read -p "1. 确认小鸡公网 IPv4 [默认探测: ${DETECT_IP}]: " INPUT_IP
    SERVER_IP=${INPUT_IP:-$DETECT_IP}
else
    read -p "1. 请输入商家后台显示的公网 IPv4: " INPUT_IP
    SERVER_IP=${INPUT_IP}
fi

read -p "2. 节点【公网外部映射端口】(例如 59688): " EXT_NODE_PORT
read -p "   节点【小鸡内部监听端口】[默认 30000，内外一致直接填外部端口]: " INPUT_INT_NODE_PORT
INT_NODE_PORT=${INPUT_INT_NODE_PORT:-30000}

read -p "3. 订阅【公网外部映射端口】(例如 59689): " EXT_SUB_PORT
read -p "   订阅【小鸡内部监听端口】[默认 8080，内外一致直接填外部端口]: " INPUT_INT_SUB_PORT
INT_SUB_PORT=${INPUT_INT_SUB_PORT:-8080}

read -p "4. 客户端显示的卡片名称 [默认: 日本自建（400g）]: " INPUT_NAME
NODE_NAME=${INPUT_NAME:-"日本自建（400g）"}

read -p "5. 总流量额度(GB) [纯数字, 默认: 400]: " INPUT_TOTAL
TRAFFIC_GB=${INPUT_TOTAL:-400}

read -p "6. 已用流量底数(GB) [支持两位小数，新机直接回车填 0.00]: " INPUT_USED
USED_GB=${INPUT_USED:-0.00}

read -p "7. 面板到期时间 [格式 YYYY-MM-DD，回车默认当前+30天]: " INPUT_DATE
if [ -z "$INPUT_DATE" ]; then
    EXPIRE_DATE=$(date -d "+30 days" +%Y-%m-%d 2>/dev/null || date -v+30d +%Y-%m-%d)
else
    EXPIRE_DATE="$INPUT_DATE"
fi

RAND_TOKEN=$(tr -dc A-Za-z0-9 </dev/urandom | head -c 16)
read -p "8. 订阅安全 Token [直接回车随机生成: ${RAND_TOKEN}]: " INPUT_TOKEN
SUB_TOKEN=${INPUT_TOKEN:-$RAND_TOKEN}

# 密钥与标识生成
UUID=$(/opt/sing-box/sing-box generate uuid | tr -d "\r\n ")
KEYPAIR=$(/opt/sing-box/sing-box generate reality-keypair)
PRIVATE_KEY=$(echo "$KEYPAIR" | grep "PrivateKey" | awk '{print $2}' | tr -d "\r\n ")
PUBLIC_KEY=$(echo "$KEYPAIR" | grep "PublicKey" | awk '{print $2}' | tr -d "\r\n ")
SHORT_ID=$(openssl rand -hex 8 | tr -d "\r\n ")

# 持久化环境变量
cat << ENV_EOF> "$ENV_FILE"
SERVER_IP="${SERVER_IP}"
EXT_NODE_PORT="${EXT_NODE_PORT}"
INT_NODE_PORT="${INT_NODE_PORT}"
EXT_SUB_PORT="${EXT_SUB_PORT}"
INT_SUB_PORT="${INT_SUB_PORT}"
NODE_NAME="${NODE_NAME}"
TRAFFIC_GB="${TRAFFIC_GB}"
USED_GB="${USED_GB}"
EXPIRE_DATE="${EXPIRE_DATE}"
SUB_TOKEN="${SUB_TOKEN}"
UUID="${UUID}"
PRIVATE_KEY="${PRIVATE_KEY}"
PUBLIC_KEY="${PUBLIC_KEY}"
SHORT_ID="${SHORT_ID}"
ENV_EOF

chmod 600 "$ENV_FILE"

# 1. 写入服务端 sing-box 配置（加入独立 DNS，规避 Podman 容器 resolv.conf 阻断）
cat << SBOF > /opt/sing-box/config/config.json
{
  "log": {
    "level": "warn"
  },
  "dns": {
    "servers": [
      {
        "tag": "dns-direct",
        "address": "223.5.5.5",
        "detour": "direct"
      }
    ]
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

/opt/sing-box/sing-box check -c /opt/sing-box/config/config.json && echo "[+] 服务端 sing-box 配置格式校验通过"

# 2. 写入工业级 Loyalsoldier 全分流客户端 template.yaml
cat << YAMLOF > /opt/sing-box/ui/template.yaml
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
  fake-ip-filter:
    - "*.icloud.com"
    - "gateway.icloud.com"
    - "*.apple.com"
    - "*.msftconnecttest.com"
    - "*.msftncsi.com"
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
    port: ${EXT_NODE_PORT}
    uuid: ${UUID}
    network: tcp
    tls: true
    udp: true
    xudp: true
    packet-addr: true
    packet-encoding: packetaddr
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

rule-providers:
  reject:
    type: http
    behavior: domain
    format: text
    url: "[https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/reject.txt](https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/reject.txt)"
    path: ./ruleset/reject.yaml
    interval: 86400
  icloud:
    type: http
    behavior: domain
    format: text
    url: "[https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/icloud.txt](https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/icloud.txt)"
    path: ./ruleset/icloud.yaml
    interval: 86400
  apple:
    type: http
    behavior: domain
    format: text
    url: "[https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/apple.txt](https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/apple.txt)"
    path: ./ruleset/apple.yaml
    interval: 86400
  google:
    type: http
    behavior: domain
    format: text
    url: "[https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/google.txt](https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/google.txt)"
    path: ./ruleset/google.yaml
    interval: 86400
  proxy:
    type: http
    behavior: domain
    format: text
    url: "[https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/proxy.txt](https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/proxy.txt)"
    path: ./ruleset/proxy.yaml
    interval: 86400
  direct:
    type: http
    behavior: domain
    format: text
    url: "[https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/direct.txt](https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/direct.txt)"
    path: ./ruleset/direct.yaml
    interval: 86400
  gfw:
    type: http
    behavior: domain
    format: text
    url: "[https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/gfw.txt](https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/gfw.txt)"
    path: ./ruleset/gfw.yaml
    interval: 86400
  tld-not-cn:
    type: http
    behavior: domain
    format: text
    url: "[https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/tld-not-cn.txt](https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/tld-not-cn.txt)"
    path: ./ruleset/tld-not-cn.yaml
    interval: 86400
  telegramcidr:
    type: http
    behavior: ipcidr
    format: text
    url: "[https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/telegramcidr.txt](https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/telegramcidr.txt)"
    path: ./ruleset/telegramcidr.yaml
    interval: 86400
  cncidr:
    type: http
    behavior: ipcidr
    format: text
    url: "[https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/cncidr.txt](https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/cncidr.txt)"
    path: ./ruleset/cncidr.yaml
    interval: 86400
  lancidr:
    type: http
    behavior: ipcidr
    format: text
    url: "[https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/lancidr.txt](https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/lancidr.txt)"
    path: ./ruleset/lancidr.yaml
    interval: 86400

rules:
  - IP-CIDR,${SERVER_IP}/32,DIRECT,no-resolve
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
YAMLOF

echo "=================================================="
echo "[+] 步骤二完成：参数已成功保存至 $ENV_FILE"
echo "=================================================="
STEP2_EOF
bash /tmp/step2.sh
```

---

# 五、步骤三：原生轻量多协议动态订阅服务实现 (Perl 5)

整段复制并粘贴执行。使用原生 Perl 5 套接字与 `IO::Select` 构建非阻塞、带超时回收的订阅分发服务：
* 路径 `/`：下发完整 **Clash 工业级分流配置**
* 路径 `/vless`：下发 **单节点 VLESS 链接**
* 路径 `/base64`：下发 **Base64 订阅**（支持 Shadowrocket / v2rayN）
* 自动统计网卡物理流量，并实时注入 `Subscription-Userinfo` 头部

```bash
cat << 'STEP3_EOF' > /tmp/step3.sh
#!/bin/bash
set -e

cat << 'PERL_EOF' > /opt/sing-box/sub.pl
#!/usr/bin/perl
use strict;
use warnings;
use IO::Socket::INET;
use IO::Select;

# 1. 动态读取参数
my %conf;
sub reload_conf {
    my $file = '/opt/sing-box/env/node.env';
    return unless -f $file;
    open(my $fh, '<', $file) or return;
    while (my $line = <$fh>) {
        chomp $line;
        if ($line =~ /^(\w+)="(.*)"$/) {
            $conf{$1} = $2;
        }
    }
    close($fh);
}
reload_conf();

my $SUB_PORT     = $conf{INT_SUB_PORT} || 8080;
my $SECRET_TOKEN = $conf{SUB_TOKEN}    || "";

# 2. 到期时间计算
sub date_to_ts {
    my ($y, $m, $d) = @_;
    return 0 unless defined $y && defined $m && defined $d;
    my @mdays = (31,28,31,30,31,30,31,31,30,31,30,31);
    if (($y % 4 == 0 && $y % 100 != 0) || $y % 400 == 0) {
        $mdays[1] = 29;
    }
    my $days = 0;
    for (my $i = 1970; $i < $y; $i++) {
        $days += (($i % 4 == 0 && $i % 100 != 0) || $i % 400 == 0) ? 366 : 365;
    }
    for (my $i = 0; $i < $m - 1; $i++) {
        $days += $mdays[$i];
    }
    $days += $d - 1;
    return $days * 86400;
}

# 3. 读取网卡物理流量（排除 lo 回环）
sub read_traffic {
    open(my $dfh, '<', '/proc/net/dev') or return (0, 0);
    my ($rx, $tx) = (0, 0);
    while (my $line = <$dfh>) {
        next if $line =~ /^\s*(lo|Inter-|face)/;
        if ($line =~ /^\s*(\S+?):\s*(.*)$/) {
            my $iface = $1;
            next if $iface eq 'lo';
            my @cols = split(/\s+/, $2);
            $rx += $cols[0] if defined $cols[0] && $cols[0] =~ /^\d+$/;
            $tx += $cols[8] if defined $cols[8] && $cols[8] =~ /^\d+$/;
        }
    }
    close($dfh);
    return ($rx, $tx);
}

# 4. 原生算法实现 Base64 编码
sub to_base64 {
    my $data = shift;
    my $res = "";
    my $chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/";
    my $len = length($data);
    for (my $i = 0; $i < $len; $i += 3) {
        my $chunk = substr($data, $i, 3);
        my $pad = 3 - length($chunk);
        $chunk .= "\0" x $pad;
        my $bytes = unpack("N", "\0" . $chunk);
        for (my $j = 0; $j < 4 - $pad; $j++) {
            $res .= substr($chars, ($bytes >> (18 - $j * 6)) & 0x3F, 1);
        }
        $res .= "=" x $pad;
    }
    return $res;
}

# 5. 生成 VLESS 分享链接
sub get_vless_link {
    reload_conf();
    my $name_enc = $conf{NODE_NAME} || "Reality-Node";
    $name_enc =~ s/([^a-zA-Z0-9_.~-])/sprintf("%%%02X", ord($1))/eg;
    return "vless://$conf{UUID}\@$conf{SERVER_IP}:$conf{EXT_NODE_PORT}?" .
           "encryption=none&flow=xtls-rprx-vision&security=reality&sni=gateway.icloud.com" .
           "&fp=chrome&pbk=$conf{PUBLIC_KEY}&sid=$conf{SHORT_ID}&type=tcp#$name_enc\n";
}

# 6. 初始化非阻塞服务器
my $server = IO::Socket::INET->new(
    LocalAddr => '0.0.0.0',
    LocalPort => $SUB_PORT,
    Proto     => 'tcp',
    Listen    => 20,
    ReuseAddr => 1
) or die "Cannot bind $SUB_PORT: $!\n";

my $sel = IO::Select->new($server);

while (my @ready = $sel->can_read) {
    for my $fh (@ready) {
        if ($fh == $server) {
            my $client = $server->accept();
            next unless $client;

            my $c_sel = IO::Select->new($client);
            my $req_line = "";
            if ($c_sel->can_read(3)) {
                $req_line = <$client> || "";
                while ($c_sel->can_read(0.2)) {
                    my $l = <$client>;
                    last unless defined $l;
                    $l =~ s/\r?\n$//;
                    last if $l eq "";
                }
            }

            # 鉴权
            if ($SECRET_TOKEN eq "" || index($req_line, $SECRET_TOKEN) != -1) {
                reload_conf();
                my ($rx, $tx) = read_traffic();
                my $base_bytes = int(($conf{USED_GB} || 0.00) * 1024 * 1024 * 1024);
                my $upload     = $tx + int($base_bytes / 2);
                my $download   = $rx + int($base_bytes / 2);
                my $total      = int(($conf{TRAFFIC_GB} || 400.00) * 1024 * 1024 * 1024);

                my ($y, $m, $d) = split(/-/, ($conf{EXPIRE_DATE} || '2026-12-31'));
                my $expire_ts = date_to_ts($y, $m, $d);

                my ($body, $ctype, $disp);
                if ($req_line =~ /GET\s+\/vless/i) {
                    $body = get_vless_link();
                    $ctype = "text/plain; charset=utf-8";
                    $disp  = "inline";
                } elsif ($req_line =~ /GET\s+\/base64/i) {
                    $body = to_base64(get_vless_link());
                    $ctype = "text/plain; charset=utf-8";
                    $disp  = "inline";
                } else {
                    my $tpl = '/opt/sing-box/ui/template.yaml';
                    if (open(my $yfh, '<', $tpl)) {
                        local $/;
                        $body = <$yfh>;
                        close($yfh);
                    } else {
                        $body = "# Template Missing\n";
                    }
                    $ctype = "text/yaml; charset=utf-8";
                    $disp  = "attachment; filename=\"config.yaml\"";
                }

                my $len = length($body);
                my $resp = "HTTP/1.1 200 OK\r\n" .
                           "Content-Type: $ctype\r\n" .
                           "Content-Disposition: $disp\r\n" .
                           "Content-Length: $len\r\n" .
                           "Subscription-Userinfo: upload=$upload; download=$download; total=$total; expire=$expire_ts\r\n" .
                           "Connection: close\r\n\r\n" .
                           $body;
                print $client $resp;
            } else {
                print $client "HTTP/1.1 403 Forbidden\r\nContent-Length: 0\r\nConnection: close\r\n\r\n";
            }
            close($client);
        }
    }
}
PERL_EOF

chmod +x /opt/sing-box/sub.pl
echo "[+] 步骤三完成：原生 Perl 动态多协议订阅服务构建成功！"
STEP3_EOF
bash /tmp/step3.sh
```

---

# 六、步骤四：systemd 进程守护与开机自启动构建

整段复制并粘贴执行。自动注册并启动两个独立的系统单元：

```bash
cat << 'STEP4_EOF' > /tmp/step4.sh
#!/bin/bash
set -e

echo "=================================================="
echo "          [4/5] 注册 systemd 系统守护服务         "
echo "=================================================="

# 1. 注册 sing-box 服务
cat << 'SVC1_EOF' > /etc/systemd/system/sing-box.service
[Unit]
Description=sing-box service
After=network.target nss-lookup.target network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
ExecStart=/opt/sing-box/sing-box run -c /opt/sing-box/config/config.json
Restart=always
RestartSec=3s
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
SVC1_EOF

# 2. 注册 Perl 动态订阅服务
cat << 'SVC2_EOF' > /etc/systemd/system/sing-box-sub.service
[Unit]
Description=sing-box Perl Dynamic Subscription Service
After=network.target network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
ExecStart=/usr/bin/perl /opt/sing-box/sub.pl
Restart=always
RestartSec=2s

[Install]
WantedBy=multi-user.target
SVC2_EOF

systemctl daemon-reload
systemctl enable sing-box sing-box-sub
systemctl restart sing-box sing-box-sub

echo "=================================================="
echo "[+] 服务拉起完成！本地监听自检状态："
ss -tulpn | grep -E "(sing-box|perl)"
echo "=================================================="
STEP4_EOF
bash /tmp/step4.sh
```

---

# 七、步骤五：一键提取全协议客户端订阅链接

整段复制并粘贴执行，终端将打印当前 VPS 的三种格式导入链接：

```bash
cat << 'STEP5_EOF' > /tmp/step5.sh
#!/bin/bash
set -e
source /opt/sing-box/env/node.env

NODE_NAME_URI=$(echo -n "$NODE_NAME" | jq -sRr @uri | tr -d '\r\n')
VLESS_DIRECT="vless://${UUID}@${SERVER_IP}:${EXT_NODE_PORT}?encryption=none&flow=xtls-rprx-vision&security=reality&sni=gateway.icloud.com&fp=chrome&pbk=${PUBLIC_KEY}&sid=${SHORT_ID}&type=tcp#${NODE_NAME_URI}"

echo "=================================================="
echo "          [5/5] 客户端多协议导入信息              "
echo "=================================================="
echo ""
echo "1. 【Clash Verge / Meta 专属订阅链接】"
echo "http://${SERVER_IP}:${EXT_SUB_PORT}/?token=${SUB_TOKEN}"
echo ""
echo "2. 【Base64 通用订阅链接】(支持 Shadowrocket / v2rayN 等)"
echo "http://${SERVER_IP}:${EXT_SUB_PORT}/base64?token=${SUB_TOKEN}"
echo ""
echo "3. 【VLESS 单节点直连链接】"
echo "$VLESS_DIRECT"
echo ""
echo "=================================================="
STEP5_EOF
bash /tmp/step5.sh
```

### Clash Verge 客户端配置导入步骤：
1. 复制终端输出的第 1 项 `http://${SERVER_IP}:${EXT_SUB_PORT}/?token=${SUB_TOKEN}`。
2. 打开 **Clash Verge**，进入左侧 **“订阅 (Profiles)”**。
3. 粘贴至输入框，点击 **“导入 (Import)”**。
4. 订阅卡片将以你设置的节点名称命名，并显示实时同步的已用流量与到期时间。
5. 切换至 **“代理 (Proxies)”** 界面点击闪电图标测速，节点将直接返回正常绿色延迟。

---

# 八、全系统一键自动化综合体检与故障自愈脚本

当遇到拉取失败或节点超时，运行以下全自动排查脚本：

```bash
cat << 'CHECK_EOF' > /opt/sing-box/check.sh
#!/bin/bash
clear
echo "=========================================================="
echo "          NAT VPS 节点与多协议订阅全自动自检诊断          "
echo "=========================================================="

ENV_FILE="/opt/sing-box/env/node.env"
if [ ! -f "$ENV_FILE" ]; then
    echo "[x] 错误：未找到环境文件 $ENV_FILE，请先运行步骤二！"
    exit 1
fi
source "$ENV_FILE"

echo "【1. 检查环境变量配置】"
echo "  - 公网 IP: $SERVER_IP"
echo "  - 节点外部端口: $EXT_NODE_PORT \vert{} 内部监听端口: $INT_NODE_PORT"
echo "  - 订阅外部端口: $EXT_SUB_PORT \vert{} 内部监听端口: $INT_SUB_PORT"
echo "  - UUID: $UUID"
echo "  - 公钥: $PUBLIC_KEY"

echo ""
echo "【2. 检查 systemd 服务运行状态】"
if systemctl is-active --quiet sing-box; then
    echo "  [OK] sing-box 节点核心服务：正在运行 (active)"
else
    echo "  [x] sing-box 服务异常！查看报错: journalctl -u sing-box -n 20"
fi

if systemctl is-active --quiet sing-box-sub; then
    echo "  [OK] sing-box-sub 订阅服务：正在运行 (active)"
else
    echo "  [x] sing-box-sub 订阅服务异常！查看报错: journalctl -u sing-box-sub -n 20"
fi

echo ""
echo "【3. 检查内部端口监听情况】"
PORT_CHECK=$(ss -tulpn)
if echo "$PORT_CHECK" \vert{} grep -q ":${INT_NODE_PORT} "; then
    echo "  [OK] 节点内部端口 ${INT_NODE_PORT} 正在正常监听"
else
    echo "  [x] 节点内部端口 ${INT_NODE_PORT} 未监听！"
fi

if echo "$PORT_CHECK" \vert{} grep -q ":${INT_SUB_PORT} "; then
    echo "  [OK] 订阅内部端口 ${INT_SUB_PORT} 正在正常监听"
else
    echo "  [x] 订阅内部端口 ${INT_SUB_PORT} 未监听！"
fi

echo ""
echo "【4. 本地订阅回环拉取测试】"
HTTP_CODE=$(curl -s -o /tmp/sub_test.yaml -w "%{http_code}" "[http://127.0.0.1](http://127.0.0.1):${INT_SUB_PORT}/?token=${SUB_TOKEN}")
if [ "$HTTP_CODE" = "200" ]; then
    echo "  [OK] 本地订阅拉取成功 (HTTP 200 OK)，文件大小: $(wc -c < /tmp/sub_test.yaml) 字节"
    CLIENT_SERVER=$(grep "server:" /tmp/sub_test.yaml | head -n1 | awk '{print $2}')
    CLIENT_PORT=$(grep "port:" /tmp/sub_test.yaml | head -n1 | awk '{print $2}')
    echo "  - 客户端下发节点地址: ${CLIENT_SERVER}:${CLIENT_PORT}"
else
    echo "  [x] 本地订阅拉取失败！HTTP 返回码: $HTTP_CODE"
fi
rm -f /tmp/sub_test.yaml

echo ""
echo "【5. 测试宿主机与 Reality 伪装域名握手】"
HANDSHAKE_TEST=$(curl -Iv --connect-timeout 5 [https://gateway.icloud.com:443](https://gateway.icloud.com:443) 2>&1)
if echo "$HANDSHAKE_TEST" | grep -q "SSL connection using"; then
    echo "  [OK] 小鸡与 gateway.icloud.com:443 TLS 1.3 握手通畅"
else
    echo "  [!] 提示：小鸡与 gateway.icloud.com 握手受阻，建议关注连接情况"
fi

echo ""
echo "=========================================================="
echo "诊断结论："
echo "如果以上全部为 [OK]，但电脑端 Clash Verge 导入依然提示失败或测速报 Timeout："
echo "1. 请去商家后台「端口转发」面板确认：外部端口 ${EXT_NODE_PORT} 与 ${EXT_SUB_PORT} 是否已添加转发规则！"
echo "2. 若电脑开着其它商业代理，请先断开或在规则中将小鸡公网 IP ${SERVER_IP} 设为 DIRECT 直连。"
echo "=========================================================="
CHECK_EOF
chmod +x /opt/sing-box/check.sh
bash /opt/sing-box/check.sh
```
