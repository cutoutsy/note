### Kerberos
Kerberos名词来源于希腊神话“三个头的狗——地狱之门守护者”，系统设计上采用客户端/服务器结构与DES加密技术，并能够相互认证，即客户端和服务器端均可对对方进行身份认证。可以用于防止窃听、防止replay攻击、保护数据完整性等场合，是一种应用对称密钥体制就行密钥管理的系统。支持SSO。

开发地:MIT

目标：Kerberos是一种网络认证协议，是通过密钥系统为客户机/服务器应用程序提供强大的认证服务。

### 认证过程
客户机向认证服务器（AS）发送请求，要求得到某服务器的证书，然后AS的响应包含这些用客户端密钥的证书。证书构成为：1）服务器“ticket”; 2）一个临时加密密钥（又称为会话密钥“session key”）。客户机将ticket(包括用服务器密钥加密的客户机身份和一份会话密钥的拷贝)传送到服务器上。会话密钥可以（现已经由客户机和服务器共享）用来认证客户机或服务器，也可用来为通信双发以后的通讯提供加密服务，或通过交换独立子会话密钥为通信双方提供进一步的通信加密服务。

上述认证交换过程需要只读方式访问kerberos数据库。但有时，数据库中的记录必须进行修改，如添加新的规则或改变规则密钥是。修改过程通过客户机和第三方kerberos服务器（kerberos管理器KADM）间的协议完成。


### 工作工程
> * 假设你要在一台电脑上访问另一个服务器（你可以发送telnet或类似的登录请求）。你知道服务器要接受你的请求必须要有一张Kerberos的“入场券”。
> * 要得到这张入场券，你首先要向验证服务器（AS）请求验证。验证服务器会创建基于你的密码（从你的用户名而来）的一个“会话密钥”（就是一个加密密钥），并产生一个代表请求的服务的随机值。这个会话密钥就是“允许入场的入场券”。
> * 然后，你把这张允许入场的入场券发到授权服务器（TGS）。TGS物理上可以和验证服务器是同一个服务器，只不过它现在执行的是另一个服务。TGS返回一张可以发送给请求服务的服务器的票据。
> * 服务器或者拒绝这张票据，或者接受这张票据并执行服务。
> * 因为你从TGS收到的这张票据是打上时间戳的，所以它允许你在某个特定时期内（一般是八小时）不用再验证就可以使用同一张票来发出附加的请求。使这张票拥有一个有限的有效期使其以后不太可能被其他人使用。

### 缺陷
kerberos要求参与通信的主机的时钟同步。票据具有一定的有效期，因此，如果主机的时钟与kerberos服务器的时钟不同步认证会失败。默认设置要求时钟的时间差不超过10分钟。

### Kerberos安装及使用

OS： Centos 7.2.1511
Yum源： 阿里源

#### 选择一个主机来运行KDC
安装krb-5libs, krb5-server, krb5-auth-dialog
```shell
yum install krb5-server krb5-libs krb5-auth-dialog
```
软件安装完成后，会在KDC主机上生成配置文件/etc/krb5.conf和/var/kerbero/krb5kdc/kdc.conf。分别反映了realm name以及domain-to-realm mappings。

#### 配置kdc.conf
配置示例：
```shell
kdc_ports = 88
kdc_tcp_ports = 88
 
[realms]
EXAMPLE.COM = {
#master_key_type = aes256-cts
acl_file = /var/kerberos/krb5kdc/kadm5.acl
dict_file = /usr/share/dict/words
admin_keytab = /var/kerberos/krb5kdc/kadm5.keytab
max_renewable_life = 7d
supported_enctypes = aes128-cts:normal des3-hmac-sha1:normal arcfour-hmac:normal des-hmac-sha1:normal des-cbc-md5:normal des-cbc-crc:normal
```
EXAMPLE.COM:是设定derealms。名字随意。Kerberos可以支持多个realms，会增加复杂度。大小写敏感，一般为了识别使用全部大写。这个realms跟机器的host没有多大关系。
max_renewable_life = 7d: 涉及到是否能进行ticket的renew。必须配置。
master_key_type: 和supported_enctypes默认使用aes256-cts。由于，JAVA使用aes256-cts验证方式需要安装额外的Jar包，推荐不使用。
acl_file: 标注了admin的用户权限。文件格式是Kerberos_principal permissions [target_principal] [restrictions]支持通配符等。
admin_keytabb: KDC进行校验的keytab。
supported_enctypes: 支持验证方式。注意把aes256-cts去掉。

#### 配置krb5.conf
该配置文件包含Kerberos的配置信息。例如，KDC位置，Kerberos的admin的reams等。需要所有使用Kerberos的机器上的配置文件都同步。
配置示例：
```shell
[logging]
default=FILE:/var/log/krb5libs.log
kdc = FILE:/var/log/krb5kdc.log
admin_server = FILE:/var/log/kadmind.log
 
[libdefaults]
default_realm = EXAMPLE.COM
dns_lookup_realm = false
dns_lookup_kdc = false
ticket_lifetime = 24h
renew_lifetime = 7d
forwardable = true
# udp_preference_limit = 1
 
[realms]
HADOOP.COM = {
kdc = kerberos.com
admin_server = kerberos.com
}
 
[domain_realm]
.exmaple.com = EXAMPLE.COM
example.com = EXAMPLE.COM
```
[libdefaults]:每种链接的默认配置，需要注意以下几个关键的小配置
default_realm = EXAMPLE.COML： 默认的realm, 必须跟要配置的realm的名称一致。
ticket_lifetime: 表明凭证生效的时限，一般为24小时。
renew_lifetime: 表明凭证最长可以被延期的时限，一般为一周。当凭证国企之后，对安全认证的服务的后续访问则会失败。
[realms]: 列举使用的realm
kdc: kdc的位置。格式： 机器：端口
admin_server: admin的位置。格式L： 机器：端口
[default_domain]: 代表默认的域名

#### 创建/初始化Kerberos database
完成上面两个配置文件后，就可以初始化并启动了。
```shell
/usr/sbin/kdb5-util create -s -r EXAMPLE.COM
```
-s： 生成stash file。并在其中存储master server kec(krb5kdc)；
-r: 指定一个realm name。当krb5.conf中轻易了多个realm时才是必要的。
在此过程中，要求输入database的管理密码，这里设置的密码很重要，忘记就无法管理Kerberos server。

当Kerberos database创建好后，可以看到目录/var/kerberos/krb5kdc下生成了几个文件：
```shell
-rw------- 1 root root   22 Dec  7 01:26 kadm5.acl
-rw------- 1 root root  434 Feb 16 14:51 kdc.conf
-rw------- 1 root root 8192 Feb 14 13:44 principal
-rw------- 1 root root 8192 Feb 14 13:44 principal.kadm5
-rw------- 1 root root    0 Feb 14 13:44 principal.kadm5.lock
-rw------- 1 root root    0 Feb 16 15:31 principal.ok
```

#### 添加database administrator
我们需要为Kerberos database添加administrative principals(即能够管理database的principals),至少要添加1个principal来使得Kerberos的管理进程kadmin能够在网络上与程序kadmin进行通讯。

在master KDC上执行：
```shell
/usr/sbin/kadmin.local -q "addprinc admin/admin"
```
并设置密码

```
kadmin.local
```
可以直接运行在master KDC上，而不需要首先通过Kerberos的认证，实际上他只需要对本地文件的读写权限。

#### 为database administrator设置ACL权限
/var/kerberos/krb5kdc/kadm5.acl。Kerberos的kadmin daemon会使用该文件来管理对Kerberos database的访问权限。

为administrator设置权限：编辑内容为：
```shell
*/admin@CUTOUTSY.COM    *
```
*代表全部权限

#### 启动Kerberos daemons
```shell
service krb5kdc start
service kadmin start
```

### 参考文献
百度百科
http://www.cnblogs.com/xiaodf/p/5968178.html