# 6lowpan简介 #

如何使传感器网络与互联网互通互联一直是无线传感网络技术走向商用面临的一大挑战。大量的传感器节点接入互联网需要大量的IP地址，显然在IPv4地址逐渐枯竭的情况下，人们便将目光聚焦在了IPv6上。基于IEEE 802.15.4实现IPv6通信的6lowpan(IPv6 over Low power WPAN) 草案的发布有望解决传感器网络与互联网互通的问题。

IETF于2004年11月成立了6lowpan工作组，研究了IPv6协议如何在IEEE 802.15.4上进行传输。该工作组针对6lowpan发表了多篇RFC，其中RFC4919阐述了6lowpan的概况，需解决的问题，目标等；RFC4944对6lowpan适配层协议格式做了定义；RFC6282对IPv6数据包的压缩进行了定义；RFC6568阐述了6lowpan的应用场景；RFC6606阐述了6lowpan路由面对的问题和要求；RFC6775阐述了6lowpan的邻居发现协议。

# 6lowpan协议栈支持 #

6lowpan协议在许多开源软件上有了实现，最为出名的当属瑞士计算机科学院开发的Contiki上的6lowoan实现与加州大学伯克利分校开发的TinyOS上的6lowpan实现。

Contiki早在2011年发布的Contiki 2.5版本中就实现了6lowpan/RPL协议,之后发布的2.6,2.7版本也不断对6lowpan/RPL存在的Bug进行了修复，[见原文](http://www.contiki-os.org/download.html)。

TinyOS于2012年8月发布TinyOS 2.12，该版本中完整实现了6lowpan/RPL协议栈，[见原文](http://www.tinyos.net/)。

除此之外，像[OpenWSN](https://openwsn.atlassian.net/wiki/display/OW/Home),[RIOT](http://riot-os.org/#features),[JenNet-IP](http://www.jennic.com/products/protocol_stacks/jennet-ip)等也实现了6lowpan协议栈。

# 6lwopan概况 #

## IEEE 802.15.4标准概况

IEEE 802.15.4协议旨在制定一个低成本、低功耗、低复杂度、具备自组网能力的无线个域网标准，制定了物理层和媒体接入层规范，是目前无线传感网通信标准的最佳选择。

IEEE 802.15.4支持包括全功能设备（FFD）和部分功能设备（RFD）两种网络设备和两种网络拓扑结构：星形拓扑结构、点对点拓扑结构。其具有如下的特点：

- 支持不同的工作频段和数据传输率
	
	802.15.4协议在不同的工作频率下提供不同的数据速率：250kbits/s(2.4GHz),40kbits/s(915MHz),20kbit/s(868MHz)
	
- 允许传输的报文长度较短

	MAC层允许的最大报文长度为127字节，除去报头的25字节和安全相关的21字节后，提供给上层报文长度仅剩下81字节，而IPv6报头就包含了40字节，因而在6lowpan研究中，IPv6数据包的压缩工作必不可少	

- 对设备要求低

	802.15.4网络的设备主要是单片机、传感器等，他们的运算能力十分有限，因而802.15.4协议就必须设计的较为简单。802.15.4协议包括了49个参数，包括了14个物理层参数和35个介质控制参数，同时还将设备分为全功能设备和部分功能设备，部分功能设备仅需支持38个参数
	
- 低功耗  

	802.15.4网络中大部分节点是用电池供电的，而且不需要经常更换电池，因而就需要低功耗的支持。星形拓扑中RFD设备不能直接通信，而是通过FFD设备进行转发，这样RFD的路由成本就大大降低了

## 6lwopan需解决的问题

该问题的讨论主要在RFC4919中，因为IEEE 802.15.4的网络特性使得IPv6不能直接构建在802.15.4网络上，所以6lowpan必须解决如下问题：

- 可用的IP连接
	
	IPv6巨大的地址空间和无状态地址自动配置使得传感器节点很容易方便接入互联网，但是由于报文长度和能量方面限制，IPv6无法直接应用于IEEE 802.15.4网络

- 路由协议
	
	6lowpan需支持包括mesh和start在内的拓扑结构，使用mesh拓扑时，报文可能在多跳网络中进行路由，而802.15.4网络中的节点很难维护IPv6中庞大的路由表，因而需要更为简单的路由协议，对运算能力和存储空间要求更小。（为了研究该问题，IETF在2008年成立了新的工作组[ROLL](https://datatracker.ietf.org/doc/rfc6550/)，该工作组制定了RPL协议）

- 组播限制
	
	邻居发现协议许多功能需依赖于IP组播，而802.15.4网络的不稳定信道不能保证所有的节点都能收到IPv6组播报文

- 有限的配置和管理

	由于IEEE 802.15.4网络中设备功能简单，因此6lowpan的网络配置和初始化功能应该要尽量简单

- 安全性
	
	IEEE 802.15.4提供基于AES的链路层安全支持，然而并没有定义初始化、密钥管理、上层安全性等细节，这是6lowpan需要考虑的。

## 6lowpan解决方案

### 6lowpan适配层与报文格式

为了使IPv6报文能在IEEE 802.15.4网络上传输，在MAC层和IP层之间加入了适配层，6lowpan协议栈如下图所示：

由于在报文格式上做了一定的修改，在MAC层头部之后加入了适应层头部，适应层头部包含了几种类型，当IPv6报文要在IEEE 802.15.4链路上传输时，IPv6报文必须封装在这几种格式的适应层报文中。

- 包含IPv6报头的lowpan数据报文
	 
	 	 +---------------+-------------+---------+
      	 | IPv6 Dispatch | IPv6 Header | Payload |
      	 +---------------+-------------+---------+

- 采用LOWPAN_HC1压缩IPv6报头的lowpan数据报文

		 +--------------+------------+---------+
      	 | HC1 Dispatch | HC1 Header | Payload |
      	 +--------------+------------+---------+

- 采用LOWPAN_HC1压缩IPv6报头并采用mesh传输的lowpan数据报文

		+-----------+-------------+--------------+------------+---------+
      	| Mesh Type | Mesh Header | HC1 Dispatch | HC1 Header | Payload |
      	+-----------+-------------+--------------+------------+---------+

- 采用LOWPAN_HC1压缩IPv6报头并分片传输的lowpan数据报文

		+-----------+-------------+--------------+------------+---------+
        | Frag Type | Frag Header | HC1 Dispatch | HC1 Header | Payload |
        +-----------+-------------+--------------+------------+---------+


- 采用LOWPAN_HC1压缩IPv6报文并采用mesh传输，分片传输的lowpan数据报文

		+-------+-------+-------+-------+---------+---------+---------+
        | M Typ | M Hdr | F Typ | F Hdr | HC1 Dsp | HC1 Hdr | Payload |
        +-------+-------+-------+-------+---------+---------+---------+

- 采用LOWPAN_HC1压缩IPv6报文并采用mesh传输并支持广播的lowpan数据报文
		
		+-------+-------+-------+-------+---------+---------+---------+
        | M Typ | M Hdr | B Dsp | B Hdr | HC1 Dsp | HC1 Hdr | Payload |
        +-------+-------+-------+-------+---------+---------+---------+

#### Mesh Addressing Header字段

当节点在多跳拓扑中需要为其他节点转发数据时，在转发过程中适配层报文头部字段M Typ值将设为1，M Hdr紧跟其后

                     1                   2                   3
       0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |1 0|V|F|HopsLft| originator address, final address
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

V字段：指出Originator Address是64位长地址还是16位短地址，0为长地址，1为短地址

F字段：指出Final Address是64位长地址还是16位短地址，0为长地址，1为短地址

Hops Left：剩余跳数，每经过一个转发节点该字段减1，如果该字段减到0则丢弃该帧

Originator Address：	 适配层源地址

Final Destination Address：适配层目的地址

#### Fragmentation Header字段

如果一个数据包可封装在一个802.15.4数据帧中，那么lowpan数据包不应该被分片，如果数据包不适合在一个单一的802.15.4帧中，应对其进行分片。由于分片偏移只能表示八个字节的整数倍，除了最后一个数据报的所有分片必须是8个字节的倍数。第一段分片定义如下：

                     1                   2                   3
       0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |1 1 0 0 0|    datagram_size    |         datagram_tag          |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

第二片及其后面的分片定义如下：

	                           1                   2                   3
       0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |1 1 1 0 0|    datagram_size    |         datagram_tag          |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |datagram_offset|
      +-+-+-+-+-+-+-+-+

 datagram_size：这个11位字段编码表示链路层分片前整个IP包的大小

 datagram_tag:对于同一数据包的不同分片datagram_tag值应该是一样的，不同数据包需按递增顺序来增加datagram_tag值

 datagram_offset：该字段值只在第二个分片以及之后的分片中出现，该字段值需按照成8增长，表示与第一个分片的长度差

### IPv6报头压缩

IPv6报头如下所示，下面将说明除地址压缩外的IPv6报头压缩格式。

	 0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |Version| Traffic Class |              Flow Label               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |      PlayLoad Length          |  Next Header  |   Hop Limit   |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                            Source                             |
    |                            Address                            |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                            Destination                        |
    |                            Address                            |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

压缩后的格式为：

	   0                                       1
       0   1   2   3   4   5   6   7   8   9   0   1   2   3   4   5
     +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
     | 0 | 1 | 1 |  TF   |NH | HLIM  |CID|SAC|  SAM  | M |DAC|  DAM  |
     +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+

对应的压缩格式为：

<table>
<tbody>
<tr><td><em>IPv6域</em></td><td><em>压缩域</em></td><td><em>详细信息</em></td></tr>
<tr><td></td><td>011</td><td>前三比特固定为111</td></tr>
<tr><td>Version</td><td>省略</td><td>省略版本号</td></tr>
<tr><td>Traffic Class&Flow Label</td><td>TF</td><td></td></tr>
<tr><td>PlayLoad Length</td><td>省略</td><td>省略有效载荷长度</td></tr>
<tr><td>Next Header</td><td>NH</td><td></td></tr>
<tr><td>Hop Limit</td><td>HLIM</td><td></td></tr>
</tbody>
</table>

具体请见RFC6282，一时半会儿还没彻底搞明白

### UDP报头压缩

TCP的机制比较复杂，所以TCP报头的压缩并未在RFC文档中看到，UDP报文如下所示：


				 0      7 8     15 16    23 24    31  
                 +--------+--------+--------+--------+ 
                 |     Source      |   Destination   | 
                 |      Port       |      Port       | 
                 +--------+--------+--------+--------+ 
                 |                 |                 | 
                 |     Length      |    Checksum     | 
                 +--------+--------+--------+--------+ 
                 |                                     
                 |          data octets ...            
                 +---------------- ...                 

可见UDP报头比较简单，包含有源端口，目的端口，长度，检验和4个域，UDP压缩中需进行端口的压缩并省略掉长度域，对于检验和并不进行压缩。

UDP报文接在IP报文后，属于IP报头的下一报头，需以‘11110’开始，压缩格式如下所示：

	 				 0   1   2   3   4   5   6   7
                     +---+---+---+---+---+---+---+---+
                     | 1 | 1 | 1 | 1 | 0 | C |   P   |
                     +---+---+---+---+---+---+---+---+

端口压缩：因为P字段只有两个比特，因此只能覆盖4种情况：
	
	00：所有源端口和目的端口都会保留
	01：目的端口的前8比特会被省略，其他部分和源端口被保留
	10：源端口的前8比特会被省略，其他部分和目的端口被保留
	11：目的端口和源端口的前12比特会被省略，其他部分保留

长度：长度可通过更低层次协议报头计算得到，UDP长度被省略

检验和：	UDP中必须用到检验和，介于这个原因，6lowpan并不压缩检验和部分

### 适配层功能

由于IPv6层提供的接口与802.15.4提供的接口有一定的差异，IPv6无法直接使用IEEE 802.15.4，所以适配层对MAC接口提供一定的封装，为IP层提供标准的接口，适配层需要的提供的功能有：

- 网络拓扑管理
	
	IEEE 802.15.4MAC协议支持包括星型拓扑、树状拓扑及点对点的Mesh拓扑等多种网络拓扑结构，但是MAC层协议并不负责这些拓扑结构的形成，它仅仅提供相关的功能性原语。因此适配层必须负责以合适的顺序调用相关原语，完成网络拓扑的形成。

- 地址管理

	 在6LowPAN中节点有两类地址:64 位长地址和16 位短地址。64 位长地址为标准的IPv6地址，但是IEEE802，5.4MAC的有效报文长度仅为127 字节，如果所有节点均采用64 位长地址进行通信的话，则留给上层协议的有效负荷长度将大大减少。因此在PAN内部使用 16位短地址进行通信，这就需要在适配层采用一种动态的地址分配机制，使得6LowPAN节点能够正确获得PAN内独一无二的16 位短地址。

- 路由协议
	
	IETF的ROLL工作组提出了RPL路由协议，具体情形请见RFC6550。

- 数据包的分片和重组
	
	IPV6规定的链路层最小MTU为1280字节，而IEEE802.15.4MAC层最大帧长仅为127 字节，因此，适配层需要通过对IP报文进行分片和重组来传输超过IEEE802.15.4层最大帧长的报文。

- 头部压缩和解压缩
	
	 IEEE802.15.4MAC层除去报文头部后的最大有效负荷长度为102 字节，而IPV6 基本报头为40字节，为上层数据留下的有效长度仅仅剩下50字节左右的空间。为了使IPv6报文能够在IEEE802 15.4链路上传输，我们需要IPV6 报文头部及传输层报文头部进行一定的压缩，进而更大限度的提高传输的效率。

- 组播支持

	邻居发现协议很大程度上依赖于组播，但是 IEEE802.15.4MAC协议并不提供组播功能，仅仅提供有限的广播功能，如果单纯的将所有组播功能退化为广播，则会对节点能耗上造成很大的开销，因此需要在适配层通过有效的可控广播洪泛的方法实现IP报文组播的功能。

	

	