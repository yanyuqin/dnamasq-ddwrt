# 步骤
## 1、R6300V2 刷KONG AC firmware，内核4.4.153   
## 2、初始化jffs  
         mount -o bind /jffs/opt /opt  
## 3、安装optware   
  
	     wget  http://bin.entware.net/armv7sf-k3.2/installer/generic.sh  
         运行generic.sh
##  4、安装dnsmasq和dnscrypt-proxy2  
> opkg list |grep ^dns  
> opkg install dnsmasq  
> opkg install  dnscrypt-proxy  
> opkg install wget dig 等
## 5、配置DNSmasq 
### 5.1基本配置 /opt/etc/dnsmasq.conf
> **vi dnsmasq.conf**  
> port=53
> bogus-priv  
> no-resolv  
> no-poll  
> all-servers
> server=1.2.4.8  
> server=180.76.76.76  
> conf-dir=/opt/etc/dnsmasq.d  
> listen-address=127.0.0.1,192.168.1.1  
> cache-size=150  
> dhcp-leasefile=/opt/tmp/dnsmasq.leases  
> dhcp-lease-max=50  
> dhcp-option=br0,3,192.168.1.1  
> dhcp-authoritative  
> dhcp-range=br0,192.168.1.100,192.168.1.149,255.255.255.0,1440m  
> 上行server选阿里、百度、114、cnnic中**ping值最低的**, 
> （114=114.114.114.114;ali=223.5.5.5;baidu=180.76.76.76 cnnic=1.2.4.8)  
### 5.2 dnsmasq.d目录配置  
#### GFW上行配置
从github下载到/opt/etc/dnsmasq.d 目录下
> /opt/bin/wget https://cokebar.github.io/gfwlist2dnsmasq/dnsmasq_gfwlist.conf   \
> -P /opt/etc/dnsmasq.d --no-check-certificate  
   
#### 防ISP污染配置 
> /opt/bin/wget  https://github.com/felixonmars/dnsmasq-china-list/raw/master/bogus-nxdomain.china.conf \  
> -P /opt/etc/dnsmasq.d --no-check-certificate	 
##  6、配置DNScrypt-proxy
> dnsmasq_gfwlist.conf GFW域名默认走127.0.0.1#5353
> listen_addresses = ['127.0.0.1:5353']  
> ipv4_servers = true  
> ipv6_servers = false  
> dnscrypt_servers = false
> doh_servers = true**  
> force_tcp = true      
##	 7、启动服务测试  
> dig -p 5353 @127.0.0.1 www.google.com  
> dig  @127.0.0.1 www.google.com  
> dig  @127.0.0.1 www.baidu.com  
	
	

	



