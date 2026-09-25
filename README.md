# 全通用 NAT VPS 极简节点与动态流量订阅部署指南

> **适用环境**：独角鲸云 / 碳云 / 微基主机等各类共享 IPv4 容器（Podman/LXC）与 KVM 虚拟机  
> **核心组件**：官方静态 sing-box (VLESS-REALITY-Vision) + 原生 Perl 5 动态流量订阅服务  
> **技术特性**：图文完整、分步透明、防翻译插件篡改、无外部库依赖、防 Web 终端死锁、彻底解决 Clash Verge 导入与测速 Timeout。

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
- [三、步骤一：环境与 sing-box 内核安装](#三步骤一环境与-sing-box-内核安装)
- [四、步骤二：参数交互引导与配置固化](#四步骤二参数交互引导与配置固化)
- [五、步骤三：拉起核心服务与动态订阅](#五步骤三拉起核心服务与动态订阅)
- [六、步骤四：客户端导入与测速](#六步骤四客户端导入与测速)
- [七、全功能一键自检与排错指南](#七全功能一键自检与排错指南)

---

## 一、准备工作：NAT VPS 购买与基础配置

### 1.1 登录注册入口
访问商家官网（以独角鲸云为例），点击右上角「登录 / 注册」。

![登录注册入口](https://github.com/user-attachments/assets/eece47e0-a361-4072-932c-3cb961d44742)

### 1.2 认证方式
支持邮箱验证码或第三方授权登录，按提示完成账户认证。

![认证方式](https://github.com/user-attachments/assets/fd196e13-32c4-4762-af2c-3bb32b9dbe29)

### 1.3 后台首页
登录后进入控制台，可在此查看实例列表、账单与端口配置。

![后台首页](https://github.com/user-attachments/assets/14165be5-1fe4-4729-9624-0f41d89993cb)

### 1.4 账单充值
确保账户有充足余额以购买或维持实例正常运行。

![账单充值](https://github.com/user-attachments/assets/aaed142f-6531-4964-8cb9-0c42e63652dd)

### 1.5 新建实例与选区
点击「新建实例」，选择靠近你的机房区域（如日本、香港、美国等）。

![新建实例与选区](https://github.com/user-attachments/assets/1fd2cc42-c06c-4f62-9197-cea3af093c75)

### 1.6 选配置套餐
根据预算选择合适的套餐规格，注意查看月流量与端口数量。

![选配置套餐](https://github.com/user-attachments/assets/dd62b50a-41e8-4186-a4c2-420d3df41781)

### 1.7 镜像选择系统
推荐选择 **Debian (Podman)** 或 Debian 12 镜像，兼容性最佳。

![镜像选择系统](https://github.com/user-attachments/assets/2bc73cb0-c22a-422b-89da-91237b58fd5c)

---

## 二、端口转发与 Web 控制台

### 2.1 端口转发规则
NAT VPS 使用共享公网 IPv4，必须通过母机做端口映射：
* **避开低位干扰**：强烈建议在面板中申请 **50000 以上的高位端口**（例如 `55555` 与 `55556`），避开 10000~30000 骨干网经常被 QoS 或丢包的号段。
* **确认两条 TCP 规则**：
  * **节点连接规则**：记录外部端口（例如 `55555`），内部端口通常一致。
  * **订阅服务规则**：记录外部端口（例如 `55556`），内部端口通常一致。

![端口转发规则](https://github.com/user-attachments/assets/c364fe94-c038-4c5d-8936-6629a70d8c75)

### 2.2 Web 控制台入口
进入实例详情页，点击「控制台」按钮调出网页终端。所有分步脚本均使用独立文件写入方式运行，防止网页终端粘贴截断。

![Web 控制台入口](https://github.com/user-attachments/assets/c1ab8281-ca61-4fc4-99b2-253d83deb611)

---

## 三、步骤一：环境与 sing-box 内核安装

整段复制并粘贴到 Web 终端执行。安装逻辑封装在独立文件中运行，自动识别硬件架构（amd64 / arm64），下载 URL 使用 Base64 编码防止网页翻译插件误篡改：

```bash
cat << 'STEP1_EOF' > /tmp/step1.sh
#!/bin/bash
set -e
export DEBIAN_FRONTEND=noninteractive

echo "[*] 更新系统并安装基础依赖..."
apt update -y && apt install -y -o Dpkg::Options::="--force-confdef" -o Dpkg::Options::="--force-confold" \
    curl wget tar perl ca-certificates openssl jq procps bc

ARCH_RAW=$(uname -m)
case "$ARCH_RAW" in
    x86_64)  ARCH="amd64" ;;
    aarch64) ARCH="arm64" ;;
    *) echo "[!] 不支持的硬件架构: $ARCH_RAW" && exit 1 ;;
esac

echo "[*] 系统硬件架构识别为: $ARCH"

TARGET="/opt/sing-box/sing-box"
mkdir -p /opt/sing-box /tmp/sb_dl
rm -f /tmp/sb_dl/sing-box.tar.gz

# Base64 编码防翻译插件篡改
B64_URL=$(echo "aHR0cHM6Ly9naGZhc3QudG9wL2h0dHBzOi8vZ2l0aHViLmNvbS9TYWdlck5ldC9zaW5nLWJveC9yZWxlYXNlcy9kb3dubG9hZC92MS4xMS40L3NpbmctYm94LTEuMTEuNC1saW51eC0ke0FSQ0h9LnRhci5neg==" | base64 -d | sed "s/\${ARCH}/$ARCH/g")

echo "[*] 正在下载 sing-box 官方内核..."
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

## 四、步骤二：参数交互引导与配置固化

整段复制并粘贴执行。自动引导参数填写，分离收集内外端口，并写入符合 Clash Meta / Verge 标准（含 `format: text` 声明、Fake-IP 过滤、Packet-Addr 对齐、Chrome 指纹）的完整 Loyalsoldier 分流模板：

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
echo "          NAT VPS 节点交互式配置向导              "
echo "=================================================="

if [ -n "$DETECT_IP" ]; then
    read -p "1. 确认小鸡公网 IPv4 [默认探测: ${DETECT_IP}]: " INPUT_IP
    SERVER_IP=${INPUT_IP:-$DETECT_IP}
else
    read -p "1. 请输入商家后台分配的公网 IPv4: " INPUT_IP
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
CONF_EOF

chmod 600 "$CONF_FILE"

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

/opt/sing-box/sing-box check -c /opt/sing-box/config.json && echo "[+] 服务端配置格式校验通过"

# 2. 写入完整的 Loyalsoldier 全规则 template.yaml
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

echo "[+] 步骤二完成，参数已成功保存至 $CONF_FILE"
STEP2_EOF
bash /tmp/step2.sh
```

---

## 五、步骤三：拉起核心服务与动态订阅

整段复制并粘贴执行。自动配置双 systemd 服务单元，Perl 脚本内置超时套接字与安全 Header 机制，杜绝进程挂死：

```bash
cat << 'STEP3_EOF' > /tmp/step3.sh
#!/bin/bash
set -e

# 1. 写入 sing-box 系统服务
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

[Install]
WantedBy=multi-user.target
SVC_EOF

# 2. 写入原生 Perl 动态订阅服务
cat << 'PERL_EOF' > /opt/sing-box/sub.pl
#!/usr/bin/perl
use strict;
use warnings;
use IO::Socket::INET;
use IO::Select;

my %conf;
my $conf_file = '/opt/sing-box/deploy.conf';
open(my $fh, '<', $conf_file) or die "Cannot open $conf_file: $!";
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

my ($y, $m, $d) = split(/-/, $EXPIRE_DATE);
my $expire_ts = date_to_ts($y, $m, $d);

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
            my $req = "";
            if ($c_sel->can_read(3)) {
                $req = <$client> || "";
                while ($c_sel->can_read(0.5)) {
                    my $line = <$client>;
                    last unless defined $line;
                    $line =~ s/\r?\n$//;
                    last if $line eq "";
                }
            }

            if ($SECRET_TOKEN eq "" || index($req, $SECRET_TOKEN) != -1) {
                my ($rx, $tx) = read_traffic();
                my $base_bytes = int($BASE_USED_GB * 1024 * 1024 * 1024);
                my $upload     = $tx + int($base_bytes / 2);
                my $download   = $rx + int($base_bytes / 2);
                my $total      = int($TOTAL_GB * 1024 * 1024 * 1024);

                my $tpl = '/opt/sing-box/ui/template.yaml';
                if (open(my $yfh, '<', $tpl)) {
                    local $/;
                    my $body = <$yfh>;
                    close($yfh);
                    my $len = length($body);

                    my $resp = "HTTP/1.1 200 OK\r\n" .
                               "Content-Type: text/yaml; charset=utf-8\r\n" .
                               "Content-Disposition: attachment; filename=\"config.yaml\"\r\n" .
                               "Content-Length: $len\r\n" .
                               "Subscription-Userinfo: upload=$upload; download=$download; total=$total; expire=$expire_ts\r\n" .
                               "Connection: close\r\n\r\n" .
                               $body;
                    print $client $resp;
                } else {
                    print $client "HTTP/1.1 500 Internal Server Error\r\nContent-Length: 0\r\nConnection: close\r\n\r\n";
                }
            } else {
                print $client "HTTP/1.1 403 Forbidden\r\nContent-Length: 0\r\nConnection: close\r\n\r\n";
            }
            close($client);
        }
    }
}
PERL_EOF

chmod +x /opt/sing-box/sub.pl

# 3. 写入订阅 systemd 服务
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

## 六、步骤四：客户端导入与测速

运行以下命令，打印专属订阅链接与直连节点参数：

```bash
cat << 'STEP4_EOF' > /tmp/step4.sh
#!/bin/bash
set -e
source /opt/sing-box/deploy.conf

NODE_NAME_URI=$(echo -n "$NODE_NAME" | jq -sRr @uri | tr -d '\r\n')
VLESS_LINK="vless://${UUID}@${SERVER_IP}:${EXT_NODE_PORT}?encryption=none&flow=xtls-rprx-vision&security=reality&sni=gateway.icloud.com&fp=chrome&pbk=${PUBLIC_KEY}&sid=${SHORT_ID}&type=tcp&headerType=none#${NODE_NAME_URI}"

echo "=================================================="
echo "                客户端导入信息                    "
echo "=================================================="
echo ""
echo "【专属 Clash 订阅链接】(直接复制并在 Clash Verge 中导入)"
echo "http://${SERVER_IP}:${EXT_SUB_PORT}/sub?token=${SUB_TOKEN}"
echo ""
echo "【VLESS 备用直连链接】"
echo "$VLESS_LINK"
echo ""
echo "=================================================="
STEP4_EOF
bash /tmp/step4.sh
```

### Clash Verge 客户端配置导入步骤：
1. 复制终端输出的 `http://${SERVER_IP}:${EXT_SUB_PORT}/sub?token=${SUB_TOKEN}` 专属订阅链接。
2. 打开 **Clash Verge**，进入左侧 **“订阅 (Profiles)”**。
3. 粘贴至输入框，点击 **“导入 (Import)”**。
4. 订阅卡片将以你设置的节点名称命名，并显示实时同步的已用流量与到期时间。
5. 切换至 **“代理 (Proxies)”** 界面点击闪电图标测速，节点将直接返回正常绿色延迟。

---

## 七、全功能一键自检与排错指南

如果遇到导入失败或节点超时，整段复制并在小鸡终端运行以下脚本，自动定位问题：

```bash
cat << 'CHECK_EOF' > /tmp/check.sh
#!/bin/bash
clear
echo "=========================================================="
echo "          NAT VPS 节点与订阅全自动自检诊断                "
echo "=========================================================="

CONF="/opt/sing-box/deploy.conf"
if [ ! -f "$CONF" ]; then
    echo "[x] 错误：未找到配置文件 $CONF，请先运行步骤二！"
    exit 1
fi
source "$CONF"

echo "【1. 检查配置参数】"
echo "  - 公网 IP: $SERVER_IP"
echo "  - 节点外部端口: $EXT_NODE_PORT \vert{} 内部监听端口: $INT_NODE_PORT"
echo "  - 订阅外部端口: $EXT_SUB_PORT \vert{} 内部监听端口: $INT_SUB_PORT"
echo "  - UUID: $UUID"
echo "  - 公钥: $PUBLIC_KEY"
echo "  - Short ID: $SHORT_ID"

echo ""
echo "【2. 检查 systemd 服务运行状态】"
if systemctl is-active --quiet sing-box; then
    echo "  [OK] sing-box 核心服务：正在运行 (active)"
else
    echo "  [x] sing-box 服务异常！状态: $(systemctl is-active sing-box)"
fi

if systemctl is-active --quiet sing-box-sub; then
    echo "  [OK] sing-box-sub 订阅服务：正在运行 (active)"
else
    echo "  [x] sing-box-sub 订阅服务异常！状态: $(systemctl is-active sing-box-sub)"
fi

echo ""
echo "【3. 检查内部端口监听】"
PORT_CHECK=$(ss -tulpn)
if echo "$PORT_CHECK" \vert{} grep -q ":${INT_NODE_PORT} "; then
    echo "  [OK] 节点内部端口 ${INT_NODE_PORT} 正常监听"
else
    echo "  [x] 节点内部端口 ${INT_NODE_PORT} 未监听！"
fi

if echo "$PORT_CHECK" \vert{} grep -q ":${INT_SUB_PORT} "; then
    echo "  [OK] 订阅内部端口 ${INT_SUB_PORT} 正常监听"
else
    echo "  [x] 订阅内部端口 ${INT_SUB_PORT} 未监听！"
fi

echo ""
echo "【4. 本地订阅拉取与内容校验】"
SUB_CODE=$(curl -s -o /tmp/sub_test.yaml -w "%{http_code}" "[http://127.0.0.1](http://127.0.0.1):${INT_SUB_PORT}/sub?token=${SUB_TOKEN}")
if [ "$SUB_CODE" = "200" ]; then
    echo "  [OK] 本地订阅拉取成功 (HTTP 200 OK)，文件大小: $(wc -c < /tmp/sub_test.yaml) 字节"
    CLIENT_SERVER=$(grep "server:" /tmp/sub_test.yaml | head -n1 | awk '{print $2}')
    CLIENT_PORT=$(grep "port:" /tmp/sub_test.yaml | head -n1 | awk '{print $2}')
    CLIENT_PBK=$(grep "public-key:" /tmp/sub_test.yaml | head -n1 | awk '{print $2}')
    echo "  - 客户端下发节点地址: ${CLIENT_SERVER}:${CLIENT_PORT}"
    echo "  - 客户端下发公钥匹配: $([ "$CLIENT_PBK" = "$PUBLIC_KEY" ] && echo "[OK] 一致" || echo "[x] 不一致!")"
else
    echo "  [x] 本地订阅拉取失败！HTTP 返回码: $SUB_CODE"
fi
rm -f /tmp/sub_test.yaml

echo ""
echo "【5. 测试宿主机与 Reality 伪装域名握手】"
HANDSHAKE_TEST=$(curl -Iv --connect-timeout 5 [https://gateway.icloud.com:443](https://gateway.icloud.com:443) 2>&1)
if echo "$HANDSHAKE_TEST" | grep -q "SSL connection using"; then
    echo "  [OK] 小鸡与 gateway.icloud.com:443 TLS 握手正常"
else
    echo "  [!] 提示：小鸡与 gateway.icloud.com 握手受阻，建议更换 SNI 域名"
fi

echo ""
echo "=========================================================="
echo "诊断结论："
echo "如果以上全部为 [OK]，但电脑端 Clash Verge 导入依然提示失败或测速报 Timeout："
echo "1. 请去商家后台「端口转发」面板确认：外部端口 ${EXT_NODE_PORT} 与 ${EXT_SUB_PORT} 是否已添加转发规则！"
echo "2. 若电脑开着其它商业代理，请先断开或在规则中将小鸡公网 IP ${SERVER_IP} 设为 DIRECT 直连。"
echo "=========================================================="
CHECK_EOF
bash /tmp/check.sh
```
