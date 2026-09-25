# 全通用 NAT VPS 极简节点与动态流量订阅部署指南

> **适用环境**：独角鲸云 / 各类共享 IPv4 容器（Podman/LXC）与 KVM 虚拟机  
> **核心组件**：sing-box (VLESS-REALITY-Vision) + 原生 Perl 5 动态流量订阅服务  
> **部署方式**：透明、模块化、分步交互式，拒绝一键黑盒脚本  
> **架构支持**：自动适配 x86_64 (amd64) 与 aarch64 (arm64)

---

## 目录
- [一、准备工作：NAT VPS 购买与基础配置](#一准备工作nat-vps-购买与基础配置)
  - [1.1 登录注册入口](#11-登录注册入口)
  - [1.2 认证方式](#12-认证方式)
  - [1.3 后台首页](#13-后台首页)
  - [1.4 账单充值](#14-账单充值)
  - [1.5 新建实例与选区](#15-新建实例与选区)
  - [1.6 选配置套餐](#16-选配置套餐)
  - [1.7 镜像选择系统](#17-镜像选择系统)
- [二、端口转发与 Web 控制台](#二端口转发与-web-控制台)
  - [2.1 端口转发规则](#21-端口转发规则)
  - [2.2 Web 控制台入口](#22-web-控制台入口)
- [三、步骤一：环境与内核安装](#三步骤一环境与内核安装)
- [四、步骤二：逐项参数交互输入并持久化](#四步骤二逐项参数交互输入并持久化)
- [五、步骤三：拉起系统服务](#五步骤三拉起系统服务)
- [六、步骤四：输出客户端导入链接](#六步骤四输出客户端导入链接)
- [七、Perl 动态流量统计服务设计](#七perl-动态流量统计服务设计)
- [八、常见问题与排错](#八常见问题与排错)

---

## 一、准备工作：NAT VPS 购买与基础配置

### 1.1 登录注册入口
访问商家官网完成账号注册。进入官网后，点击右上角「登录 / 注册」按钮。

![登录注册入口](https://github.com/user-attachments/assets/eece47e0-a361-4072-932c-3cb961d44742)

### 1.2 认证方式
支持邮箱验证码或第三方授权登录，按提示完成身份认证。

![认证方式](https://github.com/user-attachments/assets/fd196e13-32c4-4762-af2c-3bb32b9dbe29)

### 1.3 后台首页
登录后进入用户后台。可在此查看已有实例、账单与账户状态。

![后台首页](https://github.com/user-attachments/assets/14165be5-1fe4-4729-9624-0f41d89993cb)

### 1.4 账单充值
确保账户余额充足，避免实例因欠费被暂停。点击「充值」选择支付方式完成到账。

![账单充值](https://github.com/user-attachments/assets/aaed142f-6531-4964-8cb9-0c42e63652dd)

### 1.5 新建实例与选区
点击「新建实例」，选择机房区域（如日本、美国等）。

![新建实例与选区](https://github.com/user-attachments/assets/1fd2cc42-c06c-4f62-9197-cea3af093c75)

### 1.6 选配置套餐
根据预算与用途选择配置规格，确认「月流量」与「端口数量」。

![选配置套餐](https://github.com/user-attachments/assets/dd62b50a-41e8-4186-a4c2-420d3df41781)

### 1.7 镜像选择系统
推荐选择 **Debian (Podman)** 或 Debian 12 镜像，兼容性最佳。

![镜像选择系统](https://github.com/user-attachments/assets/2bc73cb0-c22a-422b-89da-91237b58fd5c)

---

## 二、端口转发与 Web 控制台

### 2.1 端口转发规则
NAT VPS 的关键差异在于端口映射：
* **独角鲸云等商家**：内外端口通常一致（建议分配使用 50000 以上的高位端口，如 `55555` 与 `55556`，避开骨干网低位干扰）。
* **部分传统 NAT 商家**：外部端口随机分配（如 `48921`），容器内部只开放固定端口（如 `10086`）。

请在商家面板中确认并记录两条 TCP 转发规则：
1. **节点服务端口**：记录外部端口与内部端口。
2. **订阅服务端口**：记录外部端口与内部端口。

![端口转发规则](https://github.com/user-attachments/assets/c364fe94-c038-4c5d-8936-6629a70d8c75)

### 2.2 Web 控制台入口
通过商家提供的 Web 终端（xterm.js）登录小鸡。后续命令均写入独立脚本文件后执行，防止网页终端粘贴换行截断。

![Web 控制台入口](https://github.com/user-attachments/assets/c1ab8281-ca61-4fc4-99b2-253d83deb611)

---

## 三、步骤一：环境与内核安装

整段复制并粘贴到 Web 终端执行。安装逻辑写入 `/tmp/step1.sh` 独立运行，使用 Base64 隔离 URL 并自动判定硬件架构（amd64 / arm64）：

```bash
cat << 'STEP1_EOF' > /tmp/step1.sh
#!/bin/bash
set -e
export DEBIAN_FRONTEND=noninteractive

echo "[*] 更新源并安装基础依赖..."
apt update -y && apt install -y -o Dpkg::Options::="--force-confdef" -o Dpkg::Options::="--force-confold" \
    curl wget tar perl ca-certificates openssl jq procps bc

ARCH_RAW=$(uname -m)
case "$ARCH_RAW" in
    x86_64)  ARCH="amd64" ;;
    aarch64) ARCH="arm64" ;;
    *) echo "[!] 不支持的硬件架构: $ARCH_RAW" && exit 1 ;;
esac

echo "[*] 检测系统架构: $ARCH"

TARGET="/opt/sing-box/sing-box"
mkdir -p /opt/sing-box /tmp/sb_dl
rm -f /tmp/sb_dl/sing-box.tar.gz

# Base64 物理隔离，防翻译插件注入 Markdown 语法
B64_URL=$(echo "aHR0cHM6Ly9naGZhc3QudG9wL2h0dHBzOi8vZ2l0aHViLmNvbS9TYWdlck5ldC9zaW5nLWJveC9yZWxlYXNlcy9kb3dubG9hZC92MS4xMS40L3NpbmctYm94LTEuMTEuNC1saW51eC0ke0FSQ0h9LnRhci5neg==" | base64 -d | sed "s/\${ARCH}/$ARCH/g")

echo "[*] 正在拉取内核..."
curl -fsSL -o /tmp/sb_dl/sing-box.tar.gz "$B64_URL"

tar -zxvf /tmp/sb_dl/sing-box.tar.gz -C /tmp/sb_dl/
mv /tmp/sb_dl/sing-box-*/sing-box "$TARGET"
chmod +x "$TARGET"
ln -sf "$TARGET" /usr/local/bin/sing-box
rm -rf /tmp/sb_dl

if "$TARGET" version &>/dev/null; then
    echo "=================================================="
    echo "[+] sing-box 内核安装成功！版本: $("$TARGET" version | head -n1)"
    echo "=================================================="
else
    echo "[!] 内核验证失败，请检查网络连接。"
    exit 1
fi
STEP1_EOF
bash /tmp/step1.sh
```

---

## 四、步骤二：逐项参数交互输入并持久化

本步骤将【外部公网端口】与【内部监听端口】彻底分离收集，支持直接回车默认内外一致。配置持久化保存于 `/opt/sing-box/deploy.conf`，并生成服务端 `config.json` 与客户端完整的 Loyalsoldier 全规则分流模板 `template.yaml`：

```bash
cat << 'STEP2_EOF' > /tmp/step2.sh
#!/bin/bash
set -e
CONF_DIR="/opt/sing-box"
CONF_FILE="$CONF_DIR/deploy.conf"
mkdir -p "$CONF_DIR/ui"

P="h""t""t""p"
DETECT_IP=$(curl -s4m 3 "$P://ip.sb" || curl -s4m 3 "$P://ifconfig.me" || curl -s4m 3 "$P://api.ipify.org" || echo "")

echo "=================================================="
echo "      NAT VPS 节点与订阅交互式配置向导            "
echo "=================================================="

if [ -n "$DETECT_IP" ]; then
    read -p "1. 确认小鸡公网 IPv4 [默认探测: ${DETECT_IP}]: " INPUT_IP
    SERVER_IP=${INPUT_IP:-$DETECT_IP}
else
    read -p "1. 请输入商家后台显示的公网 IPv4: " INPUT_IP
    SERVER_IP=${INPUT_IP}
fi

read -p "2. 节点【外部公网端口】(客户端连接用, 例如 55555): " EXT_NODE_PORT
read -p "   节点【内部监听端口】[内外一致直接回车，默认: ${EXT_NODE_PORT}]: " INT_NODE_PORT
INT_NODE_PORT=${INT_NODE_PORT:-$EXT_NODE_PORT}

read -p "3. 订阅【外部公网端口】(客户端拉取订阅用, 例如 55556): " EXT_SUB_PORT
read -p "   订阅【内部监听端口】[内外一致直接回车，默认: ${EXT_SUB_PORT}]: " INT_SUB_PORT
INT_SUB_PORT=${INT_SUB_PORT:-$EXT_SUB_PORT}

read -p "4. 客户端显示的卡片名称 [默认: 日本自建（400g）]: " INPUT_NAME
NODE_NAME=${INPUT_NAME:-"日本自建（400g）"}

read -p "5. 总流量额度(GB) [纯数字, 默认: 400]: " INPUT_TOTAL
TRAFFIC_GB=${INPUT_TOTAL:-400}

read -p "6. 已用过的流量底数(GB) [支持两位小数，新机直接回车填 0.00]: " INPUT_USED
USED_GB=${INPUT_USED:-0.00}

read -p "7. 面板到期时间 [格式 YYYY-MM-DD，直接回车默认当前+30天]: " INPUT_DATE
if [ -z "$INPUT_DATE" ]; then
    EXPIRE_DATE=$(date -d "+30 days" +%Y-%m-%d 2>/dev/null || date -v+30d +%Y-%m-%d)
else
    EXPIRE_DATE="$INPUT_DATE"
fi

RAND_TOKEN=$(tr -dc A-Za-z0-9 </dev/urandom | head -c 16)
read -p "8. 订阅安全 Token [直接回车随机生成: ${RAND_TOKEN}]: " INPUT_TOKEN
SUB_TOKEN=${INPUT_TOKEN:-$RAND_TOKEN}

UUID=$(/opt/sing-box/sing-box generate uuid | tr -d "\r\n ")
KEYPAIR=$(/opt/sing-box/sing-box generate reality-keypair)
PRIVATE_KEY=$(echo "$KEYPAIR" | grep "PrivateKey" | awk '{print $2}' | tr -d "\r\n ")
PUBLIC_KEY=$(echo "$KEYPAIR" | grep "PublicKey" | awk '{print $2}' | tr -d "\r\n ")
SHORT_ID=$(openssl rand -hex 8 | tr -d "\r\n ")

cat << CONF_EOF> "$CONF_FILE"
SERVER_IP="$SERVER_IP"
EXT_NODE_PORT="$EXT_NODE_PORT"
INT_NODE_PORT="$INT_NODE_PORT"
EXT_SUB_PORT="$EXT_SUB_PORT"
INT_SUB_PORT="$INT_SUB_PORT"
NODE_NAME="$NODE_NAME"
TRAFFIC_GB="$TRAFFIC_GB"
USED_GB="$USED_GB"
EXPIRE_DATE="$EXPIRE_DATE"
SUB_TOKEN="$SUB_TOKEN"
UUID="$UUID"
PRIVATE_KEY="$PRIVATE_KEY"
PUBLIC_KEY="$PUBLIC_KEY"
SHORT_ID="$SHORT_ID"
CONF_EOF

chmod 600 "$CONF_FILE"

# 1. 写入服务端 sing-box 配置
cat << SBOF > /opt/sing-box/config.json
{
  "log": { "level": "warn" },
  "inbounds": [
    {
      "type": "vless",
      "tag": "vless-in",
      "listen": "0.0.0.0",
      "listen_port": $INT_NODE_PORT,
      "users": [{ "uuid": "$UUID", "flow": "xtls-rprx-vision" }],
      "tls": {
        "enabled": true,
        "server_name": "gateway.icloud.com",
        "reality": {
          "enabled": true,
          "handshake": { "server": "gateway.icloud.com", "server_port": 443 },
          "private_key": "$PRIVATE_KEY",
          "short_id": ["$SHORT_ID"]
        }
      }
    }
  ],
  "outbounds": [{ "type": "direct", "tag": "direct" }]
}
SBOF

/opt/sing-box/sing-box check -c /opt/sing-box/config.json && echo "[+] 服务端配置格式校验通过"

# 2. 写入完整的 Loyalsoldier 全规则客户端 template.yaml
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
  - name: "$NODE_NAME"
    type: vless
    server: $SERVER_IP
    port: $EXT_NODE_PORT
    uuid: $UUID
    network: tcp
    tls: true
    udp: false
    flow: xtls-rprx-vision
    servername: gateway.icloud.com
    reality-opts:
      public-key: $PUBLIC_KEY
      short-id: $SHORT_ID
    client-fingerprint: chrome

proxy-groups:
  - name: "国外流量"
    type: select
    proxies:
      - "$NODE_NAME"
      - DIRECT
  - name: "国外媒体"
    type: select
    proxies:
      - "$NODE_NAME"
      - "国外流量"
  - name: "AI平台"
    type: select
    proxies:
      - "$NODE_NAME"
      - "国外流量"
  - name: "微软服务"
    type: select
    proxies:
      - DIRECT
      - "$NODE_NAME"
  - name: "苹果服务"
    type: select
    proxies:
      - DIRECT
      - "$NODE_NAME"
  - name: "漏网之鱼"
    type: select
    proxies:
      - "国外流量"
      - DIRECT

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
YAMLOF

echo "[+] 步骤二完成，参数已固化至 $CONF_FILE"
STEP2_EOF
bash /tmp/step2.sh
```

---

## 五、步骤三：拉起系统服务

构建 sing-box 核心服务与防扫描 Token 鉴权的 Perl 动态订阅服务，自动加入开机自启并拉起：

```bash
cat << 'STEP3_EOF' > /tmp/step3.sh
#!/bin/bash
set -e

# 1. 写入 sing-box systemd 单元
cat << 'SVC_EOF' > /etc/systemd/system/sing-box.service
[Unit]
Description=sing-box service
After=network.target nss-lookup.target network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
ExecStart=/opt/sing-box/sing-box run -c /opt/sing-box/config.json
Restart=always
RestartSec=3s
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
SVC_EOF

# 2. 写入 Perl 动态流量统计服务（带 Token 鉴权 + lo 过滤 + 纯算数时间戳）
cat << 'PERL_EOF' > /opt/sing-box/sub.pl
#!/usr/bin/perl
use strict;
use warnings;
use IO::Socket::INET;

my %conf;
open(my $fh, '<', '/opt/sing-box/deploy.conf') or die "Cannot open deploy.conf: $!";
while (my $line = <$fh>) {
    chomp $line;
    if ($line =~ /^(\w+)="(.*)"$/) {
        $conf{$1} = $2;
    }
}
close($fh);

my $SUB_PORT     = $conf{INT_SUB_PORT} || 8080;
my $SECRET_TOKEN = $conf{SUB_TOKEN}    || "";
my $BASE_USED_GB = $conf{USED_GB}      || 0.00;
my $TOTAL_GB     = $conf{TRAFFIC_GB}   || 400.00;
my $EXPIRE_DATE  = $conf{EXPIRE_DATE}  || '2026-12-31';
my $NODE_NAME    = $conf{NODE_NAME}    || 'Node';

# 纯算数天数累加算法，摆脱系统 Time::Local 缺失依赖
sub date_to_ts {
    my ($y, $m, $d) = @_;
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

my ($y, $m, $d) = split(/-/, $EXPIRE_DATE);
my $expire_ts = date_to_ts($y, $m, $d);

# 读取 /proc/net/dev 统计真实网卡流量，显式过滤 lo 接口
sub read_traffic {
    open(my $fh, '<', '/proc/net/dev') or return (0, 0);
    my ($rx, $tx) = (0, 0);
    while (my $line = <$fh>) {
        next if $line =~ /^\s*(lo|Inter-|face)/;
        if ($line =~ /^\s*(\S+):\s*(\d+)\s+\d+\s+\d+\s+\d+\s+\d+\s+\d+\s+\d+\s+\d+\s+(\d+)/) {
            next if $1 eq 'lo';
            $rx += $2;
            $tx += $3;
        }
    }
    close($fh);
    return ($rx, $tx);
}

my $NAME_ENCODED = $NODE_NAME;
$NAME_ENCODED =~ s/([^a-zA-Z0-9_.~-])/sprintf("%%%02X", ord($1))/eg;

my $server = IO::Socket::INET->new(
    LocalAddr => '0.0.0.0',
    LocalPort => $SUB_PORT,
    Proto     => 'tcp',
    Listen    => 20,
    ReuseAddr => 1
) or die "Cannot bind $SUB_PORT: $!\n";

while (my $client = $server->accept()) {
    my $req_line = <$client> || "";
    
    # 门禁校验：请求行必须携带正确 Token，彻底防止全网测绘泄露节点敏感凭据
    if ($SECRET_TOKEN eq "" || index($req_line, $SECRET_TOKEN) != -1) {
        my ($rx, $tx) = read_traffic();
        my $base_bytes = int($BASE_USED_GB * 1024 * 1024 * 1024);
        my $upload     = $tx + int($base_bytes / 2);
        my $download   = $rx + int($base_bytes / 2);
        my $total      = int($TOTAL_GB * 1024 * 1024 * 1024);

        if (open(my $y_fh, '<', '/opt/sing-box/ui/template.yaml')) {
            local $/;
            my $body = <$y_fh>;
            close($y_fh);
            my $len = length($body);

            my $header = "HTTP/1.1 200 OK\r\n" .
                         "Content-Type: text/yaml; charset=utf-8\r\n" .
                         "Content-Disposition: attachment; filename=\"config.yaml\"; filename*=UTF-8''${NAME_ENCODED}.yaml\r\n" .
                         "Content-Length: $len\r\n" .
                         "Subscription-Userinfo: upload=$upload; download=$download; total=$total; expire=$expire_ts\r\n" .
                         "Connection: close\r\n\r\n";
            print $client $header;
            print $client $body;
        }
    } else {
        # 阻断未授权嗅探
        print $client "HTTP/1.1 403 Forbidden\r\nContent-Length: 0\r\nConnection: close\r\n\r\n";
    }
    close($client);
}
PERL_EOF

chmod +x /opt/sing-box/sub.pl

# 3. 写入 Perl 订阅 systemd 单元
cat << 'SVC2_EOF' > /etc/systemd/system/sing-box-sub.service
[Unit]
Description=sing-box Perl Subscription Service
After=network.target network-online.target
Wants=network-online.target

[Service]
Type=simple
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
echo "[+] 服务拉起完成！监听端口状态自检："
ss -tulpn | grep -E "(sing-box|perl)"
echo "=================================================="
STEP3_EOF
bash /tmp/step3.sh
```

---

## 六、步骤四：输出客户端导入链接

整段复制并粘贴执行，终端将打印格式化的客户端导入链接：

```bash
cat << 'STEP4_EOF' > /tmp/step4.sh
#!/bin/bash
set -e
source /opt/sing-box/deploy.conf

VLESS_LINK="vless://${UUID}@${SERVER_IP}:${EXT_NODE_PORT}?encryption=none&flow=xtls-rprx-vision&security=reality&sni=gateway.icloud.com&fp=chrome&pbk=${PUBLIC_KEY}&sid=${SHORT_ID}&type=tcp&headerType=none#${NODE_NAME}"

echo "=================================================="
echo "                客户端导入信息                    "
echo "=================================================="
echo ""
echo "【专属 Clash 订阅链接】"
echo "http://${SERVER_IP}:${EXT_SUB_PORT}/token=${SUB_TOKEN}&name=${NODE_NAME}.yaml"
echo ""
echo "【VLESS 分享链接】"
echo "$VLESS_LINK"
echo ""
echo "=================================================="
STEP4_EOF
bash /tmp/step4.sh
```

### Clash Verge 客户端导入操作：
1. 复制终端输出的 `http://...` 专属订阅链接。
2. 打开 **Clash Verge**，进入左侧 **“订阅 (Profiles)”**。
3. 粘贴至输入框，点击 **“导入 (Import)”**。
4. 订阅卡片将以你设置的节点名称命名，并显示实时同步的已用流量与到期时间。
5. 切换至 **“代理 (Proxies)”** 界面点击闪电图标测速，节点将返回正常延迟。

---

## 七、Perl 动态流量统计服务设计

### 核心实现特点：
* **零外部模块依赖**：使用内置纯算数算法解析天数计算 Unix 时间戳，杜绝因精简系统缺失 `Time::Local` 引起的报错退出。
* **物理网卡真实过滤**：读取 `/proc/net/dev` 时显式排除 `lo` 接口，避免内部进程通信导致流量虚高。
* **底数上下行均摊**：将用户填写的底数按 `50% / 50%` 分配给上传和下载，符合 Clash 客户端卡片展示直觉。
* **安全鉴权门禁**：对进站 HTTP 请求行强校验 `SUB_TOKEN`，未带有效 Token 的请求一律返回 403 并阻断连接，保护节点公钥、UUID 与端口配置不被测绘扫出。

---

## 八、常见问题与排错

### 1. 订阅拉取正常，但节点测速报 Timeout
* **外部端口被阻断**：部分商家的 10000~30000 号段端口容易受到骨干网阻断。请在后台端口转发中，将外部端口更换为 50000 以上的高位端口（如 `55555`）。
* **母鸡端口映射未生效**：在小鸡终端运行抓包命令检查是否有数据到达：
  ```bash
  tcpdump -nn -i any port 你的内部节点端口
  ```
  在客户端触发测速，若终端无任何文字刷新，表明数据包在母鸡外部已被拦截。

### 2. 开着其他代理时，无法更新自建订阅
电脑开着商业代理时，商业节点的出口防火墙通常会屏蔽高位自定义端口。
* **一劳永逸解法**：在主力订阅的配置规则顶部添加小鸡公网 IP 直连规则：
  ```yaml
  rules:
    - IP-CIDR,你的小鸡公网IP/32,DIRECT,no-resolve
  ```

### 3. 查看运行日志与重启服务
* 重启服务：
  ```bash
  systemctl restart sing-box sing-box-sub
  ```
* 查看日志：
  ```bash
  journalctl -u sing-box -n 50 --no-pager
  journalctl -u sing-box-sub -n 50 --no-pager
  ```
