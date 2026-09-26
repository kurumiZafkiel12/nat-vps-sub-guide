# 全通用 NAT VPS 极简节点与动态流量订阅部署指南


> 本项目用于在 NAT VPS 环境中快速部署个人节点服务。

> 本教程以 **独角鲸云 NAT VPS** 为例。

> 支持：

- 独角鲸云
- 其它 NAT VPS
- Debian 系统 VPS


核心组件：


sing-box

VLESS Reality Vision

自动订阅服务

Clash Meta



项目目标：


购买 NAT VPS

↓

执行一条命令

↓

自动检测环境

↓

自动生成节点

↓

自动生成订阅

↓

客户端直接使用



---

# 一、NAT VPS简介


## 1.1 什么是 NAT VPS


普通 VPS：

拥有独立公网 IPv4。


结构：


公网IP

|

服务器

|

服务端口



例如：


1.2.3.4:443



客户端直接访问服务器。


---


NAT VPS：

多个用户共享公网 IPv4。

通过 NAT 网关进行端口映射。


结构：


共享公网IP

    |

    |

NAT网关

    |

    |

你的VPS实例

    |

    |

内部监听端口



例如独角鲸云：


公网端口:

59688

    ↓

内部端口:

30000



sing-box 实际监听：


30000



用户访问：


公网IP:59688



NAT自动转发：


59688

↓

30000



---

## 1.2 NAT VPS部署特点


优点：

- 价格低
- 资源占用小
- 适合个人节点
- 适合长期运行


注意：

NAT VPS 与普通 VPS 最大区别：

公网端口 ≠ VPS监听端口。


因此部署时：

需要区分：


公网映射端口

↓

内部服务端口



---

# 1.3 本项目自动化设计


传统部署：


填写IP

填写UUID

填写密钥

修改配置

启动服务

生成订阅



容易出错。


本项目改为：



脚本启动

↓

自动检测 Debian

↓

自动检测架构

↓

自动获取公网IP

↓

自动生成UUID

↓

自动生成Reality密钥

↓

自动生成配置

↓

自动启动服务



用户只需要确认 NAT 映射端口。


---

# 1.4 支持环境


推荐：


Debian 12



支持：


Debian 11+



自动检测：


系统版本

CPU架构

网络环境

systemd状态



---

# 二、独角鲸云 NAT VPS准备与端口映射


## 2.1 创建 NAT VPS 实例


本教程以独角鲸云为例。


进入控制台：


实例管理


选择：


新建实例



根据需求选择：

- 地区
- 套餐
- 系统


推荐：


Debian 12



![新建实例与选区](https://github.com/user-attachments/assets/1fd2cc42-c06c-4f62-9197-cea3af093c75)



---


## 2.2 登录 VPS


实例创建完成后，保存：


SSH地址

SSH端口

用户名

密码



后续部署全部通过 SSH 或 Web Terminal 完成。


![Web控制台入口](https://github.com/user-attachments/assets/c1ab8281-ca61-4fc4-99b2-253d83deb611)



---


# 2.3 NAT端口映射


NAT VPS必须配置端口转发。


独角鲸云示例：



公网端口:

59688

↓

内部端口:

30000



说明：


公网端口：

用于客户端连接。


内部端口：

用于 VPS 内部程序监听。



结构：



Clash客户端

    |

    |

公网IP:59688

    |

    |

NAT网关

    |

    |

VPS:30000

    |

    |

sing-box



---


![端口转发规则](https://github.com/user-attachments/assets/c364fe94-c038-4c5d-8936-6629a70d8c75)



---


# 2.4 部署前需要准备的信息


新版自动部署不会要求填写大量参数。


脚本自动获取：

|项目|方式|
|-|-|
|公网IP|自动检测|
|系统版本|自动检测|
|CPU架构|自动检测|
|网卡|自动检测|
|UUID|自动生成|
|Reality密钥|自动生成|


用户只需要确认：


## 节点端口


例如：



公网:

59688

内部:

30000



---


## 订阅端口


建议额外映射一个端口。


例如：



公网:

59689

内部:

8080



用途：


59688

节点连接

59689

订阅访问



---


# 2.5 部署前检测


登录 VPS 后执行：


```bash
curl -fsSL https://你的地址/check.sh | bash

脚本自动显示：

========================

环境检测

========================


系统:

Debian 12


架构:

x86_64


公网IP:

xxx.xxx.xxx.xxx


网卡:

eth0


systemd:

支持


========================

用户确认：

是否继续部署？

[Y/n]
2.6 NAT端口填写原则

由于 NAT 商家的公网映射信息：

VPS 内部无法读取。

所以：

只有端口映射需要确认。

例如：

脚本询问：

请输入节点内部监听端口:

默认 30000:

用户：

直接回车。

脚本生成：

sing-box监听:

30000

然后用户在独角鲸云设置：

公网59688

↓

内部30000

# 三、NAT VPS 一键自动部署脚本（核心）


## 3.1 创建 install.sh


复制：


```bash
nano install.sh

填入：

#!/bin/bash

set -e


# ==========================
# NAT VPS 自动部署脚本
# Debian 11/12
# ==========================


GREEN="\033[32m"
RESET="\033[0m"


echo "=========================="
echo " NAT VPS 自动部署"
echo "=========================="


# --------------------------
# 检测系统
# --------------------------

if [ -f /etc/os-release ]; then

    source /etc/os-release

    echo -e "${GREEN}系统:${RESET} $PRETTY_NAME"

else

    echo "无法检测系统"

    exit 1

fi



# --------------------------
# 检测架构
# --------------------------


ARCH=$(uname -m)


case "$ARCH" in

x86_64)

    SB_ARCH="amd64"

    ;;


aarch64|arm64)

    SB_ARCH="arm64"

    ;;


*)

    echo "不支持架构: $ARCH"

    exit 1

    ;;

esac



echo "架构: $SB_ARCH"



# --------------------------
# 获取公网IP
# --------------------------


PUBLIC_IP=$(curl -4 -s https://api.ipify.org)


if [ -z "$PUBLIC_IP" ]; then

    PUBLIC_IP="未知"

fi


echo "公网IP: $PUBLIC_IP"



read -p "是否使用此IP? [Y/n]:" confirm



# --------------------------
# 输入NAT内部端口
# --------------------------


read -p \
"节点内部监听端口 (默认30000): " NODE_PORT


NODE_PORT=${NODE_PORT:-30000}



read -p \
"订阅内部监听端口 (默认8080): " SUB_PORT


SUB_PORT=${SUB_PORT:-8080}



# --------------------------
# 安装依赖
# --------------------------


apt update -y


apt install -y \
curl \
wget \
jq \
openssl \
tar \
perl \
ca-certificates



# --------------------------
# 安装sing-box
# --------------------------


mkdir -p /opt/sing-box


VERSION=$(curl -s \
https://api.github.com/repos/SagerNet/sing-box/releases/latest \
| jq -r '.tag_name')



wget -q \
-O /tmp/singbox.tar.gz \
"https://github.com/SagerNet/sing-box/releases/download/${VERSION}/sing-box-${VERSION#v}-linux-${SB_ARCH}.tar.gz"



mkdir -p /tmp/singbox


tar xf /tmp/singbox.tar.gz \
-C /tmp/singbox



cp $(find /tmp/singbox -name sing-box -type f | head -1) \
/opt/sing-box/sing-box



chmod +x /opt/sing-box/sing-box



# --------------------------
# 生成参数
# --------------------------


UUID=$(
/opt/sing-box/sing-box generate uuid
)



REALITY=$(
/opt/sing-box/sing-box generate reality-keypair
)



PRIVATE_KEY=$(echo "$REALITY" \
| grep PrivateKey \
| awk '{print $2}')



PUBLIC_KEY=$(echo "$REALITY" \
| grep PublicKey \
| awk '{print $2}')



SHORT_ID=$(openssl rand -hex 8)



echo "UUID=$UUID" \
>/opt/sing-box/info.txt


echo "PUBLIC_KEY=$PUBLIC_KEY" \
>>/opt/sing-box/info.txt


echo "PRIVATE_KEY=$PRIVATE_KEY" \
>>/opt/sing-box/info.txt


echo "SHORT_ID=$SHORT_ID" \
>>/opt/sing-box/info.txt



# --------------------------
# 生成配置
# --------------------------


mkdir -p /opt/sing-box/config



cat > /opt/sing-box/config/config.json <<EOF
{
"log":{
"level":"info"
},

"inbounds":[
{

"type":"vless",

"listen":"::",

"listen_port":$NODE_PORT,


"users":[
{
"uuid":"$UUID",
"flow":"xtls-rprx-vision"
}
],


"tls":{

"enabled":true,


"server_name":"www.apple.com",


"reality":{

"enabled":true,


"handshake":{

"server":"www.apple.com",

"server_port":443

},


"private_key":"$PRIVATE_KEY",


"short_id":[

"$SHORT_ID"

]

}

}

}

],


"outbounds":[
{
"type":"direct"
}
]

}
EOF



echo ""
echo "=========================="
echo "部署完成"
echo "=========================="

echo "节点端口:"
echo "$NODE_PORT"

echo ""

echo "订阅端口:"
echo "$SUB_PORT"

echo ""

echo "参数保存:"
echo "/opt/sing-box/info.txt"


保存：

chmod +x install.sh

运行：

./install.sh

# 四、自动订阅系统


## 4.1 创建订阅服务


安装依赖：


```bash
apt install -y perl

创建目录：

mkdir -p /opt/sing-box/sub
4.2 创建订阅服务器

创建文件：

nano /opt/sing-box/sub/server.pl

写入：

#!/usr/bin/perl


use strict;
use warnings;

use HTTP::Server::Simple::CGI;


# ==========================
# NAT VPS订阅服务
# ==========================


my $PORT = 8080;


my $INFO = "/opt/sing-box/info.txt";



# --------------------------
# 读取节点参数
# --------------------------


sub read_info {


    open(my $fh,"<",$INFO)
    or die "无法读取节点信息";


    my %data;


    while(<$fh>){

        chomp;


        my ($k,$v)=split("=",$_,2);


        $data{$k}=$v;

    }


    close($fh);


    return %data;

}




# --------------------------
# 生成Clash配置
# --------------------------


sub generate_yaml {


    my %info=read_info();


    my $uuid=$info{"UUID"};

    my $public=$ENV{"PUBLIC_IP"};

    my $port=$ENV{"NODE_PUBLIC_PORT"};



return <<"EOF";

port: 7890

socks-port: 7891


allow-lan: false


mode: rule



proxies:


- name: NAT-VPS-Reality


  type: vless


  server: $public


  port: $port


  uuid: $uuid


  network: tcp


  udp: true


  tls: true


  flow: xtls-rprx-vision


  servername: www.apple.com


  reality-opts:


    public-key: $info{"PUBLIC_KEY"}


    short-id: $info{"SHORT_ID"}


EOF

}



# --------------------------
# HTTP服务
# --------------------------


package MyServer;


use base qw(HTTP::Server::Simple::CGI);



sub handle_request {


my ($self,$cgi)=@_;


print "HTTP/1.0 200 OK\r\n";

print "Content-Type:text/yaml\r\n\r\n";


print main::generate_yaml();



}



package main;


my $server=MyServer->new($PORT);


$server->run();


保存。

授权：

chmod +x /opt/sing-box/sub/server.pl
4.3 测试订阅服务

运行：

perl /opt/sing-box/sub/server.pl

另开窗口测试：

curl http://127.0.0.1:8080

返回：

proxies:

- name: NAT-VPS-Reality

  type: vless

说明订阅正常。

4.4 设置公网参数

因为 NAT VPS：

公网端口由商家映射。

所以启动订阅时设置变量：

例如独角鲸云：

公网IP:

1.2.3.4


节点公网端口:

59688

执行：

export PUBLIC_IP=1.2.3.4

export NODE_PUBLIC_PORT=59688

然后启动：

perl /opt/sing-box/sub/server.pl
4.5 订阅地址

如果：

公网订阅端口:

59689


内部:

8080

Clash 使用：

http://公网IP:59689

例如：

http://1.2.3.4:59689

# 五、systemd 自动启动与服务管理


## 5.1 保存部署变量


由于 NAT VPS 公网端口由商家映射，需要保存变量。


创建环境文件：


```bash
mkdir -p /opt/sing-box/env

创建：

nano /opt/sing-box/env/node.env

填写：

PUBLIC_IP=你的公网IP

NODE_PUBLIC_PORT=59688

SUB_PUBLIC_PORT=59689

示例：

PUBLIC_IP=1.2.3.4

NODE_PUBLIC_PORT=59688

SUB_PUBLIC_PORT=59689

保存。

5.2 创建 sing-box systemd 服务

创建：

nano /etc/systemd/system/sing-box.service

写入：

[Unit]

Description=NAT VPS sing-box Reality Node

After=network.target



[Service]

Type=simple


EnvironmentFile=/opt/sing-box/env/node.env


ExecStart=/opt/sing-box/sing-box run \
-c /opt/sing-box/config/config.json



Restart=always


RestartSec=5



LimitNOFILE=65535



[Install]

WantedBy=multi-user.target
5.3 创建订阅 systemd 服务

创建：

nano /etc/systemd/system/subscription.service

写入：

[Unit]

Description=NAT VPS Subscription Service

After=network.target



[Service]

Type=simple


EnvironmentFile=/opt/sing-box/env/node.env


ExecStart=/usr/bin/perl \
/opt/sing-box/sub/server.pl



Restart=always


RestartSec=5



[Install]

WantedBy=multi-user.target
5.4 修改订阅服务读取环境变量

编辑：

nano /opt/sing-box/sub/server.pl

找到：

my $public=$ENV{"PUBLIC_IP"};

my $port=$ENV{"NODE_PUBLIC_PORT"};

保持即可。

systemd 会自动传入变量。

5.5 启动服务

重新加载：

systemctl daemon-reload

启动 sing-box：

systemctl enable sing-box

systemctl restart sing-box

启动订阅：

systemctl enable subscription

systemctl restart subscription
5.6 查看运行状态

查看节点：

systemctl status sing-box

正常：

active (running)

查看订阅：

systemctl status subscription

正常：

active (running)
5.7 查看日志

sing-box日志：

journalctl -u sing-box -f

订阅日志：

journalctl -u subscription -f
5.8 自动恢复测试

停止服务：

systemctl stop sing-box

等待几秒。

查看：

systemctl status sing-box

因为：

Restart=always

服务会自动恢复。

5.9 最终服务结构

部署完成后：

Debian NAT VPS


        |

        |


systemd


        |

        +-----------+

        |           |

   sing-box    subscription


        |           |

        |           |

 VLESS Reality   Clash YAML

# 六、自动检测与故障修复脚本


## 6.1 创建检测脚本


创建文件：


```bash
nano /opt/sing-box/check.sh

写入：

#!/bin/bash


# ==========================
# NAT VPS 自动检测脚本
# ==========================


echo "================================"

echo " NAT VPS 节点检测"

echo "================================"



# --------------------------

# 检测系统

# --------------------------


echo ""

echo "[1] 系统信息"


cat /etc/os-release | grep PRETTY_NAME



echo ""

echo "架构:"

uname -m



# --------------------------

# 检测IP

# --------------------------


echo ""

echo "[2] 网络信息"



PUBLIC_IP=$(curl -4 -s https://api.ipify.org)



echo "公网IP:"

echo "$PUBLIC_IP"



# --------------------------

# 检测sing-box

# --------------------------


echo ""

echo "[3] sing-box状态"



if systemctl is-active --quiet sing-box

then

echo "✓ sing-box运行正常"

else

echo "✗ sing-box异常"

echo "查看日志:"

echo "journalctl -u sing-box"

fi



# --------------------------

# 检测订阅服务

# --------------------------


echo ""

echo "[4] 订阅服务"



if systemctl is-active --quiet subscription

then

echo "✓ subscription运行正常"

else

echo "✗ subscription异常"

echo "查看日志:"

echo "journalctl -u subscription"

fi



# --------------------------

# 检测端口

# --------------------------


echo ""

echo "[5] 监听端口"



ss -tlnp | grep -E "sing-box|perl"



# --------------------------

# 配置检测

# --------------------------


echo ""

echo "[6] sing-box配置"



/opt/sing-box/sing-box check \
-c /opt/sing-box/config/config.json



# --------------------------

# 订阅测试

# --------------------------


echo ""

echo "[7] 订阅测试"



curl -s \
http://127.0.0.1:8080 \
| head



echo ""

echo "================================"

echo "检测完成"

echo "================================"


保存。

授权：

chmod +x /opt/sing-box/check.sh
6.2 使用检测脚本

执行：

/opt/sing-box/check.sh

正常显示：

================================

 NAT VPS 节点检测

================================


系统:

Debian 12


架构:

x86_64


公网IP:

xxx.xxx.xxx.xxx


✓ sing-box运行正常


✓ subscription运行正常


LISTEN:

30000 sing-box

8080 perl


configuration is valid


检测完成
6.3 常见错误自动定位
sing-box启动失败

查看：

journalctl -u sing-box -n 50

常见原因：

配置错误

修复：

sing-box check \
-c /opt/sing-box/config/config.json
端口无法连接

检查内部监听：

ss -tlnp

应该看到：

30000 sing-box

8080 perl

如果没有：

重启：

systemctl restart sing-box

systemctl restart subscription
NAT端口不通

检查：

内部：

curl 127.0.0.1:8080

正常返回：

proxies:

如果正常：

说明 VPS 内服务没问题。

检查独角鲸云：

公网端口

↓

内部端口

是否一致。

6.4 一键修复脚本

创建：

nano /opt/sing-box/fix.sh

写入：

#!/bin/bash


echo "正在修复服务..."


systemctl restart sing-box


systemctl restart subscription


sleep 3


systemctl status sing-box --no-pager


systemctl status subscription --no-pager


echo "修复完成"


授权：

chmod +x /opt/sing-box/fix.sh

以后出现问题：

/opt/sing-box/fix.sh

即可。

# 七、客户端导入与最终使用


## 7.1 获取订阅地址


部署完成后：

脚本会保存节点信息：

/opt/sing-box/info.txt



查看：


```bash
cat /opt/sing-box/info.txt

示例：

UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxx

PUBLIC_KEY=xxxxxxxx

PRIVATE_KEY=xxxxxxxx

SHORT_ID=xxxxxxxx

7.2 生成最终订阅地址

NAT VPS 示例：

节点映射：

公网59688

↓

内部30000

订阅映射：

公网59689

↓

内部8080

最终订阅：

http://你的公网IP:59689

例如：

http://1.2.3.4:59689

注意：

不要填写内部端口：

错误：

http://1.2.3.4:8080

正确：

http://1.2.3.4:59689

7.3 Clash Verge 导入

打开：

Clash Verge

进入：

订阅管理

↓

新建订阅

填写：

http://你的公网IP:59689

保存。

等待下载配置。
7.4 检查节点参数

正常节点应该显示：

名称:

NAT-VPS-Reality


类型:

VLESS


传输:

TCP


TLS:

Reality


流控:

xtls-rprx-vision

服务器：

公网IP

端口：

公网映射端口

例如：

59688

7.5 连接测试

启动 Clash Verge。

选择：

NAT-VPS-Reality

测试：

curl -I https://www.google.com

或者访问：

测速网站

如果正常：

说明：

NAT端口映射

↓

sing-box

↓

Reality握手

↓

客户端代理

全部正常。
7.6 最终部署结构

完成后 VPS：

/opt/sing-box


├── sing-box

│

├── config

│   └── config.json

│

├── sub

│   └── server.pl

│

├── env

│   └── node.env

│

├── scripts

│   └── check.sh

│

└── info.txt

系统服务：

systemd


├── sing-box.service

│

└── subscription.service

7.7 日常维护命令

查看节点状态：

systemctl status sing-box

查看订阅状态：

systemctl status subscription

重启节点：

systemctl restart sing-box

重启订阅：

systemctl restart subscription

查看日志：

journalctl -u sing-box -f

7.8 一键检测

以后出现问题：

执行：

/opt/sing-box/check.sh

自动检测：

系统

网络

公网IP

sing-box

端口

配置

订阅

部署完成

现在你的 NAT VPS 已实现：

独角鲸云 NAT VPS

↓

自动检测 Debian

↓

自动安装 sing-box

↓

自动生成 Reality

↓

自动生成 Clash订阅

↓

systemd守护

↓

自动检测修复

↓

客户端使用

# 八、动态流量统计与 Subscription-Userinfo


## 8.1 功能说明


普通订阅：


返回节点配置



增强订阅：


节点配置

流量信息

到期时间



Clash Meta 支持：


Subscription-Userinfo



显示：


Upload

Download

Total

Expire



---


# 8.2 创建流量记录文件


创建：


```bash
mkdir -p /opt/sing-box/traffic

创建用户数据：

nano /opt/sing-box/traffic/user.conf

写入：

# 总流量

TOTAL=107374182400


# 到期时间

EXPIRE=1798761600


# 初始上传

UPLOAD=0


# 初始下载

DOWNLOAD=0

说明：

TOTAL

单位:

Byte

例如：

100GB：

107374182400
8.3 创建流量统计脚本

创建：

nano /opt/sing-box/traffic/check.sh

写入：

#!/bin/bash


source /opt/sing-box/traffic/user.conf



# 获取默认网卡

NIC=$(ip route | awk '/default/ {print $5}' | head -1)



# 获取流量


DOWNLOAD=$(cat /sys/class/net/$NIC/statistics/rx_bytes)



UPLOAD=$(cat /sys/class/net/$NIC/statistics/tx_bytes)



# 保存


sed -i \
"s/^UPLOAD=.*/UPLOAD=$UPLOAD/" \
/opt/sing-box/traffic/user.conf



sed -i \
"s/^DOWNLOAD=.*/DOWNLOAD=$DOWNLOAD/" \
/opt/sing-box/traffic/user.conf



授权：

chmod +x /opt/sing-box/traffic/check.sh
8.4 修改订阅服务器

编辑：

nano /opt/sing-box/sub/server.pl

增加：

sub userinfo {


open(my $fh,
"<",
"/opt/sing-box/traffic/user.conf")
or return "";



my %data;


while(<$fh>){

chomp;


my($k,$v)=split("=",$_,2);


$data{$k}=$v;


}


close($fh);



return

"upload=".$data{"UPLOAD"}.

"; download=".$data{"DOWNLOAD"}.

"; total=".$data{"TOTAL"}.

"; expire=".$data{"EXPIRE"};

}
8.5 添加订阅响应头

找到：

print "Content-Type:text/yaml\r\n\r\n";

替换为：

print "Content-Type:text/yaml\r\n";


print "Subscription-Userinfo: ";

print userinfo();


print "\r\n\r\n";

保存。

8.6 重启订阅服务

执行：

systemctl restart subscription
8.7 测试

执行：

curl -I http://127.0.0.1:8080

返回：

Subscription-Userinfo:

upload=123456;

download=789012;

total=107374182400;

expire=1798761600

说明成功。

8.8 Clash Verge显示

重新更新订阅。

节点页面显示：

上传:

xxx MB


下载:

xxx GB


总量:

100GB


到期:

日期

# 九、多协议订阅生成


## 9.1 功能说明


目前订阅服务器返回：


Clash YAML



增加：



VLESS链接

Base64订阅

Clash订阅



结构：


节点参数

↓

生成多个格式

↓

用户选择客户端



---


# 9.2 创建订阅生成目录


执行：


```bash
mkdir -p /opt/sing-box/sub/template
9.3 创建 VLESS 生成脚本

创建：

nano /opt/sing-box/sub/template/vless.sh

写入：

#!/bin/bash


source /opt/sing-box/info.txt


source /opt/sing-box/env/node.env



NODE_NAME="NAT-VPS-Reality"



LINK="vless://${UUID}@${PUBLIC_IP}:${NODE_PUBLIC_PORT}"


LINK+="?encryption=none"


LINK+="&security=reality"


LINK+="&sni=www.apple.com"


LINK+="&fp=chrome"


LINK+="&pbk=${PUBLIC_KEY}"


LINK+="&sid=${SHORT_ID}"


LINK+="&type=tcp"


LINK+="&flow=xtls-rprx-vision"



LINK+="#${NODE_NAME}"



echo "$LINK"

授权：

chmod +x /opt/sing-box/sub/template/vless.sh
9.4 测试 VLESS链接

执行：

/opt/sing-box/sub/template/vless.sh

输出：

vless://xxxxxxxx@IP:59688?... 

即可导入：

v2rayN
Nekoray
Shadowrocket
9.5 创建Base64订阅

创建：

nano /opt/sing-box/sub/template/base64.sh

写入：

#!/bin/bash


/opt/sing-box/sub/template/vless.sh \
| base64 -w 0

授权：

chmod +x /opt/sing-box/sub/template/base64.sh
9.6 修改订阅服务器支持格式

编辑：

nano /opt/sing-box/sub/server.pl

增加：

sub vless {


return `/opt/sing-box/sub/template/vless.sh`;

}



sub base64 {


return `/opt/sing-box/sub/template/base64.sh`;

}
9.7 增加访问路径

修改：

找到：

sub handle_request

替换逻辑：

my $path=$cgi->path_info();



if($path eq "/vless"){

print vless();

return;

}



if($path eq "/base64"){

print base64();

return;

}



print generate_yaml();
9.8 最终订阅地址
Clash
http://公网IP:59689
VLESS
http://公网IP:59689/vless

返回：

vless://xxxx
Base64
http://公网IP:59689/base64

适用于：

Shadowrocket
v2ray客户端
9.9 重启服务

执行：

systemctl restart subscription

测试：

curl http://127.0.0.1:8080/vless

正常返回：

vless://
9.10 当前架构

现在你的 NAT VPS 已经支持：

                    NAT VPS


                        |

                  sing-box


                        |

              +---------+---------+

              |                   |

        Clash订阅             VLESS链接


              |

        Base64订阅


              |

    Clash / v2rayN / Shadowrocket
