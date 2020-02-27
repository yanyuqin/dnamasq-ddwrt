本文是Linux（Ubuntu）客户端使用SS科学上网设置，配合dnscrypt-proxy使用

# 1. shadowsocks-libev 编译 
## 1.1准备工作
### Installation of basic build dependencies
### Debian / Ubuntu


    sudo apt-get install --no-install-recommends gettext build-essential autoconf libtool \
    libpcre3-dev asciidoc xmlto libev-dev libc-ares-dev automake libmbedtls-dev libsodium-dev
## CentOS / Fedora / RHEL


    sudo yum install gettext gcc autoconf libtool automake make asciidoc xmlto c-ares-devel libev-devel
### 编译相关lib
#### # Installation of libsodium


    export LIBSODIUM_VER=1.0.16
    wget https://download.libsodium.org/libsodium/releases/libsodium-$LIBSODIUM_VER.tar.gz
    tar xvf libsodium-$LIBSODIUM_VER.tar.gz
    pushd libsodium-$LIBSODIUM_VER
    ./configure --prefix=/usr && make
    sudo make install
    popd
    sudo ldconfig

#### # Installation of MbedTLS


    export MBEDTLS_VER=2.6.0
    wget https://tls.mbed.org/download/mbedtls-$MBEDTLS_VER-gpl.tgz
    tar xvf mbedtls-$MBEDTLS_VER-gpl.tgz
    pushd mbedtls-$MBEDTLS_VER
    make SHARED=1 CFLAGS="-O2 -fPIC"
    sudo make DESTDIR=/usr install
    popd
    sudo ldconfig
### 下载源码


    git clone https://github.com/shadowsocks/shadowsocks-libev.git
    cd shadowsocks-libev
    git submodule update --init --recursive
### 编译安装
./autogen.sh && ./configure && make
sudo make install

------------


# 2.ss客户端配置
放在/etc/shadowsocks-libev目录下，local.conf

    {
        "server":"4.5.6.1",
        "server_port":443,
        "local_port":1080,
        "password":"xxxxx",
        "timeout":60,
        "method":"aes-128-gcm",
        "mode":"tcp_and_udp",
        "fast_open":true,
        "plugin":"obfs-local",
        "plugin_opts":"obfs=http;obfs-host=taobao.com"
    }
    


# 3.ipset与iptables设置 
## 3.1生成chnroute

    curl 'http://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest' | \
    grep ipv4 | grep CN | awk -F\| '{ printf("%s/%d\n", $4, 32-log($5)/log(2)) }' \
    > chnroute.txt
## 3.2 ipset与iptables脚本


   

    #!/bin/bash
    
    ipset -N chnroute hash:net maxele ipset -N chnroute hash:net maxelem 65536
        
        for ip in $(cat '/etc/chnroute.txt'); do
          ipset add chnroute $ip
        done
        
        iptables -t nat -N SHADOWSOCKS
        
        # 直连服务器 IP
        iptables -t nat -A SHADOWSOCKS -d ss服务器ip/32 -j RETURN
        
        # 允许连接保留地址
        iptables -t nat -A SHADOWSOCKS -d 0.0.0.0/8 -j RETURN
        iptables -t nat -A SHADOWSOCKS -d 10.0.0.0/8 -j RETURN
        iptables -t nat -A SHADOWSOCKS -d 127.0.0.0/8 -j RETURN
        iptables -t nat -A SHADOWSOCKS -d 169.254.0.0/16 -j RETURN
        iptables -t nat -A SHADOWSOCKS -d 172.16.0.0/12 -j RETURN
        iptables -t nat -A SHADOWSOCKS -d 192.168.0.0/16 -j RETURN
        iptables -t nat -A SHADOWSOCKS -d 224.0.0.0/4 -j RETURN
        iptables -t nat -A SHADOWSOCKS -d 240.0.0.0/4 -j RETURN
        
        # 直连中国 IP
        iptables -t nat -A SHADOWSOCKS -p tcp -m set --match-set chnroute dst -j RETURN
        iptables -t nat -A SHADOWSOCKS -p icmp -m set --match-set chnroute dst -j RETURN
        
        # 重定向到 ss-redir 端口
        iptables -t nat -A SHADOWSOCKS -p tcp -j REDIRECT --to-port 1080
        #`iptables -t nat -A SHADOWSOCKS -p udp -j REDIRECT --to-port 10800
        iptables -t nat -A OUTPUT -p tcp -j SHADOWSOCKS ipset -N chnroute hash:net maxelem 65536
        
        for ip in $(cat '/etc/chnroute.txt'); do
          ipset add chnroute $ip
        done
        
        iptables -t nat -N SHADOWSOCKS
        
        # 直连服务器 IP
        iptables -t nat -A SHADOWSOCKS -d ss服务器ip/32 -j RETURN
        
        # 允许连接保留地址
        iptables -t nat -A SHADOWSOCKS -d 0.0.0.0/8 -j RETURN
        iptables -t nat -A SHADOWSOCKS -d 10.0.0.0/8 -j RETURN
        iptables -t nat -A SHADOWSOCKS -d 127.0.0.0/8 -j RETURN
        iptables -t nat -A SHADOWSOCKS -d 169.254.0.0/16 -j RETURN
        iptables -t nat -A SHADOWSOCKS -d 172.16.0.0/12 -j RETURN
        iptables -t nat -A SHADOWSOCKS -d 192.168.0.0/16 -j RETURN
        iptables -t nat -A SHADOWSOCKS -d 224.0.0.0/4 -j RETURN
        iptables -t nat -A SHADOWSOCKS -d 240.0.0.0/4 -j RETURN
        
        # 直连中国 IP
        iptables -t nat -A SHADOWSOCKS -p tcp -m set --match-set chnroute dst -j RETURN
        iptables -t nat -A SHADOWSOCKS -p icmp -m set --match-set chnroute dst -j RETURN
        
        # 重定向到 ss-redir 端口
        iptables -t nat -A SHADOWSOCKS -p tcp -j REDIRECT --to-port 1080
        #`iptables -t nat -A SHADOWSOCKS -p udp -j REDIRECT --to-port 10800
        iptables -t nat -A OUTPUT -p tcp -j SHADOWSOCKS
## 3.3测试
### 手工启ss客户端进程


    ss-redir -v -c /etc/shadowsocks-libev/local.json
### curl 访问测试


   

    #获得ip地址
    curl https://ip.cn
    curl cip.cc
    curl ifconfig.me
    # 访问各大网站，若均有网页源码输出，说明 tcp 转发已成功
        curl -sL www.baidu.com
        curl -sL www.google.com
        curl -sL www.google.com.hk
        curl -sL www.google.co.jp
        curl -sL www.youtube.com
        curl -sL mail.google.com
        curl -sL facebook.com
        curl -sL twitter.com
        curl -sL www.wikipedia.org
# systemd 服务与永久化

参考链接systemd帮助文档http://www.jinbuguo.com/systemd/systemd.service.html

## 4.1生产ipset与iptable脚本

```shell

```

启动脚本ss-up：

```shell
#!/bin/bash

ipset -N chnroute hash:net maxelem 65536

for ip in $(cat '/etc/chnroute.txt'); do
  ipset add chnroute $ip
done

iptables -t nat -N SHADOWSOCKS

# 直连服务器 IP
iptables -t nat -A SHADOWSOCKS -d 45.63.54.101/32 -j RETURN

# 允许连接保留地址
iptables -t nat -A SHADOWSOCKS -d 0.0.0.0/8 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 10.0.0.0/8 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 127.0.0.0/8 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 169.254.0.0/16 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 172.16.0.0/12 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 192.168.0.0/16 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 224.0.0.0/4 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 240.0.0.0/4 -j RETURN

# 直连中国 IP
iptables -t nat -A SHADOWSOCKS -p tcp -m set --match-set chnroute dst -j RETURN
iptables -t nat -A SHADOWSOCKS -p icmp -m set --match-set chnroute dst -j RETURN

# 重定向到 ss-redir 端口
iptables -t nat -A SHADOWSOCKS -p tcp -j REDIRECT --to-port 1080
#`iptables -t nat -A SHADOWSOCKS -p udp -j REDIRECT --to-port 10800
iptables -t nat -A OUTPUT -p tcp -j SHADOWSOCKS
```



停止脚本ss-down.sh：

```shell
#!/bin/bash

iptables -t nat -D OUTPUT -p tcp -j SHADOWSOCKS
iptables -t nat -F SHADOWSOCKS
iptables -t nat -X SHADOWSOCKS
ipset destroy chnroute

```

## 4.2 修改systemd服务脚本

shadowsocks-libev/debian 源代码目录debian下有systemd服务脚本

```shell
ls -l *service
-rw-r--r-- 1 root root 799 Feb 25 03:52 shadowsocks-libev-local@.service
-rw-r--r-- 1 root root 838 Feb 25 03:52 shadowsocks-libev-redir@.service
-rw-r--r-- 1 root root 801 Feb 25 03:52 shadowsocks-libev-server@.service
-rw-r--r-- 1 root root 840 Feb 25 03:52 shadowsocks-libev.service
-rw-r--r-- 1 root root 813 Feb 25 03:52 shadowsocks-libev-tunnel@.service

cp shadowsocks-libev-redir@.service /etc/systemd/system/shadowsocks-libev-redir@local.service

#修改对应的配置文件，就是@字符后的哪个变量
#local.conf
# vi /etc/shadowsocks-libev/%i.json  i%用local代替

```

修改服务文件/etc/systemd/system/shadowsocks-libev-redir@local.service

```shell
cat /etc/systemd/system/shadowsocks-libev-redir@local.service
[Unit]
Description=Shadowsocks-Libev Custom Client Service Redir Mode for %I
Documentation=man:ss-redir(1)
After=network-online.target

[Service]
Type=simple
CapabilityBoundingSet=CAP_NET_ADMIN CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_ADMIN CAP_NET_BIND_SERVICE
User=nobody
Group=nogroup
ExecStartPre=+/home/ipset/ss-up.sh
ExecStart=/usr/bin/ss-redir -v -c /etc/shadowsocks-libev/%i.json
ExecStop=+/home/ipset/ss-down.sh

[Install]
WantedBy=multi-user.target
```

再次修改为快速启动版，使用ipset与iptables restore快速生产哈希表，比脚本快40秒

```shell
cat /etc/systemd/system/shadowsocks-libev-redir@local.service
[Unit]
Description=Shadowsocks-Libev Custom Client Service Redir Mode for %I
Documentation=man:ss-redir(1)
After=network-online.target

[Service]
Type=simple
CapabilityBoundingSet=CAP_NET_ADMIN CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_ADMIN CAP_NET_BIND_SERVICE
User=nobody
Group=nogroup
ExecStartPre=+/sbin/ipset restore -file /home/ipset/chnroute.ipset
#ExecStartPre=+/sbin/iptables-restore -t /home/ipset/rules.v4
ExecStartPre=+/usr/sbin/netfilter-persistent restore
ExecStart=/usr/bin/ss-redir -v -c /etc/shadowsocks-libev/%i.json
###ExecStart=/usr/bin/ss-redir -c /etc/shadowsocks-libev/%i.json
ExecStop=+/home/ipset/ss-down.sh

[Install]
WantedBy=multi-user.target

```

## 4.3参考命令

```shell
ipset save chnroute -f chnroute.ipset
iptables-save > /etc/iptables/rules.v4
iptables-save > ./rules.v4
iptables-restore  < /etc/iptables/rules.v4
ExecStart=/sbin/ipset restore -file /etc/iptables/ipset

# Uncomment to save changed sets on reboot

# ExecStop=/sbin/ipset save -file /etc/iptables/ipset

ExecStop=/sbin/ipset flush
ExecStopPost=/sbin/ipset destroy
```




