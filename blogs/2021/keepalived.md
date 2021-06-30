---
title: keepalived 笔记
date: '2021-06-30 20:01:49'
sidebar: false
categories:
 - 技术
tags:
 - 高可用
publish: true
---

> 最近研究k8s高可用部署时用到这个工具，随便做下笔记。
>
> 高可用：两台业务系统启动着相同的服务，如果有一台故障，另一台自动接管,我们将将这个称之为高可用。



## 为什么用它？

k8s需要lb+keepalived来保证高可用的api-server，具体是两台机器安装nginx、keepalived，nginx stream 代理3个master节点。keepalived一个 master 一个 backup 提供 vip。



## 是什么？

Keepalived起初是为LVS设计的，专门用来监控集群系统中各个服务节点的状态，它根据TCP/IP参考模型的第三、第四层、第五层交换机制检测每个服务节点的状态，如果某个服务器节点出现异常，或者工作出现故障，Keepalived将检测到，并将出现的故障的服务器节点从集群系统中剔除，这些工作全部是自动完成的，不需要人工干涉，需要人工完成的只是修复出现故障的服务节点。

后来Keepalived又加入了VRRP的功能，VRRP（Vritrual Router Redundancy Protocol,虚拟路由冗余协议)出现的目的是解决静态路由出现的单点故障问题，通过VRRP可以实现网络不间断稳定运行，因此Keepalvied 一方面具有服务器状态检测和故障隔离功能，另外一方面也有HA cluster功能，下面介绍一下VRRP协议实现的过程。

## 配置

`/etc/keepalived/keepalived.conf`

```
global_defs {
   notification_email {  #指定keepalived在发生切换时需要发送email到的对象，一行一个
		monitor@3evip.cn
   }
   notification_email_from monitor@3evip.cn #指定发件人
   smtp_server stmp.3evip.cn #指定smtp服务器地址
   smtp_connect_timeout 30 #指定smtp连接超时时间
   router_id LVS_DEVEL #运行keepalived机器的一个标识
}

# 这块基本用不到，没怎么学
vrrp_sync_group VG_1{ #监控多个网段的实例
	group {
		inside_network #实例名
		outside_network
	}
	notify_master /path/xx.sh #指定当切换到master时，执行的脚本
	netify_backup /path/xx.sh #指定当切换到backup时，执行的脚本
	notify_fault "path/xx.sh VG_1" #故障时执行的脚本
	notify /path/xx.sh 
	smtp_alert #使用global_defs中提供的邮件地址和smtp服务器发送邮件通知
}

vrrp_instance inside_network {
    state BACKUP #指定那个为master，那个为backup，如果设置了nopreempt这个值不起作用，主备考priority决定
    interface eth0 #设置实例绑定的网卡，ifconfig 查看网卡
    dont_track_primary #忽略vrrp的interface错误（默认不设置）
    track_interface{ #设置额外的监控，里面那个网卡出现问题都会切换
		eth0
		eth1
    }
    mcast_src_ip #发送多播包的地址，如果不设置默认使用绑定网卡的primary ip
    garp_master_delay #在切换到master状态后，延迟进行gratuitous ARP请求
    virtual_router_id 50 #VPID标记，主备必须一样
    priority 100 #优先级，高优先级竞选为master
    advert_int 1 #检查间隔，默认1秒
    nopreempt #设置为不抢占 注：这个配置只能设置在backup主机上，而且这个主机优先级要比另外一台高
    preempt_delay #抢占延时，默认5分钟
    debug #debug级别
    authentication { #设置认证
        auth_type PASS #认证方式
        auth_pass 111111 #认证密码
    }
    virtual_ipaddress { #设置vip
        192.168.36.200
    }
}

# 这块也用不到
virtual_server 192.168.36.99 80 {
    delay_loop 6 #健康检查时间间隔
    lb_algo rr  #lvs调度算法rr|wrr|lc|wlc|lblc|sh|dh
    lb_kind DR  #负载均衡转发规则NAT|DR|RUN
    persistence_timeout 5 #会话保持时间
    protocol TCP #使用的协议
    persistence_granularity <NETMASK> #lvs会话保持粒度
    virtualhost <string> #检查的web服务器的虚拟主机（host：头）    
    sorry_server<IPADDR> <port> #备用机，所有realserver失效后启用
	real_server 192.168.200.5 23 {
            weight 1 #默认为1,0为失效
            inhibit_on_failure #在服务器健康检查失效时，将其设为0，而不是直接从ipvs中删除 
            notify_up <string> | <quoted-string> #在检测到server up后执行脚本
            notify_down <string> | <quoted-string> #在检测到server down后执行脚本
			TCP_CHECK {
				connect_timeout 3 #连接超时时间
				nb_get_retry 3 #重连次数
				delay_before_retry 3 #重连间隔时间
				connect_port 23  健康检查的端口的端口
				bindto <ip>   
			}
			HTTP_GET | SSL_GET{
				url{ #检查url，可以指定多个
						path /
						digest <string> #检查后的摘要信息
						status_code 200 #检查的返回状态码
				}
				connect_port <port> 
				bindto <IPADD>
				connect_timeout 5
				nb_get_retry 3
				delay_before_retry 2
			}

			SMTP_CHECK{
					host{
						connect_ip <IP ADDRESS>
						connect_port <port> #默认检查25端口
						bindto <IP ADDRESS>
					}
					connect_timeout 5
					retry 3
					delay_before_retry 2
					helo_name <string> | <quoted-string> #smtp helo请求命令参数，可选
			}
			MISC_CHECK{
					misc_path <string> | <quoted-string> #外部脚本路径
					misc_timeout #脚本执行超时时间
					misc_dynamic #如设置该项，则退出状态码会用来动态调整服务器的权重，返回0 正常，不修改；返回1，检查失败，权重改为0；返回2-255，正常，权重设置为：返回状态码-2
			}
    }
}
```

## 测试

两台机器一台 master 一台 backup，`tail -f /var/log/messages`, 关掉其中一台的keepalived，查看 `ip addr`，vip 是否切换。
