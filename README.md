#笔记：使用shadowsocks+dnsmasq+ipset+iptables 实现公办网络透明代理（智能翻墙）

##前言：
>网上大多数写的都是关于openWRT路由器关于shadowsocks的配置，
如：http://hong.im/2014/03/16/configure-an-openwrt-based-router-to-use-shadowsocks-and-redirect-foreign-traffic/ 我这里写的是关于办公网络的配置，希望对大家有所帮助。

###1. 所用到的软件 
shadowsocks : https://github.com/shadowsocks/shadowsocks-libev
dnsmasq: http://www.thekelleys.org.uk/dnsmasq/doc.html

###2. 网络拓扑
![image](https://github.com/Witee/shadowsocks/raw/master/tuopu1.png）

a. 网关服务器有两块网卡，eth1为内网，eth0为外网，通过iptables nat转发将内网包转发到外网，
-A POSTROUTING ! -s 192.168.2.1/32 -j SNAT --to-source xxx.xxx.xxx.xxx
以下是iptables相关表与链的图

![image](https://github.com/Witee/shadowsocks/raw/master/iptables.png）

b. 网关安装dnsmasq提供本地dns服务

###3.  使用代理时的拓扑图
![image](https://github.com/Witee/shadowsocks/raw/master/tuopu2.png）
上图为通过代理服务器访问国外网站的结构图

###4. 代理服务器上安装：
a. 
```
yum install build-essential autoconf libtool openssl-devel gcc git -y
```
b. 
```
git clone https://github.com/shadowsocks/shadowsocks-libev.git  
cd shadowsocks ; ./configure —prefix=/usr/local/shadowsocks ;make ; make install  ； mkdir /usr/local/shadowsocks/etc/
```
c.  shadowsocks配置文件，ss-local ss-server ss-redir都是同样的配置
```
cat /usr/local/shadowsocks/etc/config.json
{
  "server":”your.server.ip.x",
  "server_port”:1194,
  "local_port":1080,
  "password”:”your_passwd",
  "timeout":600,
  "method":"aes-256-cfb"
}
解释：
    "server":"[服务器IP地址]",
    "server_port":[服务器端口],
    "local_port":[本地端口],
    "password":"[密码]",
    "timeout":600,
    "method":"[加密方式]"
}
```
d . 启动脚本
```
[root@design etc]# cat /etc/init.d/shadowsocks

start() {
    echo "Starting ss-server..."
    /usr/bin/nohup /usr/local/shadowsocks/bin/ss-server -c /usr/local/shadowsocks/etc/config.json >>/tmp/shadowsock.log 2>&1 &

}
stop() {
    echo "Stopping ss-server..."
    killall ss-server
}

case $1 in
        start)
     start
     ;;
     stop)
     stop
     ;;
     *)
     echo "Usage : $0 start|stop"
     ;;
esac


chmod 755 /etc/init.d/shadowsocks
/etc/init.d/shadowsocks start
```

###5. 网关服务器上安装：
a.  shadowsocks安装方法同上，配置文件一至，启动脚本有所修改：
```
[root@route-back ~]# cat /etc/init.d/shadowsocks
 #!/bin/bash

start() {
    echo "Starting ss-redir..."
    /usr/bin/nohup /usr/local/shadowsocks/bin/ss-redir -c /usr/local/shadowsocks/etc/config.json >>/tmp/shadowsock.log 2>&1 &

}
stop() {
    echo "Stopping ss-redir..."
    killall ss-redir
}

case $1 in
        start)
     start
     ;;
     stop)
     stop
     ;;
     *)
     echo "Usage : $0 start|stop"
     ;;
esac

```

b. 网关服务器上使用的是ss-redir，也就是透明代理会使用到的程序，如果只是本地用来上网的话，使用ss-local。
到此代理服务器与网关服务器的隧道就建立好了。

c. 将经过网关服务器的请求转发至本地的隧道，也就是1080端口，达到代理的目的，但是这样的话，所有的请求都会使用代理，所以要反过来做，将指定IP的请求转发至隧道。
```
[root@design tmp]# cat ss-black.sh
#!/bin/sh

#create a new chain named SHADOWSOCKS
iptables -t nat -N SHADOWSOCKS

#Redirect what you want

#Google
iptables -t nat -A SHADOWSOCKS -p tcp -d 74.125.0.0/16 -j REDIRECT --to-ports 1080
iptables -t nat -A SHADOWSOCKS -p tcp -d 173.194.0.0/16 -j REDIRECT --to-ports 1080

#Youtube
iptables -t nat -A SHADOWSOCKS -p tcp -d 208.117.224.0/19 -j REDIRECT --to-ports 1080
iptables -t nat -A SHADOWSOCKS -p tcp -d 209.85.128.0/17 -j REDIRECT --to-ports 1080

#Twitter
iptables -t nat -A SHADOWSOCKS -p tcp -d 199.59.148.0/22 -j REDIRECT --to-ports 1080
iptables -t nat -A SHADOWSOCKS -p tcp -d 205.164.0.0/16 -j REDIRECT --to-ports 1080

#Shadowsocks.org
iptables -t nat -A SHADOWSOCKS -p tcp -d 199.27.76.133/32 -j REDIRECT --to-ports 1080

#1024
iptables -t nat -A SHADOWSOCKS -p tcp -d 184.154.128.246/32 -j REDIRECT --to-ports 1080

#Anything else should be ignore
iptables -t nat -A SHADOWSOCKS -p tcp -j RETURN

# Apply the rules
iptables -t nat -A PREROUTING -p tcp -j SHADOWSOCKS

```

以上是网上的脚本，只有列表中的地址会使用代理。
但是这样就得手工维护这个地址，所以要使用更智能的方法，也就是自动获得这些IP地址。

方法是：通过无污染的DNS请求域名，把解析后的IP写到列表中。
d. 具体实施：
    1> yum install ipset   # ipset的使用方法及ipset 与iptables的关系请自行google
    2> ipset -N setmefree iphash  # 新建一个IP的池子 通过 ipset list命令可以查到池中的IP，现在是空的
    3> 添加iptables 
```
iptables -t nat -A PREROUTING -p tcp -m multiport --dports 443 -m set --match-set setmefree dst -j REDIRECT --to-ports 1080
iptables -t nat -A PREROUTING -p tcp -m multiport --dports 80 -m set --match-set setmefree dst -j REDIRECT --to-ports 1080
iptables -t nat -A PREROUTING -p tcp -j RETURN
```
    将tcp请求的80 443端口并且是在池中的目的地址转发至1080端口（使用代理）；其它不使用代理。其中 setmefree 名称要与ipset -N 时使用的名称一致。
    4> 现在池中的地址是空的，所以不会请求通过代理，可以手工添加，方法为：ipset -A setmefree ip/mask ，然后ipset list就可以看到了，但这个方法还是很不智能，所以要配合dnsmasq来自动添加IP地址到池中。
    5> 具体方法：
        从官网下载dnsmasq源码包并安装，不能直接yum安装，因为yum装的不会支持ipset，编写文档时dnsmasq的版本是：dnsmasq-2.72.tar.gz 如无重大更新，请下载最新版本。
       tar -zxf dnsmasq-2.72.tar.gz ; cd dnsmasq-2.72 ; 安装方法在此文件夹的setup.html 中，也就是直接make install ,会把程序安装到 /usr/local/sbin/dnsmasq ，然后cp dnsmasq.conf.example /etc/dnsmasq.conf  ; 直接执行dnsmasq就可以直接启动，配置文件读取/etc/dnsmasq.conf ，配置文件中打开并指定文件 conf-file=/etc/dnsmasq.d/xxx.conf
      修改 xxx.conf 内容为：
```
#Google and Youtube
server=/.google.com/208.67.222.222#443
server=/.google.com.hk/208.67.222.222#443
server=/.gstatic.com/208.67.222.222#443
server=/.ggpht.com/208.67.222.222#443
server=/.googleusercontent.com/208.67.222.222#443
server=/.appspot.com/208.67.222.222#443
server=/.googlecode.com/208.67.222.222#443
server=/.googleapis.com/208.67.222.222#443
server=/.gmail.com/208.67.222.222#443
server=/.google-analytics.com/208.67.222.222#443
server=/.youtube.com/208.67.222.222#443
server=/.googlevideo.com/208.67.222.222#443
server=/.youtube-nocookie.com/208.67.222.222#443
server=/.ytimg.com/208.67.222.222#443
server=/.blogspot.com/208.67.222.222#443
server=/.blogger.com/208.67.222.222#443

#FaceBook
server=/.facebook.com/208.67.222.222#443
server=/.thefacebook.com/208.67.222.222#443
server=/.facebook.net/208.67.222.222#443
server=/.fbcdn.net/208.67.222.222#443
server=/.akamaihd.net/208.67.222.222#443

#Twitter
server=/.twitter.com/208.67.222.222#443
server=/.t.co/208.67.222.222#443
server=/.bitly.com/208.67.222.222#443
server=/.twimg.com/208.67.222.222#443
server=/.tinypic.com/208.67.222.222#443
server=/.yfrog.com/208.67.222.222#443

#Dropbox
server=/.dropbox.com/208.67.222.222#443

#1024
server=/.t66y.com/208.67.222.222#443

#shadowsocks.org
server=/.shadowsocks.org/208.67.222.222#443

#btdigg
server=/.btdigg.org/208.67.222.222#443

#sf.net
server=/.sourceforge.net/208.67.222.222#443

#feedly
server=/.feedly.com/208.67.222.222#443

# Here Comes The ipset

#Google and Youtube
ipset=/.google.com/setmefree
ipset=/.google.com.hk/setmefree
ipset=/.gstatic.com/setmefree
ipset=/.ggpht.com/setmefree
ipset=/.googleusercontent.com/setmefree
ipset=/.appspot.com/setmefree
ipset=/.googlecode.com/setmefree
ipset=/.googleapis.com/setmefree
ipset=/.gmail.com/setmefree
ipset=/.google-analytics.com/setmefree
ipset=/.youtube.com/setmefree
ipset=/.googlevideo.com/setmefree
ipset=/.youtube-nocookie.com/setmefree
ipset=/.ytimg.com/setmefree
ipset=/.blogspot.com/setmefree
ipset=/.blogger.com/setmefree

#FaceBook
ipset=/.facebook.com/setmefree
ipset=/.thefacebook.com/setmefree
ipset=/.facebook.net/setmefree
ipset=/.fbcdn.net/setmefree
ipset=/.akamaihd.net/setmefree

#Twitter
ipset=/.twitter.com/setmefree
ipset=/.t.co/setmefree
ipset=/.bitly.com/setmefree
ipset=/.twimg.com/setmefree
ipset=/.tinypic.com/setmefree
ipset=/.yfrog.com/setmefree

#Dropbox
ipset=/.dropbox.com/setmefree

#1024
ipset=/.t66y.com/setmefree

#shadowsocks.org
ipset=/.shadowsocks.org/setmefree

#btdigg
ipset=/.btdigg.org/setmefree

#sf.net
ipset=/.sourceforge.net/setmefree

#feedly
ipset=/.feedly.com/setmefree

```

注意server=与ipset=是一一对应的，意思就是通过dnsmasq解析出来的IP写到地址池(setmefree)中。
启动dnsmasq，当有访问的时候 ipset list 才会显示出地址，此时访问此列表中的地址的80 443端口的语法就会通过代理了。

###6. 注意事项
a.  xxx.conf 中使用的dns必须是无污染的，也就是没有被强制解析到错误的地址，208.67.222.222 为Opendns ，支持使用非标准端口（443,5353），据说不稳定，当不稳定的时候访问没有在列表中的域名时会不能解析出地址，所以还可以在代理服务器上安装另外一个dnsmasq，设置缓存大一点（cache-size=1000000），并通过设置resolv-file=/etc/dnsmasq.resolv.conf中的opendns来解析国外IP，网关服务器再使用代理服务器上的dns来解析，这样如果opendns如果不稳定的时候还可以使用代理服务器上的dns缓存来解析。
b. 代理服务器上安装dnsmasq方法一致，只是配置设置：
```
cache-size=1000000
resolv-file=/etc/dnsmasq.resolv.conf
# cat  /etc/dnsmasq.resolv.conf
# nameserver 208.67.222.222     # 但默认是使用53端口，请自行google如何使用443端口来查询

```

c.  ipset list 中的地址是不会自动删除的，所以最好定期执行ipset flush setmefree 来清空setmefree中的IP以保证都是正常的。
d.  网关服务器上还可以安装squid正向代理。
###7. 至此配置完成

