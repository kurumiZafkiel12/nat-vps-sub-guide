# NAT小鸡极简自建节点与动态流量订阅指南（以独角鲸云为例·纯小白避坑版）

> **适用场景**：独角鲸云或其他商家的 NAT VPS、小内存独立 VPS（64MB~512MB 内存）。  
> **核心组合**：官方静态编译 `sing-box` (VLESS-REALITY-Vision) + 原生 `Perl 5` 极轻量动态流量订阅。  
> **四大优势**：  
> 1. **超低内存**：常驻内存仅约 15MB，绝不爆内存死机。  
> 2. **真实扣流量**：直接读取 Linux 内核真实网卡收发字节，按量扣除。  
> 3. **精细分流**：自带 LoyalSoldier 规则集，国内直连、海外代理、AI与流媒体独立分流。  
> 4. **告别阻断**：彻底解决“电脑开着梯子/全局代理更新订阅时报 `failed to fetch remote profile`”的历史难题。

---

## 目录
- [一、从零准备：独角鲸云充值与实例开通（含截图示例）](#一从零准备独角鲸云充值与实例开通含截图示例)
- [二、关键步骤：独角鲸云 NAT 端口映射与参数提取（核心图解）](#二关键步骤独角鲸云-nat-端口映射与参数提取核心图解)
- [三、步骤一：连接小鸡终端与安装基础环境](#三步骤一连接小鸡终端与安装基础环境)
- [四、步骤二：输入你的小鸡专属参数（全流程唯一需改字的地方）](#四步骤二输入你的小鸡专属参数全流程唯一需改字的地方)
- [五、步骤三：自动生成专属 Reality 密钥与凭据](#五步骤三自动生成专属-reality-密钥与凭据)
- [六、步骤四：一键写入并启动 sing-box 节点服务](#六步骤四一键写入并启动-sing-box-节点服务)
- [七、步骤五：生成客户端精细分流配置 (LoyalSoldier)](#七步骤五生成客户端精细分流配置-loyalsoldier)
- [八、步骤六：部署原生 Perl 动态流量订阅守护进程](#八步骤六部署原生-perl-动态流量订阅守护进程)
- [九、步骤七：终端验证与客户端导入（Clash Verge 实操）](#九步骤七终端验证与客户端导入clash-verge-实操)
- [十、核心技巧：开着梯子也能秒级更新订阅（防死锁设置）](#十核心技巧开着梯子也能秒级更新订阅防死锁设置)
- [十一、常用维护命令与一键无损救砖回滚](#十一常用维护命令与一键无损救砖回滚)

---

## 一、从零准备：独角鲸云充值与实例开通（含截图示例）

对于新手来说，购买 NAT 机器最容易在“充值”、“找公网 IP”和“找端口”上卡住。

### 1. 独角鲸云账户充值
1. 打开独角鲸云官网并登录控制台。
2. 点击右上角或侧边栏的 **“财务中心”** -> **“在线充值”**。
3. 选择支付方式（如支付宝），输入充值金额并完成支付。

> **图示占位 1：充值中心界面**  
> ![独角鲸云充值流程示例](images/01-recharge.png)  
> *（截图指引：截取控制台“在线充值”页面，用红框标出“充值金额”与“支付确认”按钮）*

### 2. 选购与部署小鸡实例
1. 点击控制台侧边栏的 **“订购产品” / “云服务器”**。
2. 找到 NAT 套餐（例如：**日本 NAT - 400G 流量** 或 **美国商宽 NAT**）。
3. **系统镜像选择**：强烈推荐选择 **Debian 11 / Debian 12** 或 **Ubuntu 22.04**（这两种系统原生自带 Perl 且非常精简）。
4. 确认订单并开通，等待 1~3 分钟，实例状态变为 **“运行中”**。

---

## 二、关键步骤：独角鲸云 NAT 端口映射与参数提取（核心图解）

NAT 小鸡与独立 IP 机器最大的不同在于：**它没有全部开放的公网端口，所有端口都必须在面板里做端口映射**。

我们需要两个外部端口：
- **节点端口**（跑 VLESS-REALITY 翻墙节点）
- **订阅端口**（跑 Perl 动态流量订阅网页）

### 1. 获取公网 IP 与添加端口映射
1. 进入独角鲸云控制台 -> 点击你的实例进入 **“实例详情”**。
2. 记下顶部的 **公网 IPv4 地址**（例如 `66.154.108.15` 或 `13.143.176.167`，千万不要看内网 `10.x.x.x`）。
3. 找到 **“NAT 端口转发 / 端口映射”** 菜单：
   * **规则 1（给节点用）**：
     * 内网端口：填 `59689`
     * 协议类型：选择 `TCP`
     * 外部端口：系统会自动分配或让你自选（例如分配到了 `59689`）
   * **规则 2（给订阅用）**：
     * 内网端口：填 `59688`
     * 协议类型：选择 `TCP`
     * 外部端口：系统会自动分配或自选（例如分配到了 `59688`）
4. 保存后，记录下对应的公网 IP 和外网端口。

> **图示占位 2：独角鲸云端口映射后台**  
> ![独角鲸云端口映射详情](images/02-nat-ports.png)  
> *（截图指引：截取实例详情里的“NAT 转发”列表，用红框标出公网 IP、外网端口 59689、外网端口 59688）*

---

## 三、步骤一：连接小鸡终端与安装基础环境

使用 SSH 工具（如 FinalShell、Termius 或独角鲸云网页自带的 Web 终端）连接小鸡。

在终端中逐行复制执行以下命令：

```bash
# 1. 更新系统软件包索引并安装 curl、perl、openssl
apt update && apt install -y curl perl openssl

# 2. 检查是否安装了官方静态编译的 sing-box，未安装则自动下载部署
if ! command -v sing-box &> /dev/null; then
    echo "正在下载官方静态 sing-box..."
    ARCH=$(uname -m)
    if [ "$ARCH" = "x86_64" ]; then
        SB_ARCH="amd64"
    elif [ "$ARCH" = "aarch64" ]; then
        SB_ARCH="arm64"
    else
        SB_ARCH="amd64"
    fi
    curl -Lo /usr/local/bin/sing-box [https://github.com/SagerNet/sing-box/releases/download/v1.11.4/sing-box-1.11.4-linux-$](https://github.com/SagerNet/sing-box/releases/download/v1.11.4/sing-box-1.11.4-linux-$){SB_ARCH}.tar.gz
    tar -zxvf /usr/local/bin/sing-box -C /tmp/
    mv /tmp/sing-box-*/sing-box /usr/local/bin/sing-box
    chmod +x /usr/local/bin/sing-box
    rm -rf /tmp/sing-box*
fi

# 3. 验证基础组件版本
sing-box version
perl -v | head -n 2
```

> **查看网卡名称**：在终端运行 `ip -br link`，记下状态为 `UP` 的网卡名（通常为 `eth0`、`ens3`，千万不要填 `lo`）。  
> 
> **图示占位 3：终端网卡查看示意图**  
> ![终端网卡查看示例](images/03-network-iface.png)  
> *（截图指引：在终端输入 `ip -br link`，用红框圈出 `eth0` 所在列）*

---

## 四、步骤二：输入你的小鸡专属参数（全流程唯一需改字的地方）

**无需手动修改复杂的文件代码**！  
只需在本地文本中把下面 7 行改成你在独角鲸云后台拿到的参数，然后整段复制粘贴到小鸡终端中敲回车：

```bash
# =================【用户参数自定义区域】=================
SERVER_IP="66.154.108.15"            # 独角鲸云面板显示的公网 IPv4
NODE_PORT="59689"                    # 映射给节点的外部端口 (VLESS)
SUB_PORT="59688"                     # 映射给订阅的外部端口 (HTTP)
NODE_NAME="日本自建（400g）"         # 客户端显示的卡片与节点名称
TRAFFIC_GB=400                       # 总流量配额 (单位: GB，纯数字)
SUB_TOKEN="jp_token_8899"            # 自定义防扫 Token (随手写字母数字)
NET_IFACE="eth0"                     # 步骤一查出的真实网卡名 (通常是 eth0)
# ========================================================

# 自动计算总字节数与到期时间戳 (默认设为 30 天后到期)
TOTAL_BYTES=$(( TRAFFIC_GB * 1024 * 1024 * 1024 ))
EXPIRE_TIME=$(( $(date +%s) + 30 * 86400 ))

# 建立专属目录与备份目录
mkdir -p /opt/sing-box/ui /opt/sing-box/backup
```

---

## 五、步骤三：自动生成专属 Reality 密钥与凭据

直接整段复制执行以下代码。它会自动生成专属的 UUID、REALITY 密钥对及 Short ID，并把它们**暂存在系统环境变量中，供后面步骤自动填充，完全不需要手动复制长串字符**：

```bash
# 1. 自动生成专属客户端凭据 UUID
UUID=$(sing-box generate uuid)

# 2. 自动生成专属 REALITY 密钥对
KEYPAIR=$(sing-box generate reality-keypair)
PRIVATE_KEY=$(echo "$KEYPAIR" | grep "PrivateKey" | awk '{print $2}')
PUBLIC_KEY=$(echo "$KEYPAIR" | grep "PublicKey" | awk '{print $2}')

# 3. 自动生成 8 字节十六进制短标识符 Short ID
SHORT_ID=$(openssl rand -hex 8)

# 打印在屏幕上给你看
echo "================ 你的专属安全凭证 ================"
echo "UUID        : ${UUID}"
echo "PrivateKey  : ${PRIVATE_KEY}"
echo "PublicKey   : ${PUBLIC_KEY}"
echo "Short ID    : ${SHORT_ID}"
echo "=================================================="
```

---

## 六、步骤四：一键写入并启动 sing-box 节点服务

直接整段复制并粘贴到终端回车执行。脚本会自动把你前面生成的 UUID、私钥、Short ID 和端口拼装并启动服务：

```bash
cat <<EOF> /opt/sing-box/config.json
{
  "log": {
    "level": "warn"
  },
  "inbounds": [
    {
      "type": "vless",
      "tag": "vless-in",
      "listen": "::",
      "listen_port": ${NODE_PORT},
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

# 写入 systemd 守护进程
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

# 重载并启动 sing-box
systemctl daemon-reload
systemctl enable --now sing-box
systemctl restart sing-box

# 验证状态（应显示 active running）
systemctl status sing-box --no-pager
```

---

## 七、步骤五：生成客户端精细分流配置 (LoyalSoldier)

此步骤生成客户端拉取时接收的 YAML。预置了 LoyalSoldier 官方规则集，包含国内直连、国外代理、微软苹果 CDN 直连、AI 平台独立分流：

```bash
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
EOF
```

---

## 八、步骤六：部署原生 Perl 动态流量订阅守护进程

纯原生 Perl 5 守护进程，常驻内存仅约 1.5MB：
- 实时读取 Linux 系统内核 `/proc/net/dev` 真实扣除流量。
- 自带 Token 模糊匹配，防止开着全局代理时请求路径被改写导致断开。

复制并粘贴执行：

```bash
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

# 写入 systemd 守护配置
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

# 启动订阅服务
systemctl daemon-reload
systemctl enable --now clash-sub
systemctl restart clash-sub

# 检查运行状态
systemctl status clash-sub --no-pager
```

---

## 九、步骤七：终端验证与客户端导入（Clash Verge 实操）

### 1. 服务端本地可用性验证
在 VPS 终端执行以下命令：

```bash
# 验证端口监听
ss -tulpn | grep -E "(${NODE_PORT}|${SUB_PORT})"

# 本地抓取测试 (看是否有正常 HTTP/1.1 200 OK 返回)
curl -i "[http://127.0.0.1](http://127.0.0.1):${SUB_PORT}/token=${SUB_TOKEN}" | head -n 8

echo ""
echo "=========================================================="
echo "搭建成功！你的专属客户端导入链接为："
echo "http://${SERVER_IP}:${SUB_PORT}/token=${SUB_TOKEN}&name=${NODE_NAME}.yaml"
echo "=========================================================="
```

### 2. 在 Clash Verge 中一键导入
1. 打开 Clash Verge，点击左侧 **“订阅 (Profiles)”**。
2. 在顶部的输入框中，直接粘贴上面终端打印出来的链接：
   ```text
   [http://66.154.108.15:59688/token=jp_token_8899&name=日本自建](http://66.154.108.15:59688/token=jp_token_8899&name=日本自建)（400g）.yaml
   ```
3. 点击 **“导入 (Import)”**。

> **图示占位 4：Clash Verge 导入与成功展示**  
> ![Clash Verge 订阅导入效果](images/04-clash-import.png)  
> *（截图指引：截取 Clash Verge 订阅卡片，红框标出卡片标题“日本自建（400g）”和下方的流量进度条）*

---

## 十、核心技巧：开着梯子也能秒级更新订阅（防死锁设置）

**小白常踩的最大坑**：如果电脑当前正开着其他商业梯子/全局 VPN，点击更新订阅时，可能会报错 `failed to fetch remote profile`。这是因为中间梯子屏蔽了小鸡的高位端口（如 `59688`）。

### 一劳永逸解法（无需每次关闭梯子）：
1. 在 Clash Verge 中找到你平时主力使用的代理订阅卡片，右键选择 **“编辑扩展配置 (Edit Rules / Script)”**。
2. 在规则列表（`rules:`）的最顶部添加一行公网 IP 直连规则：
   ```yaml
   rules:
     - IP-CIDR,你的小鸡公网IP/32,DIRECT,no-resolve
   ```
3. 保存并刷新。

> **图示占位 5：Clash Verge 直连规则添加位置**  
> ![Clash Verge 直连分流配置](images/05-clash-direct-rule.png)  
> *（截图指引：截取配置编辑窗口，红框标出添加在最顶部的 `IP-CIDR` 直连规则行）*

**生效效果**：  
当你开着系统代理更新该订阅时，流量会自动绕过梯子，走你本机的本地宽带直连出站，彻底避开高位端口拦截，实现秒拉取！

---

## 十一、常用维护命令与一键无损救砖回滚

### 1. 建立初始工作状态备份
部署完成后，在 VPS 执行一次备份：
```bash
mkdir -p /opt/sing-box/backup
cp -a /opt/sing-box/config.json /opt/sing-box/backup/config.json.bak
cp -a /opt/sing-box/sub.pl /opt/sing-box/backup/sub.pl.bak
cp -a /opt/sing-box/ui/index.html /opt/sing-box/backup/index.html.bak
```

### 2. 误操作一键无损救砖还原
如果以后改坏了任何地方导致报错，执行这一段立刻满血复活：
```bash
cp /opt/sing-box/backup/config.json.bak /opt/sing-box/config.json
cp /opt/sing-box/backup/sub.pl.bak /opt/sing-box/sub.pl
cp /opt/sing-box/backup/index.html.bak /opt/sing-box/ui/index.html
systemctl restart sing-box clash-sub
```
