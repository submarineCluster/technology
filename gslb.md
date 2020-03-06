# [GSLB概要和实现原理](https://chongit.github.io/2015/04/15/GSLB概要和实现原理/)

By [Chong Fan](http://chongit.github.io/)

 2015-04-15 更新日期:2015-05-16

**文章目录**

1. [1. What is GSLB](https://chongit.github.io/2015/04/15/GSLB概要和实现原理/#What_is_GSLB)
2. [2. Why GSLB](https://chongit.github.io/2015/04/15/GSLB概要和实现原理/#Why_GSLB)
3. \3. How implements GSLB
   1. [3.1. DNS](https://chongit.github.io/2015/04/15/GSLB概要和实现原理/#DNS)
   2. [3.2. HTTP redirection](https://chongit.github.io/2015/04/15/GSLB概要和实现原理/#HTTP_redirection)
   3. [3.3. IP Route](https://chongit.github.io/2015/04/15/GSLB概要和实现原理/#IP_Route)
   4. [3.4. 统一调度服务层](https://chongit.github.io/2015/04/15/GSLB概要和实现原理/#统一调度服务层)
4. [4. 方案的对比](https://chongit.github.io/2015/04/15/GSLB概要和实现原理/#方案的对比)
5. [5. 总结](https://chongit.github.io/2015/04/15/GSLB概要和实现原理/#总结)

## What is GSLB

**Global Server Load Balancing**

**中文:全局负载均衡**

**SLB(Server load balancing)**是对集群内物理主机的负载均衡，而**GSLB**是对物理集群的负载均衡。
这里的负载均衡可能不只是简单的流量均匀分配,而是会根据策略的不同实现不同场景的应用交付。

**GSLB**是依赖于用户和实际部署环境的互联网资源分发技术，不同的目的对应着一系列不同的技术实现。

## Why GSLB

**总结为:**

- 高可用性
- 更快的响应时间
- 多版本分发

具体：

1. Disater recovery,发生故障时提供一个备用的**位置**来获取资源或者能提供可简易调整流量的装置，或两者都能提供。

2. Load sharing,基于多个地理位置的流量分发，可以做到：

   a.尽量节省带宽
   b.限制给定位置的能力
   c.限制暴露断电,地理灾害等问题

3. Performance,将资源置于离用户更近的地方，增强用户体验。

4. 多版本,根据本地政策提供不同版本的资源，或者根据自定义的规则提供为特殊用户提供特殊版本，如灰度交付等。

## How implements GSLB

主流的技术实现

### **DNS**

GSLB会替代最终的DNS的服务器从而实现自己的解析策略，返回给用户最合适的IP(列表)。

![image](https://chongit.github.io/image/gslb//GSLB-DNS.png)

一个普通的DNS请求：

```
① 用户提交域名
② 客户端解析域名
③ DNS服务器解析出IP
④ 客户端请求IP
⑤ 返回结束
```

加入了GSLB的请求：

```
① 提交域名
② 客户端解析域名
③ NS解析到GSLB-
④ GSLB解析并返回IP
⑤ 客户端请求IP
⑥ 返回结束
```

***特点：\***

- 这个技术对原业务的侵入性最小，被商业ADC广泛实现，如A10，F5等。
- 但是可以得到的信息很有限，IP的定位只能靠Local DNS，因为得不到源IP.

### HTTP redirection

使用HTTP重定向将内容转发到不同位置.

```
a. 请求的域名均解析为GSLB机器的IP.
b. GSLB根据源IP等信息解析出新的IP并使用HTTP重定向技术将用户请求重定向到目标主机.
```

![image](https://chongit.github.io/image/gslb/http-redirection.png)

请求过程：

```
① 提交域名
② 客户端解析域名
③ DNS解析域名为GSLB
④ 客户端提交请求给GSLB服务器
⑤ GSLB解析出目标IP并发起HTTP转发
⑥ 客户读转发请求到目标IP
⑦ 返回结束
```

***特点：\***

- 这个方案只适用于HTTP.
- 这个方案的实现可以是L7负载均衡工具如Nginx、HTTPD等

### IP Route

更改IP首部实现使用跳转.并利用IP tunneling技术实现只对请求负载均衡(响应直接返回).

```
a. 请求的域名均解析为GSLB机器的IP.
b. 负载均衡设备可以解析出目标地址,然后封装IP包发给目标地址.
c. 目标服务器收到请求包并处理,解析出被封装的IP包可以得到客户端地址,把响应直接返回.
```

![image](https://chongit.github.io/image/gslb/vs-tun.jpg)

请求过程:

```
① 提交域名
② 客户端解析域名
③ DNS解析域名为GSLB-
④ 客户端提交请求给GSLB服务器
⑤ GSLB发送请求到目标服务器
⑥ 目标服务器直接返回请求给客户端结束
```

***特点\***

- 这个方案能解决不能获得源IP和HTTP Only的问题，也不会成为性能瓶颈.但由于是IP层的LB，因此得到的信息很有限，也就是做分发的策略会很有限.***

- 实现方式主要就是LVS的`VS/TUN`模式.如果自己编写会更灵活，但难度较大.***

### 统一调度服务层

客户端SDK+调度服务完成GSLB设备的功能。

```
a. 客户端使用原地址请求服务时，SDK会交付一个解析过的地址给客户端.(或对网络请求模块做Proxy)
b. SDK会通过一定的策略从调度服务中获取解析地址(一个或多个).
c. SDK和调度服务会某种形式保持联系(HTTP or TCP).
d. 出于性能考虑，SDK本身有Cache功能同时有时效限制(TTL)。
```

![image](https://chongit.github.io/image/gslb/GSLB-schedule-layer.png)

调用试请求过程:

```
① 客户端请求
② 调用SDK
③ SDK没有命中缓存
 ④ SDK请求调度服务
 ⑤ 调度服务返回新地址
 ⑥ 客户端用新地址发起请求
```

代理式请求过程:

```
① 客户端发起请求
② 网络请求Proxy拦截请求
③ Proxy没有命中缓存
④ Proxy请求调度服务
⑤ 调度服务返回新地址
⑥ Proxy请求新地址
```

***特点\***

- 让客户端(SDK)具备了负载均衡知识,而因此让服务端可以获得任何想要知道的信息，从而可以做更全面的解析策略，但侵入性是最大的。

***常见策略实现\***

1. 地理区域。地理&IP表。
2. IP权重，为每个IP分配权重，权重决定流量比例。
3. 往返时间RTT，分active RTT(请求时ping)和passive RTT(采集tcp的syn->act的时间)。
4. 业务自定义条件,如根据语言,UserID等.

## 方案的对比

**方案1：**

```
工具：使用现有的商用解决方案：F5，A10 Thunder.
优点：
  能灵活配置GSLB,满足就近选择，位置备份等基本GSLB需求。
  除了GSLB以外，还能带来对性能和安全的全解决方案，如防DDos，硬件加速SSL加解密等等。
  当我们的业务群越来越庞大时，这些需求会越来越迫切。

缺点：
  贵。
```

**方案2**

```
工具：编写HTTP转发服务或使用Nginx,HTTPD
优点：自由实现，HTTP层可获取的信息更多因此LB策略更灵活.
缺点：只能是HTTP的转发，另外可能会成为性能瓶颈.
```

**方案3**

```
工具：使用LVS的VS/TUN模式
优点：free
缺点: 能实现的LB策略只能是LVS所支持的那些，如果想自己定义似乎不可能。
```

**方案4**

```
工具：编写统一调度服务
优点：有HTTP转发的所有优点，而且不需要流量经过，因此性能要比HTTP转发高很多。
缺点：
  1. 需要客户端的参与。（基本排除了浏览器和升级困难）。稍显复杂的缓存策略。
  2. 这个服务会成为一个`移动接入层`，将会具备相当规模。(成本)。
```

## 总结

***选择方案首先选择能完全满足自己业务需求的方案.然后在从中选择`功能&性能/价格比`最高的.\***

以上四个方案，也可以分为L7层和其他.对于需要更多业务信息参与的负载均衡，则必须从7层协议入手.

方案2和方案4都可以解析7层协议的内容，其中方案4则能更灵活的实现通用的负载均衡,条件是必须有客户端的支持.因此如果能解决客户端的问题则方案4是比较好的方案，否则只能选择方案2了.

**另外也不一定选择一种方案，能有效结合这些方案会使负载均衡能力更强大.**

比如方案4无法干预浏览器的请求,这个时候就需要使用其他方案来弥补(方案1,2,3皆可).

同时还要区分动态资源和静态资源的请求，以上方案主要使针对动态资源的负载均衡,对于静态资源,CDN提供了更好的解决方案.

------

参考资料：

Citrix的NetScaler说明书： http://support.citrix.com/servlet/KbServlet/download/22506-102-671576/gslb-primer_FINAL_1019.pdf

GSLB和CND: http://blog.csdn.net/u010340143/article/details/9062213

智能DNS: http://www.cnblogs.com/peon/archive/2007/12/30/1021219.html

负载均衡: http://blog.arganzheng.me/posts/load-balance.html

LVS: http://www.linuxvirtualserver.org/zh/lvs1.html