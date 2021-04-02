### nginx安装

1. 安装依赖

```shell
sudo yum -y install gcc gcc-c++ automake pcre pcre-devel zlip zlib-devel openssl openssl-devel
```

2. 编译安装nginx

```shell
tar zxvf nginx-1.14.1.tar.gz

cd nginx-1.14.1

#--prefix=路径为nginx想要安装到的目录
./configure --prefix=/home/${user}/nginx  --with-http_ssl_module(https模块)

make && make install
```

3. 修改配置文件

```shell
mv /home/${user}/nginx/conf/nginx.conf /home/${user}/nginx/conf/nginx.conf.bak

vim /home/${user}/nginx/conf/nginx.conf
```

4. 启动nginx

```shell
cd /home/${user}/nginx/sbin

./nginx -c /home/ppr_user/nginx/conf/nginx.conf
```

5. 配置HTTPS

```shell
#生成http证书
openssl req -x509 -nodes -days 36500 -newkey rsa:2048 -keyout test.key -out test.crt

Common Name 输入VIP：
Generating a 2048 bit RSA private key
............................+++
.....+++
writing new private key to 'test.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:c
string is too short, it needs to be at least 2 bytes long
Country Name (2 letter code) [XX]:c
string is too short, it needs to be at least 2 bytes long
Country Name (2 letter code) [XX]:c
string is too short, it needs to be at least 2 bytes long
Country Name (2 letter code) [XX]:c
string is too short, it needs to be at least 2 bytes long
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:c
Locality Name (eg, city) [Default City]:c
Organization Name (eg, company) [Default Company Ltd]:c
Organizational Unit Name (eg, section) []:c
Common Name (eg, your name or your server's hostname) []:180.2.31.95
Email Address []:test@gse.com.cn
```

修改配置文件中证书目录：

```shell
server {
    ...
    ssl_certificate      /home/ppr_user/nginx/conf/crt/test.crt;	#https证书路径
    ssl_certificate_key  /home/ppr_user/nginx/conf/crt/test.key;	#https证书密钥路径
    ...
}
```

### keepalived安装

1. 编译keepalived

```shell
tar zxvf keepalived-2.0.20.tar.gz

cd ~/keepalived-2.0.20

#--prefix=路径为keepalived想要安装到的目录
./configure --prefix=/home/${user}/keepalived

make && make install
```

2. 配置keepalived

```shell
#拷贝keepalived源码到/etc/init.d/目录
cp /home/${user}/package/keepalived-2.0.20/keepalived/etc/init.d/keepalived.rh.init /etc/init.d/keepalived

mkdir /etc/keepalived

cp /home/${user}/keepalived/etc/sysconfig/keepalived /etc/sysconfig/
```

3. 上传配置文件keepalived.conf，nginx_check.conf到/etc/keepalived/

### 配置文件样例

nginx

```shell

#user  nobody;
worker_processes  1;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

pid        logs/nginx.pid;


events {
	use epoll;	#使用epoll事件处理模型
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    #access_log  logs/access.log  main;

    sendfile        on;

    keepalive_timeout  65;
	
	#限流
	limit_req_zone $binary_remote_addr zone=myLimit:10m rate=100r/s;	#100为每秒并发
	
	limit_conn_zone $binary_remote_addr zone=perip:10m;
	limit_conn_zone $server_name zone=perserver:10m;

	#反向代理服务器配置
    upstream jboss_pool {
        #server jboss地址:端口号 weight表示权值，权值越大，被分配的几率越大;
        server 180.2.31.96:8080 weight=4 max_fails=2 fail_timeout=30s;		#ip和端口以实际服务器为准
        server 180.2.31.97:8080 weight=4 max_fails=2 fail_timeout=30s;		#ip和端口以实际服务器为准
		ip_hash;
    }

    server {
        listen       8081;			#http端口号
        server_name  180.2.31.96;	#nginx所在服务器实际ip
		
		#强制跳转HTTPS
		rewrite ^(.*) https://$server_name:8443$1 permanent;		#此处8443为https端口号，以实际为准
    }
	
	#HTTPS服务器配置
	# HTTPS server

    server {
        listen       443 ssl;			#此处为https端口号
        server_name  180.2.31.95;		#此处为VIP
		client_max_body_size 100M;

		#ssl on;
		
        ssl_certificate      /home/ppr_user/nginx/conf/crt/test.crt;	#https证书路径
        ssl_certificate_key  /home/ppr_user/nginx/conf/crt/test.key;	#https证书密钥路径

        ssl_session_cache    shared:SSL:1m;
        ssl_session_timeout  5m;

		ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers  HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers  on;
		error_page 497 https://$host:$server_port$request_uri;

        location / {
            proxy_pass   http://jboss_pool;
			# 获取请求的host
			proxy_set_header Host $host;
			# 获取请求的ip地址
			proxy_set_header X-real-ip $remote_addr;
			# 获取请求的多级ip地址，当请求经过多个反向代理时，会获取多个ip，英文逗号隔开
			proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
			
			proxy_redirect http:// https://;

			#限流/并发控制
			limit_req zone=myLimit burst=20 nodelay;
			limit_conn perip 100;
			limit_conn perserver 100;
			limit_conn_log_level info;
			limit_req_log_level info;
			limit_conn_status 503;
			limit_req_status 503;
        }
    }

	
}
```

keepalived

```shell
!Configuration File for keepalived
#MASTER
global_defs {
    #
    notification_email {
        # acassen @firewall.loc
        # failover @firewall.loc
        # sysadmin @firewall.loc
    }
    #notification_email_from Alexandre.Cassen @firewall.loc
    # smtp_server 192.168.200.1
    # smtp_connect_timeout 30
    router_id 128
    # vrrp_skip_check_adv_addr# vrrp_strict# vrrp_garp_interval 0# vrrp_gna_interval 0
}
vrrp_script chk_nginx {
    script "/etc/keepalived/nginx_check.sh"
    interval 2
    weight -20
}
vrrp_instance VI_1 {
    state MASTER# 标识为主服务
    interface ens33# 绑定虚拟机的IP
    virtual_router_id 51# 虚拟路由id， 和从机保持一致
    # mcast_src_ip 192.168.138.128# 本机ip
    priority 100# 权重， 需要高于从机
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.138.130
    }
    track_script {
        chk_nginx# 执行 Nginx 监控的服务
    }
}


!Configuration File for keepalived
#BACKUP
global_defs {
    #
    notification_email {
        # acassen @firewall.loc
        # failover @firewall.loc
        # sysadmin @firewall.loc
    }
    #notification_email_from Alexandre.Cassen @firewall.loc
    # smtp_server 192.168.200.1
    # smtp_connect_timeout 30
    router_id 129
    # vrrp_skip_check_adv_addr# vrrp_strict# vrrp_garp_interval 0# vrrp_gna_interval 0
}
vrrp_script chk_nginx {
    script "/etc/keepalived/nginx_check.sh"
    interval 2
    weight -20
}
vrrp_instance VI_1 {
    state BACKUP# 标识为从服务
    interface ens33# 绑定虚拟机的IP
    virtual_router_id 51# 虚拟路由id， 和主机保持一致
    # mcast_src_ip 192.168.138.128# 本机ip
    priority 90# 权重， 需要低于主机
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.138.130
    }
    track_script {
        chk_nginx# 执行 Nginx 监控的服务
    }
}
```

nginx检测脚本nginx_check.sh

```shell
#!/bin/bash
counter=$(ps -C nginx --no-heading|wc -l)
if [ "${counter}" = "0" ]; then
	su - ppr_user -c /home/ppr_user/nginx/sbin/nginx
	sleep 1
	counter=$(ps -C nginx --no-heading|wc -l)
	if [ "${counter}" = "0" ]; then
	    exit 1
	else
		exit 0
	fi
else
	exit 0
fi     
```

