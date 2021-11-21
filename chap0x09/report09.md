# 第九章 入侵检测

## 实验目的
- 使用 Snort、Suricata 和 Guardian 工具体验入侵检测
- 使用 Suricata 代替 Snort ，重复`本实验`。
- 配置 Suricata 为 IPS 模式，重复`实验四` 。

## 实验环境
- 拓扑图</b>![](./img/top.png)

## 实验过程
### 安装
- Snort
```bash
# 禁止在apt安装时弹出交互式配置界面
export DEBIAN_FRONTEND=noninteractive

sudo apt install snort
```
- Suricata
```bash
sudo apt-get install suricata
```

### 一、配置snort为嗅探模式


#### Snort
```bash
# -b 参数表示报文存储格式为 tcpdump 格式文件
# -q 静默操作，不显示版本欢迎信息和初始化信息
snort -q -v -b -i eth3
```
- `Kali-Victim` ping `Kali-attacker`,`Kali-attacker`显示相关信息</b>![](./img/snort1.png)
#### Suricate
```bash
suricata -v -i eth3
```
- `Kali-Victim` ping `Kali-attacker`,`Kali-attacker`显示接收到相关包![](./img/suricata1.png)
### 二、配置并启用snort内置规则

#### Snort
```bash
# /etc/snort/snort.conf 中的 HOME_NET 和 EXTERNAL_NET 需要正确定义
# 例如，学习实验目的，可以将上述两个变量值均设置为 any
# -A         Set alert mode: fast, full, console, test or none  (alert file alerts only)
#                    "unsock" enables UNIX socket logging (experimental).
# -c <rules> Use Rules File <rules>
# -l <ld>    Log to directory <ld>
snort -q -A console -b -i eth3 -c /etc/snort/snort.conf -l /var/log/snort/
```
![](./img/snort2-set.png)

#### Suricate
```bash
suricata -i eth3 -c /etc/suricata/suricata.yaml
```
![](./img/suricata2-2.png)

### 三、自定义snort规则

#### Snort
```bash
# 新建自定义 snort 规则文件
cat << EOF > 
alert icmp $EXTERNAL_NET any -> $HOME_NET any (msg:"ICMP Large ICMP Packet"; dsize:>800; reference:arachnids,246; classtype:bad-unknown; sid:499; rev:4;)
EOF
```
![](./img/snort3-diy.png)
```bash
# 添加配置代码到 /etc/snort/snort.conf
include $RULE_PATH/cnss.rules
```
![](./img/snort3-include.png)
- `Kali-Victim1` 执行 `ping -s 900 172.16.111.2`,`Kali-Attacker`发生警报</b>![](./img/snort3-ping.png)

#### Suricate
```bash
cat << EOF > /etc/suricata/rules/test.rules
alert icmp 172.16.111.1 any <> 172.16.111.2 any (msg:"Informing ICMP Packet from 172.16.111.1";sid:1000001;rev:1;)
EOF

# 添加配置代码到 /etc/suricata/suricata.yaml
 - test.rules

suricata -i eth3 -c /etc/suricata/suricata.yaml -s /etc/suricata/rules/test.rules
```
![](./img/suricata3-add.png)

- `Kali-Victim1` 执行 `ping 172.16.111.2`,`Kali-Attacker`出现对应警报
![](./img/suricata3-log.png)


### 四、和防火墙联动

#### Snort
- 在`Kali-Attacker`上
```bash
# 获取脚本代码
wget https://c4pr1c3.github.io/cuc-ns/chap0x09/attach/guardian.tar.gz
# 解压缩 Guardian-1.7.tar.gz
tar zxf guardian.tar.gz
# 安装 Guardian 的依赖 lib
apt install libperl4-corelibs-perl
# 开启 snort
snort -q -A fast -b -i eth3 -c /etc/snort/snort.conf -l /var/log/snort/
```
- 编辑`guardian.conf`并保存，确认以下2个参数的配置符合主机的实际环境参数</b>![](./img/snort4-conf.png)
- `perl guardian.pl -c guardian.conf` 启动 `guardian.pl`,并在`Kali-Victim1`中用nmap暴力扫描
```bash
nmap 172.16.111.2 -A -T4 -n -vv
```
![](./img/snort4-nmap.png)
- 记录`Kali-Attacker`iptables的变化
```shell
(root💀kali)-[/home/kali_attacker/Desktop/guardian]
└─# iptables -L -n
Chain INPUT (policy ACCEPT)
target     prot opt source               destination
REJECT     tcp  --  172.16.111.1       0.0.0.0/0            reject-with tcp-reset
DROP       all  --  172.16.111.1       0.0.0.0/0

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination

# 1分钟后，guardian.pl 会删除刚才添加的2条 iptables 规则
(root💀kali)-[/home/kali_attacker/Desktop/guardian]
└─# iptables -L -n
Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
```
#### Suricate
- 添加规则
```bash
vim /etc/snort/rules/my.rules
alert tcp 172.16.111.1 any -> 172.16.111.2 any (msg:"Informing ICMP Packet from 172.16.111.1";sid:1000002;rev:1;)
alert udp 172.16.111.1 any -> 172.16.111.2 any (msg:"Informing ICMP Packet from 172.16.111.1";sid:1000003;rev:1;)
```
</b>![](./img/suricata4-diy.png)</b>![](./img/suricata4-yaml.png)

```bash
# 开启suricata
suricata -i eth3 -c /etc/suricata/suricata.yaml
# 启动 guardian.pl
perl guardian.pl -c guardian.conf
```
- 查看日志，观察到相应的alert
![](./img/suricata4-log.png)
- 开启guardian后，nmap扫描不出Port信息，同时也扫不出OS信息
![](./img/suricata4-after.png)

### 五、配置 Suricata 为 IPS 模式，重复`实验四`
- 检查是否支持NFQ
```bash
suricata --build-info
```
![](./img/suricata5-build.png)
- 设置iptables的 NFQUEUE
```bash
sudo iptables -I INPUT -p tcp -j NFQUEUE 
sudo iptables -I OUTPUT -p tcp -j NFQUEUE
```
  
- 开启suricata
```bash
suricata -i eth3 -c /etc/suricata/suricata.yaml -q 0
```

- 启动 guardian.pl
```bash
perl guardian.pl -c guardian.conf
```
- 结果</b>![](./img/suricata5-nmap.png)
  - nmap扫描时，端口探测包全部被过滤


## 实验思考题
> IDS与防火墙的联动防御方式相比IPS方式防御存在哪些缺陷？是否存在相比较而言的优势？

- IPS属于**主动防御**，根据设置的过滤器分析相对应的数据包，通过检查的数据包可以继续前进，包含恶意内容的数据包就会被丢弃，被怀疑的数据包需要接受进一步的检查。
- IDS与防火墙联动属于**被动防御**，检测规则的更新落后于攻击手段的更新。IDS检测而不阻断任何网络行为。
  - 如本实验中，根据日志信息来对新的攻击方式更新防御规则，却不对危险行为做出阻断。
- IPS基于主动响应和过滤功能，可检测到传统的防火墙+IDS 方案检测不到的攻击行为。


## 参考资料
- [Suricata User Guide](https://suricata.readthedocs.io/en/suricata-5.0.0/index.html)
- [Writing Snort Rules](http://manual-snort-org.s3-website-us-east-1.amazonaws.com/node27.html)
- [snort_manual](https://snort-org-site.s3.amazonaws.com/production/document_files/files/000/000/249/original/snort_manual.pdf?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAU7AK5ITMGOEV4EFM%2F20211119%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20211119T004151Z&X-Amz-Expires=172800&X-Amz-SignedHeaders=host&X-Amz-Signature=7012967a32ee571fa346a6f52d0f7331cf8a269146e3c89f4609c5d06f0d156b)
- [IDS  vs. IPS: Definitions, Comparisons & Why You Need Both](https://www.okta.com/identity-101/ids-vs-ips/)
- [IDS与IPS功能分析](https://blog.51cto.com/chenguang/73767)
- [CUCCS/2019-NS-Public-chencwx](https://github.com/CUCCS/2019-NS-Public-chencwx/blob/ns_chap0x09/ns_chapter9/%E5%85%A5%E4%BE%B5%E6%A3%80%E6%B5%8B.md)