# 基于 Scapy 编写端口扫描器
## 实验目的
- 掌握网络扫描之端口状态探测的基本原理

## 实验环境
- python 3.9.2
- scapy 2.4.4
- nmap 7.91
- Kali Rolling (2021.1) x64

## 实验要求

- 禁止探测互联网上的 IP ，严格遵守网络安全相关法律法规
- 完成以下扫描技术的编程实现
  - [x] [TCP connect scan](#tcp-connect-scan) / [TCP stealth scan](#tcp-stealth-scan)
  - [x] [TCP Xmas scan](#tcp-xmas-scan) / [TCP fin scan](#tcp-fin-scan) / [TCP null scan](#tcp-null-scan)
  - [x] [UDP scan](#udp-scan)


 - [x] 上述每种扫描技术的实现测试均需要测试端口状态为：开放、关闭 和 过滤 状态时的程序执行结果
 - [x] 提供每一次扫描测试的抓包结果并分析与课本中的扫描方法原理是否相符？如果不同，试分析原因；
 - [x] 在实验报告中详细说明实验网络环境拓扑、被测试 IP 的端口状态是如何模拟的
- [x] 复刻 nmap 的上述扫描技术实现的命令行参数开关

## 实验过程
### 网络拓扑
![](./img/top.png)

### 端口状态设置
- 查看当前防火墙的状态和现有规则
```bash
ufw status
```
- 关闭状态：对应端口没有开启监听, 防火墙没有开启。
```bash
ufw disable
```
- 开启状态：对应端口开启监听，防火墙处于关闭状态。
  - apache2基于TCP, 在80端口提供服务; 
  - DNS服务基于UDP,在53端口提供服务。
```bash
systemctl start apache2 # port 80
systemctl start dnsmasq # port 53
```
- 过滤状态：对应端口开启监听, 防火墙开启。
```bash
ufw enable && ufw deny 80/tcp
ufw enable && ufw deny 53/udp
```

### TCP connect scan
> 先发送一个S，然后等待回应。如果有回应且标识为RA，说明目标端口处于关闭状态；如果有回应且标识为SA，说明目标端口处于开放状态。这时TCP connect scan会回复一个RA，在完成三次握手的同时断开连接.
#### code
```python
from scapy.all import *

src_port = RandShort()
dst_ip = "172.16.111.2"
dst_port = 80

resp = sr1(IP(dst=dst_ip)/TCP(sport=src_port,dport=dst_port,flags="S"),timeout=10)

if(resp.haslayer(TCP)):
    if (resp.getlayer(TCP).flags == 0x14):   
        print "Closed"
    elif(resp.getlayer(TCP).flags == 0x12): 
        send_rst = sr(IP(dst=dst_ip)/TCP(sport=src_port,dport=dst_port,flags="AR"),timeout=10)
        print "Open"


elif resp is None:
    print "Filtered"
```
##### nmap
```
nmap -sT -p 80 172.16.111.2
```
- 端口关闭：
![](./img/tcp-cnt-cls.png)
    - nmap复刻
    ![](./img/tcp-cnt-n-cls.png)
- 端口开放：
![](./img/tcp-cnt-open.png)
    - nmap复刻
    ![](./img/tcp-cnt-n-open.png)
- 端口过滤：
![](./img/tcp-cnt-fil.png)
    - nmap复刻
    ![](./img/tcp-cnt-n-fil.png)

### TCP stealth scan
> 先发送一个S，然后等待回应。如果有回应且标识为RA，说明目标端口处于关闭状态；如果有回应且标识为SA，说明目标端口处于开放状态。这时TCP stealth scan只回复一个R，不完成三次握手，直接取消建立连接。
#### code
```python

from scapy.all import *

src_port = RandShort()
dst_ip = "172.16.111.2" 
dst_port = 80

stl_resp = sr1(IP(dst=dst_ip)/TCP(sport=src_port,dport=dst_port,flags="S"),timeout=10)

if(resp.haslayer(TCP)):
    if (resp.getlayer(TCP).flags == 0x14):
        print "Closed"
    elif(resp.getlayer(TCP).flags == 0x12):
        send_rst = sr(IP(dst=dst_ip)/TCP(sport=src_port,dport=dst_port,flags="R"),timeout=10)
        print "Open"

elif stl_resp is None:
    print "Filtered"
```
##### nmap
```
nmap -sS -p 80 172.16.111.2
```

- 端口关闭：
![](./img/tcp-stl-cls.png)
    - nmap复刻
    ![](./img/tcp-stl-n-cls.png)
- 端口开放：
![](./img/tcp-stl-open.png)
    - nmap复刻
    ![](./img/tcp-stl-n-cls.png)
- 端口过滤：
![](./img/tcp-stl-fil.png)
    - nmap复刻
    ![](./img/tcp-stl-n-cls.png)

### TCP Xmas scan
> 一种隐蔽性扫描，当处于端口处于关闭状态时，会回复一个RST包；其余所有状态都将不回复。
#### code
```python
from scapy.all import *

dst_ip = "172.16.111.2"
dst_port = 80

resp = sr1(IP(dst=dst_ip)/TCP(dport=dst_port,flags="FPU"),timeout=10)

if(resp.haslayer(TCP)):
    if(resp.getlayer(TCP).flags == 0x14):
        print "Closed"

elif resp is None:
    print "Open|Filtered"

#elif(resp.haslayer(ICMP)):
#    if(int(resp.getlayer(ICMP).type)==3 and int(resp.getlayer(ICMP).code) in [1,2,3,9,10,13]):
#        print "Filtered"
```
##### nmap
```
nmap -sX -p 80 172.16.111.2
```
- 端口关闭：
![](./img/tcp-xmas-cls.png)
    - nmap复刻
    ![](./img/tcp-xmas-n-cls.png)
- 端口开放：
![](./img/tcp-xmas-open.png)
    - nmap复刻
    ![](./img/tcp-xmas-n-open.png)
- 端口过滤：
![](./img/tcp-xmas-fil.png)
    - nmap复刻
    ![](./img/tcp-xmas-n-fil.png)

### TCP FIN scan
> 仅发送FIN包，FIN数据包能够通过只监测SYN包的包过滤器，隐蔽性较SYN扫描更⾼，此扫描与Xmas扫描也较为相似，只是发送的包未FIN包，同理，收到RST包说明端口处于关闭状态；反之说明为开启/过滤状态。
#### code
```py
from scapy.all import *

dst_ip = "172.16.111.2" 
dst_port = 80

resp = sr1(IP(dst=dst_ip)/TCP(dport=dst_port,flags="F"),timeout=10)

if(resp.haslayer(TCP)):
    if(resp.getlayer(TCP).flags == 0x14):
        print "Closed"

elif resp is None:
    print "Open|Filtered"

#elif(resp.haslayer(ICMP)):
#    if(int(resp.getlayer(ICMP).type)==3 and int(resp.getlayer(ICMP).code) in [1,2,3,9,10,13]):
#        print "Filtered"
```
##### nmap
```
nmap -sF -p 80 172.16.111.2
```

- 端口关闭：
![](./img/tcp-fin-cls.png)
    - nmap复刻：
    ![](./img/tcp-fin-n-cls.png)
- 端口开放：
![](./img/tcp-fin-open1.png)
![](./img/tcp-fin-open2.png)
    - nmap复刻：
    ![](./img/tcp-fin-n-open.png)
- 端口过滤：
![](./img/tcp-fin-fil.png)
    - nmap复刻：
    ![](./img/tcp-fin-n-fil.png)

### TCP NULL scan
> 发送的包中关闭所有TCP报⽂头标记，实验结果预期还是同理：收到RST包说明端口为关闭状态，未收到包即为开启/过滤状态.
#### code
```py
from scapy.all import *

dst_ip = "172.16.111.2" 
dst_port = 80

resp = sr1(IP(dst=dst_ip)/TCP(dport=dst_port,flags=""),timeout=10)

if(resp.haslayer(TCP)):
    if(resp.getlayer(TCP).flags == 0x14):
        print "Closed"

elif resp is None:
    print "Open|Filtered"

#elif(resp.haslayer(ICMP)):
#    if(int(resp.getlayer(ICMP).type)==3 and int(resp.getlayer(ICMP).code) in [1,2,3,9,10,13]):
#        print "Filtered"
```

##### nmap
```bash
nmap -sN -p 80 172.16.111.2
```

- 端口关闭：
![](./img/tcp-null-cls.png)
    - namp复刻
    ![](./img/tcp-null-n-cls.png)
- 端口开放：
![](./img/tcp-null-open.png)
    - namp复刻
    ![](./img/tcp-null-n-open.png)
- 端口过滤：
![](./img/tcp-null-fil.png)
    - namp复刻
    ![](./img/tcp-null-n-fil.png)

### UDP scan
>一种开放式扫描，通过发送UDP包进行扫描。当收到UDP回复时，该端口为开启状态；否则即为关闭/过滤状态.
#### code
```py
from scapy.all import *

dst_ip = "172.16.111.2"
dst_port = 53
dst_timeout = 10

def udp_scan(dst_ip,dst_port,dst_timeout):
    resp = sr1(IP(dst=dst_ip)/UDP(dport=dst_port),timeout=dst_timeout)
    if(resp.haslayer(ICMP)):
        if(int(resp.getlayer(ICMP).type)==3 and int(resp.getlayer(ICMP).code)==3):
            return "Closed"
    elif resp is None:
        return "Open|Filtered"
#   elif(int(resp.getlayer(ICMP).type)==3 and int(resp.getlayer(ICMP).code) in [1,2,9,10,13])
#       print "Filtered"

print(udp_scan(dst_ip,dst_port,dst_timeout))
```
##### nmap
```bash
nmap -sU -p 53 172.16.111.2
```
- 端口关闭：
![](./img/udp-cls.png)
    - nmap复刻：
    ![](./img/udp-n-cls.png)
- 端口开放：
![](./img/udp-cpen.png)
    - nmap复刻：
    ![](./img/udp-n-open.png)
- 端口过滤：
![](./img/udp-fil.png)
    - nmap复刻：
    ![](./img/udp-nmap-fil.png)

## 实验总结
1.扫描方式与端口状态的对应关系

  |     扫描方式/端口状态     |              开放               |      关闭       |      过滤       |
  | :-----------------------: | :-----------------------------: | :-------------: | :-------------: |
  |  TCP connect / TCP stealth  | 完整的三次握手，能抓到ACK&RST包 | 只收到一个RST包 | 收不到任何TCP包 |
  | TCP Xmas / TCP FIN / TCP NULL |         收不到TCP回复包         |  收到一个RST包  | 收不到TCP回复包 |
  |            UDP            |          收到UDP回复包          | 收不到UDP回复包 | 收不到UDP回复包 |

2. 提供每一次扫描测试的抓包结果并分析与课本中的扫描方法原理是否相符？如果不同，试分析原因。
> 完全相符。

## 参考文献
- [c4pr1c3/cuc-ns-ppt](https://github.com/c4pr1c3/cuc-ns-ppt/blob/master/chap0x05.md)
- [Get TCP Flags with Scapy](https://stackoverflow.com/questions/20429674/get-tcp-flags-with-scapy)
- [Scapy’s documentation](https://scapy.readthedocs.io/en/latest/)
- [netstat](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/netstat)
- [Nmap Network Scanning](https://nmap.org/book/)
- [CUCCS/2020-ns-public-LyuLumos](https://github.com/CUCCS/2020-ns-public-LyuLumos)
- [CUCCS/2018-NS-Public-jckling](https://github.com/CUCCS/2018-NS-Public-jckling/blob/ns-0x05/ns-0x05/5.md)
