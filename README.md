# openvpn-on-aws-cn
Deploy OpenVPN on AWS China Region
## 免责声明
本文仅供参考和个人学习使用。

## 1.部署架构
![avatar](https://github.com/toreydai/openvpn-on-aws-cn/blob/master/images/vpn01.jpg)
## 2.创建基础环境
基础环境总体配置信息如下：
![avatar](https://github.com/toreydai/openvpn-on-aws-cn/blob/master/images/vpn02.jpg)
###  2.1 创建VPC，子网及安全组
参照Amazon VPC官方文档，创建VPC，子网及安全组
<br>https://docs.aws.amazon.com/zh_cn/vpc/latest/userguide/VPC_wizard.html

### 2.2 创建EC2实例
参照Amazon EC2官方文档，创建EC2实例
<br>https://docs.aws.amazon.com/zh_cn/AWSEC2/latest/UserGuide/EC2_GetStarted.html

### 2.3 CentOS配置国内源

```
sudo cp /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak
sudo vi /etc/yum.repos.d/CentOS-Base.repo

# CentOS-Base.repo
#
# The mirror system uses the connecting IP address of the client and the
# update status of each mirror to pick mirrors that are updated to and
# geographically close to the client.  You should use this for CentOS updates
# unless you are manually picking other mirrors.
#
# If the mirrorlist= does not work for you, as a fall back you can try the
# remarked out baseurl= line instead.

[base]
name=CentOS-$releasever - Base
baseurl=https://mirrors.tuna.tsinghua.edu.cn/centos/$releasever/os/$basearch/
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=os
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-7

#released updates
[updates]
name=CentOS-$releasever - Updates
baseurl=https://mirrors.tuna.tsinghua.edu.cn/centos/$releasever/updates/$basearch/
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=updates
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-7

#additional packages that may be useful
[extras]
name=CentOS-$releasever - Extras
baseurl=https://mirrors.tuna.tsinghua.edu.cn/centos/$releasever/extras/$basearch/
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=extras
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-7

#additional packages that extend functionality of existing packages
[centosplus]
name=CentOS-$releasever - Plus
baseurl=https://mirrors.tuna.tsinghua.edu.cn/centos/$releasever/centosplus/$basearch/
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=centosplus
gpgcheck=1
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-7

yum update -y
yum upgrade -y
```

### 2.4 VPN Server禁用源/目标检查
```
EC2控制台中选中VPN Server，操作->联网->更改源/目标检查，选择禁用。
或通过AWS CLI命令行操作：
aws ec2 modify-instance-attribute --no-source-dest-check
```
## 3 部署OpenVPN Server

### 3.1 安装openvpn
```
yum install -y epel-release
yum install -y install openvpn easy-rsa net-tools bridge-utils
```
### 3.2 创建CA和证书
```
cd /usr/share/easy-rsa/3
 ./easyrsa init-pki
./easyrsa build-ca

./easyrsa build-server-full server1 nopass
./easyrsa build-client-full client1 nopass

./easyrsa gen-dh
```
### 3.3 创建TLS-Auth Key

```
openvpn --genkey --secret ./pki/ta.key 
cp -pR /usr/share/easy-rsa/3/pki/{issued,private,ca.crt,dh.pem,ta.key} /etc/openvpn/server/ 
```
### 3.4 内核参数中开启IPv4 Forwarding
```
cd  /etc/sysctl.d/
vi 99-sysctl.conf 

追加net.ipv4.ip_forward = 1
sysctl --system 
```
### 3.5 配置OpenVPN Server

```
cp /usr/share/doc/openvpn-2.4.9/sample/sample-config-files/server.conf /etc/openvpn/server/
vi /etc/openvpn/server/server.conf

# line 32: change if need (listening port of OpenVPN)
port 1194

# line 35: change if need
;proto tcp
proto udp

# line 78: specify certificates
ca ca.crt
cert issued/server1.crt
key private/server1.key

# line 85: specify DH file
dh dh.pem

# line 101: specify network to be used on VPN
# any network are OK except your local network
server 10.8.0.0 255.255.255.0

# line 143: uncomment and change to your local network
push "route 192.168.1.0 255.255.255.0"
push "route 192.168.2.0 255.255.255.0"

# line 231: keepalive settings
keepalive 10 120

# line 244: specify TLS-Auth key
tls-auth ta.key 0
key-direction 0

# line 263: uncomment (enable compress)
comp-lzo

# line 281: enable persist options
persist-key
persist-tun

# line 287: change log path
status /var/log/openvpn-status.log
log /var/log/openvpn.log
log-append  /var/log/openvpn.log

# line 306: specify log level (0 - 9, 9 means debug lebel)
verb 3
```
### 3.6 启动OpenVPN服务并设置开机自启动

```
systemctl start openvpn-server@server
systemctl enable openvpn-server@server 
```
## 4 路由设置
### 4.1 VPN Server路由
```
vi /etc/openvpn/server/server.conf
添加需要连通的VPC2子网的路由：
push "route 192.168.2.0 255.255.255.0"
```
### 4.2 VPC2中子网路由
```
添加1条Destination为10.8.0.0/24 Target为VPN server的路由
```
## 5 客户端配置
### 5.1 客户端下载并安装
```
# win7客户端
https://my-bucket-torey.s3.cn-northwest-1.amazonaws.com.cn/openvpn/openvpn-install-2.4.9-I601-Win7.exe
# win10客户端
https://my-bucket-torey.s3.cn-northwest-1.amazonaws.com.cn/openvpn/openvpn-install-2.4.9-I601-Win10.exe
```
### 5.2 客户端配置
```
# 从vpn server中下载如下几个文件，拷贝到客户端安装目录下的config文件夹
/etc/openvpn/server/ca.crt
/etc/openvpn/server/ta.key
/etc/openvpn/server/issued/client1.crt
/etc/openvpn/server/private/client1.key 

# 从C:\Program Files\OpenVPN\sample-config拷贝client.ovpn文件到config目录下进行编辑
添加及修改如下字段
remote <vpn-server-public-ip> 1194
ca ca.crt
cert client1.crt
key client1.key 
tls-auth ta.key 1
comp-lzo
修改完成后重命名为client1.ovpn
```
### 5.3 客户端测试

打开VPN客户端，连接成功
<br>![avatar](https://github.com/toreydai/openvpn-on-aws-cn/blob/master/images/vpn04.jpg)
<br>进行网络连通性测试
<br>![avatar](https://github.com/toreydai/openvpn-on-aws-cn/blob/master/images/vpn05.jpg)
### 至此，OpenVPN通过证书认证方式已经测试成功，如需通过账号密码方式进行认证，请参考本文剩余章节。

## 6 配置使用账号密码认证方式登陆

### 6.1 修改服务器端设置
```
vi /etc/openvpn/server/server.conf
文件末尾添加如下配置：
# use username and password login
auth-user-pass-verify /etc/openvpn/checkpsw.sh via-env
client-cert-not-required
username-as-common-name
script-security 3
```
### 6.2 增加密码验证脚本
```
vi /etc/openvpn/checkpsw.sh
```
cat checkpsw.sh

```
###########################################################
# checkpsw.sh (C) 2004 Mathias Sundman <mathias@openvpn.se>
#
# This script will authenticate Open××× users against
# a plain text file. The passfile should simply contain
# one row per user with the username first followed by
# one or more space(s) or tab(s) and then the password.

PASSFILE="/etc/openvpn/psw-file"
LOG_FILE="/etc/openvpn/openvpn-password.log"
TIME_STAMP=`date "+%Y-%m-%d %T"`

###########################################################

if [ ! -r "${PASSFILE}" ]; then
  echo "${TIME_STAMP}: Could not open password file \"${PASSFILE}\" for reading." >> ${LOG_FILE}
  exit 1
fi

CORRECT_PASSWORD=`awk '!/^;/&&!/^#/&&$1=="'${username}'"{print $2;exit}' ${PASSFILE}`

if [ "${CORRECT_PASSWORD}" = "" ]; then
  echo "${TIME_STAMP}: User does not exist: username=\"${username}\", password=\"${password}\"." >> ${LOG_FILE}
  exit 1
fi

if [ "${password}" = "${CORRECT_PASSWORD}" ]; then
  echo "${TIME_STAMP}: Successful authentication: username=\"${username}\"." >> ${LOG_FILE}
  exit 0
fi

echo "${TIME_STAMP}: Incorrect password: username=\"${username}\", password=\"${password}\"." >> ${LOG_FILE}
exit 1
```
密码验证脚本增加可执行权限：

```
chmod +x /etc/openvpn/checkpsw.sh
```
### 6.3 配置账号密码文件
```
$ vim /etc/openvpn/user_passwd.txt # 编辑账号密码文件，添加以下内容
user01 p@ssw0rd1
user02 p@ssw0rd2

#修改账号密码文件访问权限
cd /etc/openvpn/
chmod 400 user_passwd.txt
chown openvpn.openvpn user_passwd.txt
```
### 6.4 修改客户端配置

修改客户端软件 OpenVPN GUI 安装路径下的config目录里名为 *.ovpn 结尾的配置文件

```
# 注释掉客户端密钥认证方式
;cert laptop.crt
;key laptop.key

# 新增账号/密码验证方式
auth-user-pass
```
### 6.5 客户端测试

使用配置好的用户名密码登陆VPN客户端
<br>![avatar](https://github.com/toreydai/openvpn-on-aws-cn/blob/master/images/vpn03.jpg)
<br>VPN连接成功
<br>![avatar](https://github.com/toreydai/openvpn-on-aws-cn/blob/master/images/vpn04.jpg)
<br>进行网络连通性测试
<br>![avatar](https://github.com/toreydai/openvpn-on-aws-cn/blob/master/images/vpn05.jpg)
## 7 配置文件参考
VPN Server配置文件server.conf，及客户端配置文件client1.ovpn，参考附件。
## 8 参考链接：
centos配置国内源：
<br>https://mirrors.cnnic.cn/help/centos/
<br>openvpn配置参考步骤：
<br>https://cloud.tencent.com/developer/article/1491801
<br>https://www.nowfox.com/article/5771
<br>openvpn配置用户名密码登录
<br>https://www.linuxops.fun/2017/05/30/f6c8d148.html
