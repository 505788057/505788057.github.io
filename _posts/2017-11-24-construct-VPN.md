---
layout:     post
title:      "利用OpenVPN搭建简单VPN"
subtitle:   "Ubuntu16.04搭建VPN"
date:       2017-11-23 23:00
author:     "Yezh"
header-img: "img/post-bg-unix-linux.jpg"
catalog:      true
published:    true
tags:
    - VPN
---

## 安装OpenVPN

```
$ sudo apt-get install openvpn easy-rsa
```

安装完毕后, 在终端输入 `openvpn` 即可以看到一系列用法提示。

注: 此处同时安装了 easy-rsa 脚本工具, 用来生成所需证书。

---

## 生成OpenVPN相关证书

### 创建CA目录

选择一个目录, 在该目录下建立 `openvpn-ca` 目录(命名随意, 易懂即可), 并将上一步安装好的easy-rsa的所有文件从默认安装目录中复制到刚刚建立的CA目录下。

```
$ mkdir openvpn-ca
$ cp -r /usr/share/easy-rsa/* ./openvpn-ca/
$ cd openvpn-ca/
```

完成后进入 openvpn-ca 目录, 进行生成相关证书所需的操作。




### 配置CA变量
用文本编辑器打开当前目录下的 `vars` 文件, 会发现里面是一个shell脚本, 大致浏览一遍后就知道该文件其实就是一个配置文件, 里面导出了许多生成证书所需的环境变量。

接下来对其中的部分变量进行修改, 以生成所需证书。

```shell
# These are the default values for fields
# which will be placed in the certificate.
# Don't leave any of these fields blank.
export KEY_COUNTRY="US"
export KEY_PROVINCE="CA"
export KEY_CITY="SanFrancisco"
export KEY_ORG="Fort-Funston"
export KEY_EMAIL="me@myhost.mydomain"
export KEY_OU="MyOrganizationalUnit"

# X509 Subject Field
export KEY_NAME="EasyRSA"
```

将上面的变量值修改为与你对应的信息, 注意不能为空。

下面为我的修改:
```shell
export KEY_COUNTRY="CN"
export KEY_PROVINCE="GD"
export KEY_CITY="Guangzhou"
export KEY_ORG="SYSU"
export KEY_EMAIL="874244887@qq.com"
export KEY_OU="SDCS"

export KEY_NAME="server"
```
配置完成后, 保存并在终端输入 `source vars`, 使刚刚配置的环境变量生效。
无误的话, 应该会看到与下面类似的提示信息:
```
$ source vars
NOTE: If you run ./clean-all, I will be doing a rm -rf on /home/james/Study/webSecurity/openvpn/server/openvpn-ca/keys
```


### 生成CA证书
在生成证书前, 先执行当前目录下的 clean-all 文件, 清空所有的keys(如果有的话)

`$ ./clean-all`

1. 生成OpenVPN的根证书

    `$ ./build-ca`

    执行build-ca文件, 生成根证书颁发机构密钥（rootcertificate authority key ）和证书（certificate）

    注: 由于我们刚刚修改了vars文件，证书生成所需变量的值都会自动填充，因此, 运行时只需按回车确认即可。

    输出内容如下：

    ```
    $ ./build-ca
    Generating a 2048 bit RSA private key
    ...........+++
    ....+++
    writing new private key to 'ca.key'
    -----
    You are about to be asked to enter information that will be incorporated
    into your certificate request.
    What you are about to enter is what is called a Distinguished Name or a DN.
    There are quite a few fields but you can leave some blank
    For some fields there will be a default value,
    If you enter '.', the field will be left blank.
    -----
    Country Name (2 letter code) [CN]:
    State or Province Name (full name) [GD]:
    Locality Name (eg, city) [Guangzhou]:
    Organization Name (eg, company) [SYSU]:
    Organizational Unit Name (eg, section) [SDCS]:
    Common Name (eg, your name or your server's hostname) [SYSU CA]:zhengshu
    Name [server]:
    Email Address [874244887@qq.com]:
    ```
    执行完成后, 可以看到当前目录下多了一个keys目录, 其中存有我们刚刚生成的根证书。

2. 生成服务端所需证书

    `$ ./build-key-server server`

    执行build-key-server文件, 指定服务端证书文件名为 server

    执行过程中一路回车，中间出现 challenge password ，不要输入任何值, 也是按回车，到最后会有两个问题，输入y即可。

    输出内容很长, 此处就不粘贴出来了。

    执行完成后, 可以看到keys目录下新增了server.crt、server.key 和 server.csr三个文件。(其中server.crt和server.key两个文件是我们所需要的)



4. 生成客户端所需证书

    `$ ./build-key client1`

    执行build-key文件, 指定客户端证书文件名为 client1

    执行过程所需操作与生成服务端证书时相同。

    同样, 执行完成后, 可以看到keys目录下新增了client1.crt、client1.key 和 client1.csr三个文件。


5. 生成Diffle Hellman参数

    `$ ./build-dh`

    执行build-dh文件, 为服务端生成加密交换时所需的Diffie-Hellman文件
    执行可能需要较长时间, 完成后可在keys目录下看到新增的dh2048.pem文件。

6. 生成HMAC签名

    `$ openvpn --genkey --secret keys/ta.key`

    执行上述指令生成一个HMAC签名, 以增强服务端的TLS完整性验证能力。

    注: `keys/ta.key` 表示将生成的ta.key(一般这样命名)放到keys目录下。

至此, 所需的CA证书等文件就都生成好了。 接下来, 进行OpenVPN的具体配置。

---

## OpenVPN服务端配置
1. 复制相关文件到OpenVPN目录下

    先将之前生成的相关文件复制到 `/etc/openvpn` 配置目录下

    ```
    $ cd keys/
    $ sudo cp ca.crt ca.key server.crt server.key ta.key dh2048.pem /etc/openvpn/
    ```

    然后将OpenVPN自带的配置模板也复制到该目录下并解压

    ```
    $ sudo cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz /etc/openvpn/

    $ sudo gzip -d /etc/openvpn/server.conf.gz
    ```

2. 修改OpenVPN服务端配置文件

    用文本编辑器打开 `/etc/openvpn` 目录下的 server.conf 文件(以root权限), 对其进行如下修改:

    - 搜索 tls-auth (找到HMAC部分), 移除其开头的";"，来解注释tls-auth，并且在tls-auth的下一行增加一个key-direction参数，参数值设为0，如下所示：

        ```
        tls-auth ta.key 0 # This file is secret
        key-direction 0
        ```

    - 搜索cipher(找到加密部分)，这里我选择的是 AES-128-cbc 加密算法。移除其开头的";", 启用该加密算法, 并且在下一行添加一个auth行（身份验证行）来选择HMAC消息摘要算法，此处使用的是SHA256消息摘要算法, 如下所示：

        ```
        cipher AES-128-CBC
        auth SHA256
        ```

    - 找到user和group参数，去除它们开头的";"，如下所示：

        ```
        user nobody
        group nogroup
        ```

---

## 网络配置

为了让OpenVPN服务端可以正确路由流量, 我们需要进行一些网络配置。

1. 允许IP转发

    为了提供最基本的VPN服务, 我们需要允许服务端所在主机允许IP转发, 即能让服务端转发流量。

    以root权限打开 `/etc/sysctl.conf` 文件, 搜索net.ipv4.ip_forward, 移除开头的"#", 来启用IP转发, 改完后保存退出。

    为了让刚刚的修改生效, 还需要执行如下指令:

    `$ sudo sysctl -p`

    执行后应该会看到如下输出:

    ```
    $ sudo sysctl -p
    net.ipv4.ip_forward = 1
    ```


2. 配置防火墙规则

    接着, 我们需要配置防火墙规则来引导进入服务器的流量。

    具体配置前, 我们需要确定当前服务器的**公共网络接口**(即网卡名称), 执行如下指令:

    `$ ip route | grep default`

    可以看到终端中输出了一行内容, 而我们需要的接口应该是紧跟在 "dev" 后面的字符串

    例如, 我的输出如下, 则我的公共网络接口为 `enp2s0`

    ```
    $ ip route | grep default
    default via 172.18.159.254 dev enp2s0  proto static  metric 100
    ```

    知道接口后, 我们就可以进行具体的UFW规则配置(为NAT表设置POSTROUTING默认规则)，从而为来自VPN的任何流量设置伪装连接。

    以root权限打开 `/etc/ufw/before.rules` 文件, 并在文件开头加入部分内容, 如下所示:

    ```shell
    #
    # rules.before
    #
    # Rules that should be run before the ufw command line added rules. Custom
    # rules should be added to one of these chains:
    #   ufw-before-input
    #   ufw-before-output
    #   ufw-before-forward
    #

    # START OPENVPN RULES
    # NAT table rules
    *nat
    :POSTROUTING ACCEPT [0:0]
    # Allow traffic from OpenVPN client to enp2s0(changeto the interface you discovered!)
    -A POSTROUTING -s 10.8.0.0/8 -o enp2s0 -jMASQUERADE
    COMMIT
    # END OPENVPN RULES

    # Don't delete these required lines, otherwise there will be errors
    *filter
    ```

    **注意:** 公共网络接口要用你刚刚查到的自己的网卡名称

    添加后保存退出。

    最后, 我们让防火墙默认允许转发包:

    以root权限打开 `/etc/default/ufw` 文件, 搜索 DEFAULT_FORWARD_POLICY, 并将其值从DROP改为ACCEPT, 如下所示:

    `DEFAULT_FORWARD_POLICY="ACCEPT"`

    修改后保存退出。


3. 打开OpenVPN端口并使变化生效

    最后, 我们需要对防火墙进行调整以允许流量到OpenVPN服务端。

    配置防火墙允许UDP流量到1194端口(OpenVPN服务端默认使用端口):

    `$ sudo ufw allow 1194/udp`

    启动防火墙, 并查看当前防火墙状态:

    ```
    $ sudo ufw enable
    在系统启动时启用和激活防火墙
    $ sudo ufw status
    状态： 激活

    至                          动作          来自
    -                          --          --
    1194/udp                   ALLOW       Anywhere
    1194/udp (v6)              ALLOW       Anywhere (v6)
    ```

至此, 服务端就可以成功获取到流量并对其进行进一步处理了。

---

## OpenVPN客户端配置

1. 创建客户端配置目录

    选择一个目录, 在该目录下建立 `client-configs/keys` 目录 (命名随意, 易懂即可)

    `$ mkdir -p client-configs/keys`

    由于客户端的密钥会放到这个目录下, 所以我们有必要对该目录设置访问权限:

    `$ chmod 700 ./client-configs/keys`

2. 复制并修改客户端配置文件

    将OpenVPN自带的客户端配置模板复制到我们刚刚创建的客户端配置目录中, 作为客户端的配置文件：

    `$ cp /usr/share/doc/openvpn/examples/sample-config-files/client.conf ./client-configs/client.ovpn`

    然后对该文件进行如下修改:

    - 搜索 hostname/IP, 将其下面的remote指令按如下格式修改:

        `remote your_server_IP_address 1194`

        注: your_server_IP_address 为OpenVPN服务端所在的公网IP地址。

    - 搜索 proto, 确保选择的传输协议与在server配置文件中的一致(此处选择的是udp), 即:

        `proto udp`

    - 搜索 user 和 group, 移除它们开头的";", 如下所示:

        ```
        user nobody
        group nogroup
        ```

    - 搜索 SSL/TLS parms, 将其下面的证书及密钥路径进行修改, 如下所示:

        ```
        ca ./keys/ca.crt
        cert ./keys/client1.crt
        key ./keys/client1.key
        ```

        注: 若前面创建目录时与我不同, 则需按照你证书和密钥的实际存放路径进行书写

---

## 启动服务端

通过如下指令启动服务端:

`$ openvpn --config /etc/openvpn/server.conf`

运行上述指令尝试启动服务端, 但运行失败, 出现错误:

```
$ openvpn --config /etc/openvpn/server.conf
Options error: --dh fails with 'dh2048.pem': No such file or directory
Options error: --cert fails with 'server.crt': No such file or directory
Options error: --key fails with 'server.key': No such file or directory
Options error: --tls-auth fails with 'ta.key': No such file or directory
Options error: Please correct these errors.
Use --help for more information.
```

由报错的信息可以知道应该是配置文件中之前生成的证书等文件的路径没有指明, 而找不到文件。

用root权限打开server.conf文件, 搜索并修改相应位置的内容, 具体如下表所示:

| 修改前            | 修改后                         |
| ----------------- | ------------------------------ |
| ca ca.crt         | ca /etc/openvpn/ca.crt         |
| cert server.crt   | cert /etc/openvpn/server.crt   |
| key server.key    | key /etc/openvpn/server.key    |
| tls-auth ta.key 0 | tls-auth /etc/openvpn/ta.key 0 |
| dh dh2048.pem     | dh /etc/openvpn/dh2048.pem     |

注: 此处也可以不修改, 但是需要在 `/etc/openvpn/` 目录下运行上述启动服务端的指令。

再次运行指令, 发现还是会报错:
```
$ openvpn --config /etc/openvpn/server.conf
Options error: --key fails with '/etc/openvpn/server.key': Permission denied
Options error: --tls-auth fails with '/etc/openvpn/ta.key': Permission denied
Options error: Please correct these errors.
Use --help for more information.
```

从报错信息中可以看出是权限问题

用root权限执行指令, 可成功运行,
最后可以看到 `Initialization Sequence Completed` 信息, 表示服务端已启动。

---

## 启动客户端

在client.ovpn文件所在目录下, 运行如下指令, 启动客户端:

`$ openvpn --config client.ovpn`

运行 ifconfig 指令后, 可看到新建立的虚拟网卡tun0, 可见客户端已成功连接上服务端。

![tun0](/img/in-post/2017-11-24-construct-VPN/markdown-img-paste-20171123232833886.png)

---

至此, 我们利用OpenVPN, 已经成功搭建了一个简单的VPN了。
