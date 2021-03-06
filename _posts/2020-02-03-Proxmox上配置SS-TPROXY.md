---
layout:     post
title:     Proxmox 上配置 SS-TPROXY
subtitle:   
date:       2020-02-03
author:     Jet
header-img: img/post-bg-2015.jpg
catalog: true
tags: 
- tproxy
- proxmox
- v2ray
---
# Proxmox 上配置 SS-TPROXY

## 安装v2ray
```
bash <(curl -L -s https://install.direct/go.sh)
```
## 安装dns2tcp
~~不知道是否必要，反正我安装了。~~

应该没有必要，不过也安装了。
```
apt-get install dns2tcp
```
## 安装ss-tproxy
```
git clone https://github.com/zfl9/ss-tproxy
cd ss-tproxy
chmod +x ss-tproxy
cp -af ss-tproxy /usr/local/bin
mkdir -p /etc/ss-tproxy
cp -af ss-tproxy.conf gfwlist* chnroute* ignlist* /etc/ss-tproxy
cp -af ss-tproxy.service /etc/systemd/system
```
## 配置ss-tproxy
```
## mode
#mode='global'  # global 模式 (不分流)
#mode='gfwlist' # gfwlist 模式 (黑名单)
mode='chnroute' # chnroute 模式 (白名单)

## ipv4/6
ipv4='true'     # true:启用ipv4透明代理; false:关闭ipv4透明代理
ipv6='false'    # true:启用ipv6透明代理; false:关闭ipv6透明代理

## tproxy
tproxy='false'  # true:TPROXY+TPROXY; false:REDIRECT+TPROXY

## tcponly
tcponly='false' # true:仅代理TCP流量; false:代理TCP和UDP流量

## selfonly
selfonly='false' # true:仅代理本机流量; false:代理本机及"内网"流量

## proxy
#proxy_svraddr4=(1.2.3.4) # 服务器的 IPv4 地址或域名，允许填写多个服务器地址，空格隔开
proxy_svraddr4=(1.2.3.4)  # 服务器的IP
proxy_svraddr6=()        # 服务器的 IPv6 地址或域名，允许填写多个服务器地址，空格隔开
#proxy_svrport='80,443'   # 服务器的监听端口，可填多个端口，格式同 ipts_proxy_dst_port
#proxy_svrport='8390'
proxy_svrport='443'
proxy_tcpport='60080'    # ss/ssr/v2ray 等本机进程的 TCP 监听端口，该端口支持透明代理
proxy_udpport='60080'    # ss/ssr/v2ray 等本机进程的 UDP 监听端口，该端口支持透明代理
#proxy_startcmd='cmd'     # 用于启动本机代理进程的 shell 命令，该命令应该能立即执行完毕
#proxy_stopcmd='cmd'      # 用于关闭本机代理进程的 shell 命令，该命令应该能立即执行完毕
#proxy_startcmd='(ss-redir -c /etc/shadowsocks-libev/config.json -u </dev/null &>>/var/log/ss-redir.log &)'
#proxy_stopcmd='kill -9 $(pidof ss-redir)'
proxy_startcmd='systemctl start v2ray'
proxy_stopcmd='systemctl stop v2ray'

## dns
dns_direct='114.114.114.114'          # 本地 IPv4 DNS，不能指定端口，也可以填组织、公司内部 DNS
dns_direct6='240C::6666'              # 本地 IPv6 DNS，不能指定端口，也可以填组织、公司内部 DNS
dns_remote='8.8.8.8#53'               # 远程 IPv4 DNS，必须指定端口，提示：访问远程 DNS 会走代理
dns_remote6='2001:4860:4860::8888#53' # 远程 IPv6 DNS，必须指定端口，提示：访问远程 DNS 会走代理

## dnsmasq
#dnsmasq_bind_port='53'                  # dnsmasq 服务器监听端口，见 README
dnsmasq_bind_port='60053'
dnsmasq_cache_size='4096'               # DNS 缓存大小，大小为 0 表示禁用缓存
dnsmasq_cache_time='3600'               # DNS 缓存时间，单位是秒，最大 3600 秒
dnsmasq_log_enable='false'              # 记录详细日志，除非进行调试，否则不建议启用
dnsmasq_log_file='/var/log/dnsmasq.log' # 日志文件，如果不想保存日志可以改为 /dev/null
dnsmasq_conf_dir=()                     # `--conf-dir` 选项的参数，可以填多个，空格隔开
dnsmasq_conf_file=()                    # `--conf-file` 选项的参数，可以填多个，空格隔开
dnsmasq_conf_string=()                  # 自定义配置，一个数组元素就是一行配置，空格隔开

## chinadns
chinadns_bind_port='65353'               # chinadns-ng 服务器监听端口，通常不用改动
chinadns_timeout='3'                     # 等待上游 DNS 返回响应的超时时间，单位为秒
chinadns_repeat='1'                      # 向可信 DNS 发送几次 DNS 查询请求，默认为 1
chinadns_fairmode='false'                # 使用公平模式，具体看 chinadns-ng 的 README
chinadns_gfwlist_mode='false'            # gfwlist 模式，加载 gfwlist.txt/gfwlist.ext
chinadns_noip_as_chnip='false'           # 启用 chinadns-ng 的 `--noip-as-chnip` 选项
chinadns_verbose='false'                 # 记录详细日志，除非进行调试，否则不建议启用
chinadns_logfile='/var/log/chinadns.log' # 日志文件，如果不想保存日志可以改为 /dev/null
chinadns_privaddr4=()                    # IPv4 私有地址段，多个用空格隔开，具体见 README
chinadns_privaddr6=()                    # IPv6 私有地址段，多个用空格隔开，具体见 README

## dns2tcp
dns2tcp_bind_port='65454'               # dns2tcp 转发服务器监听端口，如有冲突请修改
dns2tcp_verbose='true'                 # 记录详细日志，除非进行调试，否则不建议启用
dns2tcp_logfile='/var/log/dns2tcp.log'  # 日志文件，如果不想保存日志可以改为 /dev/null

## ipts
ipts_if_lo='lo'                 # 环回接口的名称，在标准发行版中，通常为 lo，如果不是请修改
ipts_rt_tab='233'               # iproute2 路由表名或表 ID，除非产生冲突，否则不建议改动该选项
ipts_rt_mark='0x2333'           # iproute2 策略路由的防火墙标记，除非产生冲突，否则不建议改动该选项
ipts_set_snat='false'           # 设置 iptables 的 MASQUERADE 规则，布尔值，`true/false`，详见 README
ipts_set_snat6='false'          # 设置 ip6tables 的 MASQUERADE 规则，布尔值，`true/false`，详见 README
ipts_reddns_onstop='true'       # ss-tproxy stop 后，是否将其它主机发至本机的 DNS 重定向至直连 DNS，详见 README
ipts_proxy_dst_port='1:65535'   # 黑名单 IP 的哪些端口走代理，多个用逗号隔开，冒号为端口范围(含边界)，详见 README

## opts
opts_ss_netstat='auto'                  # auto/ss/netstat，用哪个端口检测工具，见 README
opts_ping_cmd_to_use='auto'             # auto/standalone/parameter，ping 相关，见 README
opts_hostname_resolver='auto'           # auto/dig/getent/ping，用哪个解析工具，见 README
opts_overwrite_resolv='false'           # true/false，定义如何修改 resolv.conf，见 README
opts_ip_for_check_net='114.114.114.114' # 检测外网是否可访问的 IP，ping，留空表示跳过此检查

## file
file_gfwlist_txt='/etc/ss-tproxy/gfwlist.txt'      # gfwlist/chnlist 模式预置文件
file_gfwlist_ext='/etc/ss-tproxy/gfwlist.ext'      # gfwlist/chnlist 模式扩展文件
file_ignlist_ext='/etc/ss-tproxy/ignlist.ext'      # global/chnroute 模式扩展文件
file_chnroute_set='/etc/ss-tproxy/chnroute.set'    # chnroute 地址段文件 (iptables)
file_chnroute6_set='/etc/ss-tproxy/chnroute6.set'  # chnroute6 地址段文件 (ip6tables)
file_dnsserver_pid='/etc/ss-tproxy/.dnsserver.pid' # dns 服务器进程的 pid 文件 (shell)

## 自定义钩子
pre_start() {
    iptables -t filter -P FORWARD ACCEPT
    ip6tables -t filter -P FORWARD ACCEPT

}
```
主要修改之处：
1. 
>proxy_svrport='443'
2. 
>dnsmasq_bind_port='60053'
3. 
>proxy_startcmd='systemctl start v2ray'
>proxy_stopcmd='systemctl stop v2ray'
4. iptables钩子修改（原因是iptables设置了默认DROP的策略，改为ACCEPT）

## V2RAY配置修改
首先先确认v2ray在WINDOWS上使用正常。
然后修改配置文件/etc/v2ray/config.json：
```
{
  "log": {
    "access": "/var/log/v2ray/access.log",
    "error": "/var/log/v2ray/error.log",
    "loglevel": "debug"
  },

  // inbounds /*
  "inbounds": [
    {
      "protocol": "dokodemo-door",
      "listen": "0.0.0.0", // 如果仅代理本机，可填环回地址
      "port": 60080, // 本地监听端口必须与配置文件中的一致
      "settings": {
        "network": "tcp,udp", // 注意这里是 tcp + udp
        //"network":"kcp",
        "followRedirect": true
      },
      "streamSettings": {
        "sockopt": {
          //"tproxy": "tproxy" // tproxy + tproxy 模式
          "tproxy": "redirect" // redirect + tproxy 模式
        }
      }
    }
   ],
  // inbounds */

  //outbonds /*
  "outbounds": [
    {
      "protocol": "vmess",
      "settings": {
        "vnext": [
          {
            "address": "example.com",
            "port": 443,
            "users": [
              {
                "id": "<UUID here>",
                "alterId": 16
              }
            ]
          }
        ]
      },
      "streamSettings": {
        "network": "ws",
        "security": "tls",
        "wsSettings": {
          "path": "/v2ray"
        }
      }
    }
  ]
  // inbounds */

  //routing /*
  //routing */
}

```
主要修改之处：

1. 按照官方说明，修改inbound部分

2. 按照官方说明，去掉routing部分

## /etc/hosts文件修改
tail /var/log/v2ray/error.log发现，
找不到example.com，于是修改/etc/hosts文件
```
##
151.101.192.133         raw.githubusercontent.com
##
1.2.3.4                 example.com
```

## 测试
```
root@nas:/etc/v2ray#  dig @127.0.0.1 google.com -p 65353

; <<>> DiG 9.11.5-P4-5.1-Debian <<>> @127.0.0.1 google.com -p 65353
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 61033
;; flags: qr rd ra; QUERY: 1, ANSWER: 6, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;google.com.                    IN      A

;; ANSWER SECTION:
google.com.             133     IN      A       172.217.194.138
google.com.             133     IN      A       172.217.194.100
google.com.             133     IN      A       172.217.194.101
google.com.             133     IN      A       172.217.194.102
google.com.             133     IN      A       172.217.194.113
google.com.             133     IN      A       172.217.194.139

;; Query time: 57 msec
;; SERVER: 127.0.0.1#65353(127.0.0.1)
;; WHEN: Mon Feb 03 14:39:26 CST 2020
;; MSG SIZE  rcvd: 135
```
用w3m可以打开google主页。

ANDROID手机：

设为静态IP,网关和域名均设置为PROXMOX的局域网地址，即可科学上网。

## 切换为shadowsocks-libev
经过试验，ss也工作正常。
