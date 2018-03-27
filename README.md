# 科学上网的正确姿势

## 前期准备

需要购买一台拥有 root 权限的 VPS 。

服务器购买后，安装 CentOS7，因为以下教程都是基于 CentOS7 的。
建议安装 Centos 7 x86_64 的版本,安装新的 OS 后，搬瓦工会告诉你 SSH 的端口和 root 的密码，这些是无法自定义的，要记住了。如果忘记了 root 密码，可以使用搬瓦工提供的在线SSH登录来重置 root 密码。我是直接使用 ssh 登录来配置 VPS ，Mac 下直接使用终端就好，win 下自行寻找一个 ssh 工具就好。

登录 ssh 的命令:

`$ ssh -p vps 端口号 root@vpsIP 地址`

## 配置防火墙


如果 SSH 无法登录，那说明防火墙关闭了 SSH 端口，需要通过在线 SSH 登录进去关闭防火墙重新配置

清除防火墙配置

`$ iptables -F`

清除 iptabels 所有表项，同时nat设置也没了，但是我们后续的脚本里会配置的，不用担心。如果 SSH 登录正常就不用管防火墙。
P.S. 这一步很重要,所以务必执行此命令。

安装 firewalld

`$ yum install firewalld firewall-config`

`$ systemctl start firewalld`

P.S. 我在安装完 firewalld 之后然后启动服务的时候一直显示失败，然后重启了一遍服务器就可以正常的启动 firewalld 服务了，
有类似情况的朋友可以重启一下服务器

修改 SSH 端口
$ vi /usr/lib/firewalld/services/ssh.xml
会出现一下的内容
```
<?xml version="1.0" encoding="utf-8"?>
<service>
   <short>SSH</short>
   <description>Secure Shell (SSH) is a protocol for logging into and executing commands on remote machines. It provides secure encrypted communications. If you plan on accessing your machine remotely via SSH over a firewalled interface， enable this option. You need the openssh-server package installed for this option to be useful.</description>
   <port protocol="tcp" port="22"/>
</service>
```
将 port="22"，修改成搬瓦工提供给你的端口号，然后 reload firewalld 就 OK

vi 的命令: 按 "i" 是编辑模式，编辑后按 "esc" 退出编辑模式，然后按 Shift 输入 ":wq" 保存退出vi.

`$ firewall-cmd --permanent --add-service=ssh`

`$ firewall-cmd --reload`

OK，现在准备工作都已就绪，安装了源，安装配置了防火墙，下一步开始搭建服务器了。
P.S. 如果使用 firewall-cmd 命令显示 FirewallD is not running ,请重启服务器可以解决此问题。

## 搭建 Shadowsocks 服务
Shadowsocks 感觉是目前最稳定，简单，方便的搭建 VPN 的服务

安装组件

`$ yum install m2crypto python-setuptools`

`$ easy_install pip`

`$ pip install shadowsocks`

安装时部分组件需要输入 Y 确认。小内存 VPS 可以分别安装组件。

安装完成后配置服务器参数

`$ vi  /etc/shadowsocks.json`

写入如下配置:
```
{
    "server":"0.0.0.0",
    "server_port":443,
    "local_address": "127.0.0.1",
    "local_port":1080,
    "password":"mypassword",
    "timeout":300,
    "method":"aes-256-cfb",
    "fast_open": false,
    "workers": 1
}
```
将上面的 mypassword 替换成你的密码， server_port 也是可以修改的，例如 443 是 Shadowsocks 客户端默认的端口号

如果需要修改端口，需要在防火墙里打开响应的端口，用 firewalld 操作就比较简单了

`$ vi /usr/lib/firewalld/services/ss.xml`

下面代码粘贴到里面
```
<?xml version="1.0" encoding="utf-8"?>
   <service>
     <short>SS</short>
     <description>Shadowsocks port
     </description>
     <port protocol="tcp" port="443"/>
   </service>
  ```

port 可自定义,但是需要跟上面的 server_port 对应起来,保存退出,然后重启 firewalld 服务

`$ firewall-cmd --permanent --add-service=ss`

`$ firewall-cmd --reload`

运行命令，启动 Shadowsocks 服务
运行下面的命令

`$ ssserver -c /etc/shadowsocks.json`

至此shadowsocks搭建完成，shadowsocks已经可以使用，如果你没有过高的要求，下面的步骤可以省略，下面是后台运行 Shadowsocks 的步骤。

安装 supervisor 实现后台运行
运行一下命令下载 supervisor

`$ easy_install supervisor`

然后创建配置文件

`$ echo_supervisord_conf > /etc/supervisord.conf`

修改配置文件

`$ vi /etc/supervisord.conf`

在文件末尾添加

```$xslt
[program:ssserver]
command = ssserver -c /etc/shadowsocks.json
autostart=true
autorestart=true
startsecs=3
```
设置 supervisord 开机启动，编辑启动文件

`$ vi /etc/rc.local`

在末尾另起一行添加

`$ supervisord`

保存退出（和上文类似）。另 CentOS7 还需要为 rc.local 添加执行权限

`$ chmod +x /etc/rc.local`

至此运用 supervisord 控制 Shadowsocks 开机自启和后台运行设置完成。重启服务器即可

P.S. 如果当你在运行 supervisord 命令时出现以下的错误提示

```$xslt
$ [root@zhgqthomas ~]# supervisord
/usr/lib/python2.7/site-packages/supervisor-3.2.0-py2.7.egg/supervisor/options.py:296: UserWarning: Supervisord is running as root and it is searching for its configuration file in default locations (including its current working directory); you probably want to specify a "-c" argument specifying an absolute path to a configuration file for improved security.
'Supervisord is running as root and it is searching '
Error: Another program is already listening on a port that one of our HTTP servers is configured to use. Shut this program down first before starting supervisord.
For help, use /usr/bin/supervisord -h
```
那是因为 supervisord 已经启动了,重复启动就会出现上面的错误提示.

后记: 看到评论里很多人说搬瓦工自带了 SS 的一键安装功能。确实没错，搬瓦工安装的也是 python 版本的 SS。故出了一篇用 C 语言写的 SS 服务的搭建文章, 亲测 C 语言版本的占用内存更少并且速度比 python 版本的更快。

## 搭建 Strongswan 实现在 iOS 上连接 VPN

### Appstore搜索SuperWingy下载设置服务器IP 密码 即可

 ******
##### （原文）
如果你只是需要在 Android, PC 上使用 VPN，那可以直接忽略此章内容， Shadowsocks 已经可以非常完美的帮助以上设备实现翻墙。 
但是由于 iOS 上无法使用 Shadowsocks 所以需要使用 Strongswon 建立 IPsecVPN。

下载并编译 Strongswan
首先我们来编译 Strongswan， 因为直接用 yum install 的不能用，原因不明，所以直接下载源码和依赖包进行编译

下载 Strongswan 的源码

$ wget http://download.strongswan.org/strongswan.tar.gz && tar zxvf strongswan* 
$ cd strongswan*
下载编译源码所需要的依赖包(小内存请分批下载)

$ yum install -y make gcc gmp-devel openssl openssl-devel
因搬瓦工是 OpenVZ 的所以用下面的命令来进行配置

$ ./configure --sysconfdir=/etc --disable-sql --disable-mysql --disable-ldap --enable-dhcp --enable-eap-identity --enable-eap-mschapv2 --enable-md4 --enable-xauth-eap --enable-eap-peap --enable-eap-md5 --enable-openssl --enable-shared --enable-unity --enable-eap-tls   --enable-eap-ttls --enable-eap-tnc --enable-eap-dynamic --enable-addrblock --enable-radattr --enable-nat-transport --enable-kernel-netlink --enable-kernel-libipsec
开始编译源代码

$ make && sudo make install
没有错误出现后，可进行下一步

使用 PSK + XAUTH 形式连接 Strongswan
使用该方式 iOS 设备无需证书,只需要使用账户名,密码及密钥的情况下即可连接.

配置 Strongswan
编辑 /etc/ipsec.conf

$ vi /etc/ipsec.conf
将下面的代码覆盖原有内容

```$xslt
config setup
    # strictcrlpolicy=yes
    uniqueids=never #允许多设备同时在线
    # charondebug="cfg 2, dmn 2, ike 2, net 0" #要看Log时，取消注释本行

conn IPsec_xauth_psk
    keyexchange=ikev1
    left=SERVER #这里换成你登录 VPN 用的域名或 IP
    leftauth=psk
    leftsubnet=0.0.0.0/0
    right=%any
    rightauth=psk
    rightauth2=xauth
    rightsourceip=10.0.0.0/24
    auto=add

conn %default
    keyexchange=ikev1
    dpdaction=hold
    dpddelay=600s
    dpdtimeout=5s
    lifetime=24h
    ikelifetime=240h
    rekey=no
    left=SERVER #这里换成你登录 VPN 用的域名或 IP，与生成证书时相同 
    leftsubnet=0.0.0.0/0
    leftcert=vpnHostCert.pem
    leftsendcert=always
    right=%any
    rightdns=8.8.8.8
    rightsourceip=10.0.0.0/24
```
编辑 /etc/ipsec.secrets， 创建用户名及密码

vi /etc/ipsec.secrets
将一下内容添加进去

#验证用户所需的信息
: PSK "SECRET" # 这里 SECRET 可随意替换成你想要的密钥
你的用户名 : XAUTH "你的密码"
启动 Strongswan 服务
以上设置完成后, 运行下面的命令,就可以去 iOS 中设置了

$ ipsec start
在 iOS 中选择添加 IPSec VPN 设置,然后输入服务器的域名或者 IP 地址,用户名,密码及上面的密钥就可以连接成功了

使用 RSA + .pem 证书形式来连接 Strongswan
该方法需要生成证书并通过 Email 传到 iOS 设备上方可,然后再通过账户名及密码来连接.闲麻烦的朋友可直接忽略此方式.

生成证书
建立个临时目录来生成证书

$ mkdir ~/ipsec_cert && cd ~/ipsec_cert
生成服务器证书
用的是 iOS8 不越狱翻墙方案中创建的脚本。SERVER 换成自己的域名或IP 都行

$ wget https://gist.githubusercontent.com/songchenwen/14c1c663ea65d5d4a28b/raw/cef8d8bafe6168388b105f780c442412e6f8ede7/server_key.sh
$ sh server_key.sh SERVER
生成客户端证书
同样是他的脚本，这个脚本还会生成一个 .p12 证书，这个证书需要导入到 iOS 里，USER 换成你自己的用户名 EMAIL 换成你自己的 email。运行命令的时候会提示输入一个密码,记住所输入的密码,在将此证书安装到 iPhone 设备的时候会提示输入该密码.

$ wget https://gist.githubusercontent.com/songchenwen/14c1c663ea65d5d4a28b/raw/54843ae2e5e6d1159134cd9a90a08c31ff5a253d/client_key.sh
$ sh client_key.sh USER EMAIL
复制证书到 /etc/ipsec.d/
Strongswan 需要的是 cacerts/strongswanCert.pem certs/vpnHostCert.pem private/vpnHostKey.pem 这三个文件

$ sudo cp cacerts/strongswanCert.pem /etc/ipsec.d/cacerts/strongswanCert.pem 
$ sudo cp certs/vpnHostCert.pem /etc/ipsec.d/certs/vpnHostCert.pem
$ sudo cp private/vpnHostKey.pem /etc/ipsec.d/private/vpnHostKey.pem
同步客户端证书到本地
客户端需要的是 .p12 证书和 cacerts/strongswanCert.pem 将这两个证书同步到本地，然后通过邮件发送到 iOS 设备中并安装.
P.S. 以下命令需要在本地的命令终端运行,而不是在 VPS 里运行.

$ scp -P ssh端口 root@服务器ip:~/ipsec_cert/****.p12 ~/
$ scp -P ssh端口 root@服务器ip:~/ipsec_cert/cacerts/strongswanCert.pem ~/
配置 Strongswan
编辑 /etc/ipsec.conf

$ vi /etc/ipsec.conf
将下面的代码覆盖原有内容

config setup
    # strictcrlpolicy=yes
    uniqueids=never #允许多设备同时在线
    # charondebug="cfg 2, dmn 2, ike 2, net 0" #要看Log时，取消注释本行

conn %default
    keyexchange=ikev1
    dpdaction=hold
    dpddelay=600s
    dpdtimeout=5s
    lifetime=24h
    ikelifetime=240h
    rekey=no
    left=SERVER #这里换成你登录 VPN 用的域名或 IP，与生成证书时相同 
    leftsubnet=0.0.0.0/0
    leftcert=vpnHostCert.pem
    leftsendcert=always
    right=%any
    rightdns=8.8.8.8
    rightsourceip=10.0.0.0/24

conn CiscoIPSec
    rightauth=pubkey
    rightauth2=xauth
    auto=add
编辑 /etc/ipsec.secrets， 创建用户名及密码

vi /etc/ipsec.secrets
将一下内容添加进去

#验证用户所需的信息
#用户名 : EAP "密码"
: RSA vpnHostKey.pem
你的用户名 : EAP "你的密码"
使用 firewalld 配置防火墙
不管选择哪种方式连接,都需要用 firewalld 开放 4500、500 端口和 esp 协议

$ vi /usr/lib/firewalld/services/ipsec.xml
<?xml version="1.0" encoding="utf-8"?>
<service>
  <short>IPsec</short>
  <description>Internet Protocol Security (IPsec) incorporates security for network transmissions directly into the Internet Protocol (IP). IPsec provides methods for both encrypting data and authentication for the host or network it sends to. If you plan to use a vpnc server or FreeS/WAN, do not disable this option.</description>
  <port protocol="ah" port=""/>
  <port protocol="esp" port=""/>
  <port protocol="udp" port="500"/>
  <port protocol="udp" port="4500"/>
</service>
然后输入一下命令后，至此整个搭建过程就结束了。

$ firewall-cmd --permanent --add-service=ipsec
$ firewall-cmd --permanent --add-masquerade
$ firewall-cmd --reload
开机自启 Strongswan
$ vi /etc/rc.local
文件末尾添加

ipsec start
把下载的两个证书用 email 发送到你的 iOS 上，安装后建立个 VPN 连接，选 IPsec，使用证书，选择你的用户名的证书即可，登录下试试吧。如有任何疑问请将您的疑问留在评论区。

使用 FinalSpeed 来加速翻墙
使用 FinalSpeed 来加速 Shadowsocks，可以达到飞一般的翻墙体验。看 Youtube 的 1080P 视频毫无压力。以下是使用 FinalSpeed 的注意事项:

FinalSpeed必须服务端和客户端同时配合使用,否则没有任何加速效果.
服务器建议至少256M内存
openvz架构只支持udp协议 (故搬瓦工的 VPS 只支持 udp 协议)
服务端可以和锐速共存,互不影响
在搬瓦工的 VPS 安装 FinalSpeed
FinalSpeed 支持 Linux,Centos,Ubuntu,Debian. 输入以下命令，完成一键安装

$ wget http://fs.d1sm.net/finalspeed/install_fs.sh
$ chmod +x 
$ install_fs.sh./install_fs.sh 2>&1 | tee install.log
安装完后运行以下命令来查看日志

$ tail -f /fs/server.log
出现以下内容说明安装成功

FinalSpeed server start success.
如出现错误，请到这里查看解决方法。

修改端口
FinalSpeed 默认使用 150 的端口，故需要使用 firewalld 开放 150 端口

$ vi /usr/lib/firewalld/services/fs.xml
将以下内容复制进去

<?xml version="1.0" encoding="utf-8"?>
<service>
   <short>FinalSpeed</short>
   <description>FinalSpeed</description>
   <port protocol="tcp" port="150"/>
   <port protocol="udp" port="150"/>
</service>
$ firewall-cmd --permanent --add-service=fs
$ firewall-cmd --reload
设置开机启动
$ vi /etc/rc.local
在末尾加入

sh /fs/start.sh
有需要的朋友也可以加入每天晚上 3 点自动重启功能

$ crontab -e
加入

0 3 * * *  sh /fs/restart.sh
FinalSpeed 客户端的配置
关于客户端的配置教程，就不重复造轮子了，请到这里查看。
以上就是搭建自己 VPN 服务器的全部内容，这次妈妈再也不用担心我在 Youtube 看 1080P 的视频了~

2016/08/10 更新
Proxychains 实现 Terminal 翻墙
很多人问怎么样实现 Linux/Mac 上 Termial, 实现翻墙的功能，因为有的时候我们需要 curl/wget 一些墙外的资源，可以使用 proxychains 服务。
通过 git 将 proxychains 下载到本地以后，通过 cd 命令进入到 proxychains 文件夹后分别运行以下命令

$ ./configure
$ sudo make
$ sudo make install
通过运行上面的三条命令，就将 proxychains 安装到你的电脑上了，最后需要将 proxychains 文件夹中 src文件夹里的配置文件放到 /etc 文件夹里

$ sudo cp ~/proxychains/src/proxychains.conf /etc/proxychains.conf
然后通过 vim 修改 /etc/proxychains.conf, 在最后一行添加

socks5     127.0.0.1 1080
这样就可以让 proxychains 去代理 shadowsocks 的 socks5 1080 端口了，proxychains 的搭建就算完成了，要使用的时候只要在命令前面加上 proxychains4 就可以了，例如

$ proxychains4 curl https://www.google.com
git 代理翻墙
用 git clone 下载 github 上的仓库的时候，速度很慢，才几十k每秒，稍微大点的仓库，要等到猴年马月。利用 shadowsocks 的 socks5 代理，配置好后明显加速。用下面两条命令配置好后，保持 shadowsocks 客户端开启就行了。

$ git config --global http.proxy 'socks5://127.0.0.1:1080'
$ git config --global https.proxy 'socks5://127.0.0.1:1080'
Shadowsocks 的本地端口默认是 1080, git 仓库有的在国内有的在国外,国内的有 gitcafe coding.net 开源中国 git 所以用国内的就没必要设置了，反而会慢。
