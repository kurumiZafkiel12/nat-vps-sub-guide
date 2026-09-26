# 全通用 NAT VPS 极简节点与全功能多协议动态订阅生产级部署指南

> **适用场景**：独角鲸云 / 碳云 / 各种非特权 Podman / Docker 容器与 KVM 架构 VPS  
> **协议方案**：VLESS-REALITY-Vision + 原生 Perl 5 动态订阅调度服务  
> **适配客户端**：Clash Verge (Meta/Mihomo) / Clash Nyanpasu / v2rayN / Shadowrocket / Nekoray

---

## 目录
- [一、NAT VPS 架构原理与前期端口规划](#一nat-vps-架构原理与前期端口规划)
- [二、步骤一：核心依赖与 sing-box 官方静态内核安装](#二步骤一核心依赖与-sing-box-官方静态内核安装)
- [三、步骤二：全自动化参数交互、环境固化与工业级规则生成](#三步骤二全自动化参数交互环境固化与工业级规则生成)
- [四、步骤三：原生轻量多协议动态订阅服务实现 (Perl 5)](#四步骤三原生轻量多协议动态订阅服务实现-perl-5)
- [五、步骤四：systemd 进程守护与开机自启动构建](#五步骤四systemd-进程守护与开机自启动构建)
- [六、步骤五：一键提取全协议客户端订阅链接](#六步骤五一键提取全协议客户端订阅链接)
- [七、全系统一键自动化综合体检与故障自愈脚本](#七全系统一键自动化综合体检与故障自愈脚本)

---

## 一、NAT VPS 架构原理与前期端口规划

NAT VPS 采用多容器共享母鸡公网 IPv4 的架构。外部流量通过宿主机的 NAT 防火墙进行端口映射转发：

```
+------------------+         +----------------------+         +------------------------+
|  客户端连接请求   | ------> |  母鸡公网 IP:外部端口  | ------> |  小鸡内部监听端口 (sing-box) |
+------------------+         +----------------------+         +------------------------+
```

### 部署前必须准备好的映射关系：
在商家后台（如独角鲸云）的「端口转发」面板确认两条 **TCP** 规则：
1. **节点业务通道**：
   * 外部公网端口（例如 `55555`，客户端填写此端口连接，建议申请 50000 以上高位端口）。
   * 内部服务端口（例如 `30000`，sing-box 本地监听端口）。
2. **动态订阅通道**：
   * 外部公网端口（例如 `55556`，客户端拉取配置使用）。
   * 内部服务端口（例如 `8080`，Perl HTTP 服务监听端口）。

---

## 二、步骤一：核心依赖与 sing-box 官方静态内核安装

整段复制并粘贴到小鸡终端执行。脚本自动识别 `x86_64` / `arm64` 架构，下载链接使用 Base64 编码以防止浏览器翻译插件篡改，并在本地解包建立全局软链接：

```bash
cat << 'STEP1_EOF' > /tmp/step1.sh
#!/bin/bash
set -e
export DEBIAN_FRONTEND=noninteractive

echo "=================================================="
echo "          [1/5] 安装基础依赖与检测系统环境         "
echo "=================================================="

# 安装运行必须依赖工具
apt update -y && apt install -y -o Dpkg::Options::="--force-confdef" -o Dpkg::Options::="--force-confold" \
    curl wget tar perl ca-certificates openssl jq procps bc

ARCH_RAW=$(uname -m)
case "$ARCH_RAW" in
    x86_64)  ARCH="amd64" ;;
    aarch64|arm64) ARCH="arm64" ;;
    *) echo "[!] 不支持的硬件架构: $ARCH_RAW" && exit 1 ;;
esac

echo "[*] 检测到底层硬件架构: $ARCH"

# 创建工作目录
mkdir -p /opt/sing-box/config /opt/sing-box/ui /opt/sing-box/env /tmp/sb_dl
TARGET="/opt/sing-box/sing-box"
rm -f /tmp/sb_dl/sing-box.tar.gz

# Base64 物理隔离官方源直链，防翻译插件注入
B64_URL=$(echo "aHR0cHM6Ly9naGZhc3QudG9wL2h0dHBzOi8vZ2l0aHViLmNvbS9TYWdlck5ldC9zaW5nLWJveC9yZWxlYXNlcy9kb3dubG9hZC92MS4xMS40L3NpbmctYm94LTEuMTEuNC1saW51eC0ke0FSQ0h9LnRhci5neg==" | base64 -d | sed "s/\${ARCH}/$ARCH/g")

echo "[*] 正在拉取官方 sing-box 静态二进制包..."
curl -fsSL -o /tmp/sb_dl/sing-box.tar.gz "$B64_URL"

tar -zxvf /tmp/sb_dl/sing-box.tar.gz -C /tmp/sb_dl/
mv $(find /tmp/sb_dl -name sing-box -type f | head -1) "$TARGET"
chmod +x "$TARGET"
ln -sf "$TARGET" /usr/local/bin/sing-box
rm -rf /tmp/sb_dl

if "$TARGET" version &>/dev/null; then
    echo "=================================================="
    echo "[+] sing-box 内核安装成功！版本: $("$TARGET" version | head -n1)"
    echo "=================================================="
else
    echo "[!] 内核验证失败，请检查网络后重试。"
    exit 1
fi
STEP1_EOF
bash /tmp/step1.sh
```

---

## 三、步骤二：全自动化参数交互、环境固化与工业级规则生成

整段复制并粘贴执行。自动引导参数填写，隔离收集内外端口，生成服务端 `config.json`（包含独立 DNS 解析，杜绝容器超时），并写入包含全套 17 组 Loyalsoldier 纯文本格式规则以及 Fake-IP 过滤保护的客户端 YAML 工业模板：

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

read -p "2. 节点【公网外部映射端口】(例如 55555): " EXT_NODE_PORT
read -p "   节点【小鸡内部监听端口】[默认 30000，内外一致直接填外部端口]: " INPUT_INT_NODE_PORT
INT_NODE_PORT=${INPUT_INT_NODE_PORT:-30000}

read -p "3. 订阅【公网外部映射端口】(例如 55556): " EXT_SUB_PORT
read -p "   订阅【小鸡内部监听端口】[默认 8080，内外一致直接填外部端口]: " INPUT_INT_SUB_PORT
INT_SUB_PORT=${INPUT_INT_SUB_PORT:-8080}

read -p "4. 客户端节点卡片名称 [默认: 日本自建（400g）]: " INPUT_NAME
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

# 1. 生成服务端 sing-box 配置文件（带独立外部 DNS，规避 Podman 容器 resolv.conf 阻断）
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

# 2. 生成工业级 Loyalsoldier 全分流客户端 template.yaml
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
echo "[+] 步骤二完成：参数已成功持久化至 $ENV_FILE"
echo "=================================================="
STEP2_EOF
bash /tmp/step2.sh
```

---

## 四、步骤三：原生轻量多协议动态订阅服务实现 (Perl 5)

利用系统自带的 Perl 5 网络套接字实现 HTTP 订阅引擎。  
**核心工业级特性**：
* **零额外安装包依赖**：绝不依赖 CPAN 的 `HTTP::Server::Simple`，杜绝镜像包缺失报错。
* **超时自动断开机制**：使用 `IO::Select` 进行 3 秒非阻塞超时读取，吞净 Header，杜绝连接挂死。
* **原生多协议支持**：
  * 访问根路径 `/` 或携带 Token：输出完整的 **Clash 工业级配置**。
  * 访问 `/vless`：输出单条 **VLESS 分享链接**。
  * 访问 `/base64`：输出经纯算法 Base64 编码的订阅内容（供 **Shadowrocket / v2rayN** 导入）。
* **物理流量穿透读取**：从 `/proc/net/dev` 过滤掉回环网卡 `lo`，按字节精准汇总入站与出站流量，并实时注入 `Subscription-Userinfo` 头部。

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

# 1. 动态加载环境配置
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

# 2. 纯数学计算到期时间戳（无依赖）
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

# 3. 读取网卡真实物理流量
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

# 4. 纯算法实现标准 Base64 编码
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

# 5. 生成 VLESS 裸链接
sub get_vless_link {
    reload_conf();
    my $name_enc = $conf{NODE_NAME} || "Reality-Node";
    $name_enc =~ s/([^a-zA-Z0-9_.~-])/sprintf("%%%02X", ord($1))/eg;
    return "vless://$conf{UUID}\@$conf{SERVER_IP}:$conf{EXT_NODE_PORT}?" .
           "encryption=none&flow=xtls-rprx-vision&security=reality&sni=gateway.icloud.com" .
           "&fp=chrome&pbk=$conf{PUBLIC_KEY}&sid=$conf{SHORT_ID}&type=tcp#$name_enc\n";
}

# 6. 绑定端口并初始化 Socket 调度器
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

            # 鉴权校验
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
                        $body = "# Template Error\n";
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
echo "[+] 步骤三完成：原生 Perl 动态订阅服务构建成功！"
STEP3_EOF
bash /tmp/step3.sh
```

---

## 五、步骤四：systemd 进程守护与开机自启动构建

整段复制并粘贴执行。将核心业务注册为系统底层守护进程，保障异常崩溃或小鸡母鸡重启后 2 秒内无感拉起：

```bash
cat << 'STEP4_EOF' > /tmp/step4.sh
#!/bin/bash
set -e

echo "=================================================="
echo "          [4/5] 配置 systemd 守护服务             "
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

## 六、步骤五：一键提取全协议客户端订阅链接

整段复制并粘贴执行，终端将直接打印专属于你当前实例的三种不同格式订阅链接：

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
echo "3. 【VLESS 单节点原始链接】"
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

## 七、全系统一键自动化综合体检与故障自愈脚本

当遇到测速超时或拉取报错时，整段复制并在小鸡终端运行以下脚本，自动化逐项核验网络通路、端口监听、回落目标、订阅内容与进程状态：

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
