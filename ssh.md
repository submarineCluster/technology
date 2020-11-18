## 超实用技术——SSH 的原理与应用

**SSH简介**

SSH是Secure Shell的缩写，也叫做安全外壳协议。SSH的主要目的是实现安全远程登录。



**SSH工作原理**

SSH的安全性比较好，其对数据进行加密的方式主要有两种：对称加密（密钥加密）和非对称加密（公钥加密）。

对称加密指加密解密使用的是同一套秘钥。Client端把密钥加密后发送给Server端，Server用同一套密钥解密。对称加密的加密强度比较高，很难破解。但是，Client数量庞大，很难保证密钥不泄漏。如果有一个Client端的密钥泄漏，那么整个系统的安全性就存在严重的漏洞。为了解决对称加密的漏洞，于是就产生了非对称加密。非对称加密有两个密钥：“公钥”和“私钥”。公钥加密后的密文，只能通过对应的私钥进行解密。想从公钥推理出私钥几乎不可能，所以非对称加密的安全性比较高。

SSH的加密原理中，使用了RSA非对称加密算法。
整个过程是这样的：
（1）远程主机收到用户的登录请求，把自己的公钥发给用户。
（2）用户使用这个公钥，将登录密码加密后，发送回来。
（3）远程主机用自己的私钥，解密登录密码，如果密码正确，就同意用户登录。

03



**中间人攻击**

SSH之所以能够保证安全，原因在于它采用了公钥加密，这个过程本身是安全的，但是实际用的时候存在一个风险：如果有人截获了登录请求，然后冒充远程主机，将伪造的公钥发给用户，那么用户很难辨别真伪。因为不像https协议，SSH协议的公钥是没有证书中心（CA）公证的，是自己签发的。

如果攻击者插在用户与远程主机之间（比如在公共的wifi区域），用伪造的公钥，获取用户的登录密码。再用这个密码登录远程主机，那么SSH的安全机制就不存在了。这种风险就是著名的"中间人攻击"（Man-in-the-middle attack）。那么SSH协议是怎样应对的呢？

04



**口令登录**

如果是第一次登录远程机，会出现以下提示：

```
$ ssh user@host
The authenticity of host 'host (12.18.429.21)' can't be established.
RSA key fingerprint is 98:2e:d7:e0:de:9f:ac:67:28:c2:42:2d:37:16:58:4d.
Are you sure you want to continue connecting (yes/no)?
```

<以上代码可复制粘贴，可往左滑>

因为公钥长度较长（采用RSA算法，长达1024位），很难比对，所以对其进行MD5计算，将它变成一个128位的指纹。如98:2e:d7:e0:de:9f:ac:67:28:c2:42:2d:37:16:58:4d，这样比对就容易多了。

经过比对后，如果用户接受这个远程主机的公钥，系统会出现一句提示语：

```
Warning: Permanently added 'host,12.18.429.21' (RSA) to the list of known hosts.
```

<以上代码可复制粘贴，可往左滑>

表示host主机已得到认可，然后再输入登录密码就可以登录了。

当远程主机的公钥被接受以后，它就会被保存在文件~/.ssh/known_hosts之中。下次再连接这台主机，系统就会认出它的公钥已经保存在本地了，从而跳过警告部分，直接提示输入密码。每个SSH用户都有自己的known_hosts文件，此外系统也有一个这样的文件，一般是/etc/ssh/ssh_known_hosts，保存一些对所有用户都可信赖的远程主机的公钥。

05



**公钥登录**

使用密码登录，每次都必须输入密码，非常麻烦。好在SSH还提供了公钥登录，可以省去输入密码的步骤。
所谓"公钥登录"，原理很简单，就是用户将自己的公钥储存在远程主机上。登录的时候，远程主机会向用户发送一段随机字符串，用户用自己的私钥加密后，再发回来。远程主机用事先储存的公钥进行解密，如果成功，就证明用户是可信的，直接允许登录shell，不再要求密码。

这种方法要求用户必须提供自己的公钥。如果没有现成的，可以直接用ssh-keygen生成一个： $ ssh-keygen

运行上面的命令以后，系统会出现一系列提示，可以一路回车。其中有一个问题是，要不要对私钥设置口令（passphrase），如果担心私钥的安全，这里可以设置一个。
运行结束以后，在~/.ssh/目录下，会新生成两个文件：id_rsa.pub和id_rsa。前者是公钥，后者是私钥。

这时再输入下面的命令，将公钥传送到远程主机host上面：

```
$ ssh-copy-id user@host
```

<以上代码可复制粘贴，可往左滑>

远程主机将用户的公钥，保存在登录后的用户主目录的~/.ssh/authorized_keys文件中。
这样，以后就登录远程主机不需要输入密码了。

如果还是不行，就用vim打开远程主机的/etc/ssh/sshd_config这个文件，将以下几行的注释去掉。

```
RSAAuthentication yes 　　PubkeyAuthentication yes 　　AuthorizedKeysFile .ssh/authorized_keys
```

<以上代码可复制粘贴，可往左滑>

然后，重启远程主机的ssh服务。

```
Redhat6系统
service ssh restart
Redhat7系统
systemctl restart sshd
ubuntu系统
service ssh restart
debian系统
/etc/init.d/ssh restart
```

<以上代码可复制粘贴，可往左滑>









**实战**

**生成秘钥**

```
[root@Jaking ~]# ifconfig
ens33: flags=4163mtu 1500
        inet 192.168.10.88  netmask 255.255.255.0  broadcast 192.168.10.255
        inet6 fe80::1026:b2d7:b2bc:82be  prefixlen 64  scopeid 0x20
        ether 00:0c:29:57:18:93  txqueuelen 1000  (Ethernet)
        RX packets 993461  bytes 114570794 (109.2 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 66404  bytes 45385043 (43.2 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10
[root@Jaking ~]# ssh-keygen #这里要一直按回车
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
89:86:42:59:1f:27:f1:f4:26:e8:10:bb:ae:37:e2:69 root@Jaking.localdomain
The key's randomart image is:
+--[ RSA 2048]----+
|    o +.o        |
|   o + B .       |
|  o o o o o      |
| .   = . +       |
|  . o + S        |
|   o .           |
|    .            |
|  Eoo            |
| o+o .           |
+-----------------+
[root@Jaking ~]# cat /root/.ssh/id_rsa
-----BEGIN RSA PRIVATE KEY-----
MIIEogIBAAKCAQEAuc8KoCp+dmHY99gOAt9ywPXv1YZyzdOKuDSaYZOMEp9c78sU
JDZl08LEQCSgiNKaUYZxd5XGMHkk2n9dDWKmH5NJ1KBP+Olp433A4W+BBFF71wRD
2OU8ulAwNNSsWB5Q2EXcCqHDtu9zN7fZvujPMVvVqprPqw+gXpskjRI2lG34iftf
lRJoChXSrEOJQEMwYJcps45xoy/7yOhPidpoU3BE1ojemMecL5bQTnd56eR1zjIE
pCtwNWaKm8VJfqHge/A75R60QKfv0SjsNQaddo7gqYBkj+2zbxiJY5K1WE5K4UyU
7wLzjBNZW0h/EaE73wHEKoFni8ydZ8cbjJJZhwIDAQABAoIBAARGg1QUJjzLG5b4
XborMhTGk/Ix2cpqp7J9Y2ADaSG0kQrjfV8n8UfiH2nqbdc4IVzm3w2FYL4Uy4hL
jfSU5IWtefFujuiHVmxppFqLmkhjJ5pW+siu3arb1YAhtKWCbRHM6bdE6Z/3+oq5
rET8TmgwWMZIMacaAPKsVzb3yFG5/AU4HS4V4XgmfoqEnjwrYUnySOcZKkYvoEPe
lJchN44SjrKd2MndtXRgm0GbSCbwrMj3Blmx8qutnaqzMZVIgicxu2tim6mTCWru
5SaydYQbDA3CX909qkvx4IVTYy2+6K1jfLy+ikhv3kJnivD0TAlEmJe4cR3G7zpV
kKdz1yECgYEA5Y971v7zz+GBeAhF9H2y7iUY9V3mSdWbwS2sCDXVpzwrjCYE9QGa
hbE6k5NyrUmK1GxhtWJbHUjDQMS8fvDIARJ23W3T/Y3sa6XBjN6Hq5DRyicy+0tJ
dxynEpqzFdkYt77bpcEKXhoAakpDrfrR182Wd4rk80UHdp1XlZcLIMsCgYEAzzWN
Yt2UJQ4aWRxTA0+H3NRZuzrSs8vl+i7Iw02ZsDxB39/0vCSsL0OczwdR6XK2tMvG
61Czve/8A9g/ERgFbWIGKqs777T9jgVS/JslRle4/JGCGeKZcw0msKOKqCHTYYOE
RAVZ2jPqaZZ8Gamc+TE6F5qupXhU8EB0csXpPrUCgYA5NeoqKb3/p/bJQF6W0SDf
wvUWaYF0Ez1PBp/iJ/CITjGYKv1/RhgJi6LKlqu0zihASoaLWujUQocOxDkp9b4S
rlRbWPzFKzKpnVTAU9FCC8SM+fn1sMytV8G3nEBXiJRlbrZ098gqrZY+5yU43dKg
UsdWIZJvolt6zzm9uTf3wwKBgGQo77oNf3HV+lh+v4XHKNZO8zz0tyrf8b/YY4U8
eoDc777G4+caFv0VwrO0Rx0ALV8BbZsLvIagfYJiQkICCYWRL4fqk6NQKow++JlQ
aVkySCIWN/xJM4GQptYVh420JBhr2UCEEaXPGI2Hh19kRJOT/w+v3qHvo6cqkN91
2URNAoGAGzMTIjaYBHVzdQAZT0Tb01xRXhV9BxH9WM8KPN2WH1pqxkMQ0DG0hnk9
hnC7Lv8W0kRUkDb56D+wxAyLe4GO4Zy51IGnAWGWivHmVxh6Q9ToggOiqsAGTGA/
HTGTElG7tOsXNIGu/eImgPeSKbAZ+Zi9HYNWx4SY/7OYnuwfXAM=
-----END RSA PRIVATE KEY-----
[root@Jaking ~]# cd /root/.ssh
[root@Jaking .ssh]# ls
id_rsa  id_rsa.pub  known_hosts
[root@Jaking .ssh]# cat id_rsa.pub 
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC5zwqgKn52Ydj32A4C33LA9e/VhnLN04q4NJphk4wSn1zvyxQkNmXTwsRAJKCI0ppRhnF3lcYweSTaf10NYqYfk0nUoE/46WnjfcDhb4EEUXvXBEPY5Ty6UDA01KxYHlDYRdwKocO273M3t9m+6M8xW9Wqms+rD6BemySNEjaUbfiJ+1+VEmgKFdKsQ4lAQzBglymzjnGjL/vI6E+J2mhTcETWiN6Yx5wvltBOd3np5HXOMgSkK3A1ZoqbxUl+oeB78DvlHrRAp+/RKOw1Bp12juCpgGSP7bNvGIljkrVYTkrhTJTvAvOME1lbSH8RoTvfAcQqgWeLzJ1nxxuMklmH root@Jaking.localdomain
[root@Jaking .ssh]# 
[root@Jaking .ssh]# 
[root@Jaking .ssh]# 
[root@Jaking .ssh]# ssh-copy-id root@192.168.10.10
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
root@192.168.10.10's password: 

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'root@192.168.10.10'"
and check to make sure that only the key(s) you wanted were added.
```

<以上代码可复制粘贴，可往左滑>

**验证免密登录**

```
[root@Jaking .ssh]# ssh root@192.168.10.10
Last login: Wed Nov 20 15:18:11 2019 from 192.168.10.88
[root@Jaking ~]# ifconfig
ens32: flags=4163mtu 1500
        inet 192.168.10.10  netmask 255.255.255.0  broadcast 192.168.10.255
        inet6 fe80::20c:29ff:fe84:eae5  prefixlen 64  scopeid 0x20
        ether 00:0c:29:84:ea:e5  txqueuelen 1000  (Ethernet)
        RX packets 16300  bytes 1107939 (1.0 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 13043  bytes 17924190 (17.0 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10
```

<以上代码可复制粘贴，可往左滑>

06



**SSH端口转发**

SSH端口转发有三种：动态端口转发、本地端口转发、远程端口转发。
这三种方式说起来有点难理解，通过例子会好理解一点。假设有三台主机，host1、host2、host3。

动态端口转发是找一个代理端口，然后通过代理端口去连相应的端口。动态端口转发的好处在于通过代理端口可以去找很多需要连接的端口，提高了工作效率。比如host1本来是连不上host2的，而host3却可以连上host2。host1可以找到host3作代理，然后通过host3去连接host2的相应端口

本地端口转发也是找到第三方，通过第三方再连接想要连接的端口，但这种方式的端口转发是固定的，是点对点的。比如假定host1是本地主机，host2是远程主机。由于种种原因，这两台主机之间无法连通。但是，另外还有一台host3，可以同时连上host1和host2这两台主机。通过host3，将host1连上host2。host1找到host3,host1和host3之间就像有一条数据传输的道路，通常被称为“SSH隧道”，通过这条隧道host1就可以连上host2。

远程端口转发和本地端口转发就是反过来了。假如host1在外网，host2在内网，正常情况下，host1不能访问host2。通过远程端口转发，host2可以反过来访问host1。host2和host1之间形成了一条道路，host1就可以通过这条道路去访问host2。

07



**SSH基本用法**

SSH主要用于远程登录:
假定你要以用户名user，登录远程主机host，只要一条简单命令就可以了。

```
$ ssh user@host
```

<以上代码可复制粘贴，可往左滑>

如果本地用户名与远程用户名一致，登录时可以省略用户名。

```
$ ssh host
```

<以上代码可复制粘贴，可往左滑>

SSH的默认端口是22，也就是说，你的登录请求会送进远程主机的22端口。使用p参数，可以修改这个端口。

```
$ ssh -p 2018 user@host
```



<以上代码可复制粘贴，可往左滑>

上面这条命令表示，ssh直接连接远程主机的2018端口。