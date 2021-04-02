### Keepalived+Nginx+Jboss/Tomcat高可用负载均衡
#### 架构图
![](https://gitee.com/LoopSup/image/raw/master/img/20191210141333.png)

（1）用户通过域名请求到DNS，由DNS解析域名后返回对应的IP地址，该IP及为Keepalived映射服务器的虚拟IP

（2）通过该虚拟IP访问到对应的负载均衡器（Nginx），这里Nginx部署两个，然后通过Keepalived来保证NG的高可用，正常情况下由Keepalived-M将虚拟IP映射转发至Nginx-M，如果Nginx-M出现故障，此时Keepalived会切换至Keepalived-S开始工作，从而保证了NG的单点故障问题。

（3）通过Nginx负载均衡器，将请求路由到对应的Tomcat服务。

#### Keepalived是什么
Keepalived是一个基于VRRP协议来实现的服务高可用方案，可以利用其来避免IP单点故障，类似的工具还有heartbeat、corosync、pacemaker。但是它一般不会单独出现，而是与其它负载均衡技术（如lvs、haproxy、nginx）一起工作来达到集群的高可用。

#### VRRP协议
VRRP全称 Virtual Router Redundancy Protocol，即 虚拟路由冗余协议。虚拟路由冗余协议，可以认为是实现路由器高可用的协议，即将N台提供相同功能的路由器组成一个路由器组，这个组里面有一个master和多个backup，master上面有一个对外提供服务的vip（该路由器所在局域网内其他机器的默认路由为该vip），master会发组播，当backup收不到vrrp包时就认为master宕掉了，这时就需要根据VRRP的优先级来选举一个backup当master。

keepalived可以认为是VRRP协议在Linux上的实现，主要有三个模块，分别是core、check和vrrp。core模块为keepalived的核心，负责主进程的启动、维护以及全局配置文件的加载和解析。check负责健康检查，包括常见的各种检查方式。vrrp模块是来实现VRRP协议的。

#### keepalived配置文件
keepalived只有一个配置文件keepalived.conf，里面主要包括以下几个配置区域，分别是global_defs、static_ipaddress、static_routes、vrrp_script、vrrp_instance和virtual_server。

**global_defs区域**

主要是配置故障发生时的通知对象以及机器标识,通俗点说就是出状况后发邮件通知的一个配置。

```
global_defs {
    notification_email {    故障发生时给谁发邮件通知
        a@abc.com
        b@abc.com
        ...
    }
    notification_email_from alert@abc.com    通知邮件从哪个地址发出
    smtp_server smtp.abc.com        smpt_server 通知邮件的smtp地址。
    smtp_connect_timeout 30       连接smtp服务器的超时时间
    enable_traps      开启SNMP陷阱
    router_id host163      标识本节点的字条串，通常为hostname
}
```
**vrrp_instance区域**

vrrp_instance用来定义对外提供服务的VIP区域及其相关属性

```
vrrp_instance VI_1 {
    state MASTER         #state 可以是MASTER或BACKUP
    interface ens33        #本机网卡的名字
    virtual_router_id 51      #取值在0-255之间，用来区分多个instance的VRRP组播
    priority 100            #权重
    advert_int 1       #发VRRP包的时间间隔，即多久进行一次master选举
    authentication {        #身份认证区
        auth_type PASS
        auth_pass 1111
    }
    #追踪外围脚本
    track_script {
        chk_nginx       #这里配置vrrp_script的名称
    }
    virtual_ipaddress {        #虚拟ip地址(VIP),可配置多个
        192.168.138.128
    }
}
```
**virtual_server**

超大型的LVS中用到

```
virtual_server 192.168.200.100 443 {
    delay_loop 6                               #延迟轮询时间（单位秒）
    lb_algo rr                                 #后端调试算法
    lb_kind NAT                               #LVS调度类型
    persistence_timeout 50 
    protocol TCP

    real_server 192.168.201.100 443 {       #真正提供服务的服务器
        weight 1
        SSL_GET {
            url {
              path /
              digest ff20ad2481f97b1754ef3e12ecd3a9cc   #表示用genhash算出的结果
            }
            url {
              path /mrtg/
              digest 9b3a0c85a887a256d6939da88aabd8cd
            }
            connect_timeout 3
            nb_get_retry 3      #重试次数
            delay_before_retry 3    #下次重试的时间延迟
        }
    }
}
```
**vrrp_script**

健康检查，当检查失败时会将vrrp_instance的priority减少相应的值。

```
vrrp_script chk_http_port {  
    script "/etc/keepalived/nginx_check.sh"  #脚本所在路径
    interval 1          #脚本执行间隔时间，秒
    weight -10          #优先级
}  
```

**nginx_check.sh**

```
#!/bin/bash
counter=$(ps -C nginx --no-heading|wc -l)
if [ "${counter}" = "0" ]; then
	service nginx start
	sleep 1
	counter=$(ps -C nginx --no-heading|wc -l)
	if [ "${counter}" = "0" ]; then
	    service keepalived stop
	fi
fi
```
- 脚本需要赋权

```
chmod -x /etc/keepalived/nginx_check.sh
```

#### Nginx配置

```
#nginx进程数
worker_processes  1;

#单个进程最大连接数
events {
    worker_connections  1024;
}

#http服务器配置
http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
	#长连接超时时间，单位是秒
    keepalive_timeout  65;
    
	#upstream负载均衡配置，配置路由到tomcat的服务地址以及权重
    upstream localhost{
       server 192.168.138.128:8080 weight=2;
       server 192.168.138.129:8080 weight=2;
    }
	
	#虚拟主机的配置
    server {
	    #监听端口
        listen       80;
		 #域名可以有多个，用空格隔开
        server_name  localhost;
        location / {
            root   html;
            index  index.html index.htm;
			#nginx跟后端服务器连接超时时间(代理连接超时)
            proxy_connect_timeout 3;
			#后端服务器数据回传时间(代理发送超时)
            proxy_send_timeout 30;
			#连接成功后，后端服务器响应时间(代理接收超时)
            proxy_read_timeout 30;
            proxy_pass http://localhost;
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
}
```

#### 防火墙配置

```shell
#开放vrrp协议
firewall-cmd --direct --permanent --add-rule ipv4 filter INPUT 0 --in-interface ens33 --destination 224.0.0.18 --protocol vrrp -j ACCEPT
开放nginx80端口
firewall-cmd --zone=public --add-port=80/tcp –permanent
开放jboss8080端口
firewall-cmd --zone=public --add-port=8080/tcp --permanent
更新firewall规则
firewall-cmd –reload
```

