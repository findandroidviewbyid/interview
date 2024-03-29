

网络层是为主机之间提供逻辑通信，而传输层为应用进程之间提供端到端的逻辑通信。根据应用程序的不同需求，传输层需要有两种不同的传输协议，即面向连接的TCP和无连接的UDP。

TCP 和 UDP 是今天应用最广泛的传输层协议，无论是应用开发、框架设计选型、做底层和优化，还是定位线上问题，只要碰到网络，就逃不开 TCP 协议相关的知识。

传输层需要有复用和分用的功能，即应用层所有的进程都可以通过传输层再传输到网络层，这是复用。传输层从网络层收到数据后必须交付给指明的应用进程，这是分用。主机的通信实际上是主机上的两个进程之间的通信，网络层通过IP地址建立起来两台主机的联系，但并不能建立起两个进程间的联系，因此，给应用层的每个应用进程分配一个明确的标志是非常重要的。这个标志就是协议端口（protocol port number），简称端口。虽然通信的终点是进程，但是只要把传送的报文交到目的主机的某一个合适的端口，剩下的工作就可以由TCP来完成。


在TCP/IP体系中，根据使用的协议是TCP还是UDP，分别称之为TCP报文段（Segment）或UDP用户数据报。UDP传送数据前不需要先建立连接。远程主机收到UDP报文后不需要给出任何确认。TCP则是面向连接的服务。在传送数据之前必须先建立连接，数据传送结束后释放连接。由于TCP要提供可靠的、面向连接的传输服务，因此不可避免的增加了许多开销，如确认、流量控制、计时器以及连接管理等。

## UDP协议
S
用户数据报协议UDP在IP的数据报服务纸上增加了很少的功能，即复用和分用的功能以及差错检测的功能。UDP的主要特点如下：

1. UDP是无连接的，即发送前不需要建立连接，因此减少了开销和发送数据之前的延迟。
2. UDP使用尽最大努力交付，即不保证可靠交付，因此主机不需要维持复杂的连接状态
3. UDP是面向报文的。
4. UDP没有拥塞控制。网络出现阻塞不会使源主机发送的速率降低，这对某些实时性强的应用是很重要的。
5. UDP支持一对一、一对多、多对一和多对多的交互通信。
6. UDP的首部开销小，只有8个字节。



### UDP的首部格式

用户数据报UDP有两个字段：数据字段和首部字段。首部字段有8个字节，由四个四段构成。每个字段长度为两个字节。各个字段的意义如下：

![save_share_review_picture_1628950684](https://user-images.githubusercontent.com/19853475/129449313-5296a13a-d1b0-4977-ae33-bd87969af573.jpeg)

1. 源端口 在需要对方回信时选用，不需要时可全为0
2. 目的端口 在终点交付报文时必须用到。
3. 长度 UDP用户数据报的长度，最小值为8
4. 校验和 检测UDP用户数据报在传输中是否有错，有错则丢弃。

## TCP协议

### TCP的主要特点

  1. TCP是面向连接的传输层协议
  2. 每条TCP连接只能有两个端点，即每条TCP连接都是点对点的。
  3. TCP提供可靠的交付服务
  4. TCP提供全双工通信
  5. 面向字节流。

### TCP的连接

TCP（Transport Control Protocol）是一个传输层协议，提供 Host-To-Host 数据的可靠传输，支持全双工，是一个连接导向的协议。表面上看TCP是主机与主机的通信，但事实上是主机的应用到主机的应用的通信。TCP连接的端点叫做套接字（Socket），通过IP地址拼接端口号即可构成套接字。套接字的表示方法是在点分十进制的IP地址后面写上端口号，中间用冒号隔开，如192.168.4.5:80

### 可靠传输的工作原理

TCP发送的报文段是交给IP层传送的，但是IP层不提供可靠传输，因此TCP必须采用适当的措施让两个传输层之间的通信变得可靠。

#### 停止等待协议

全双工通信的双方即是发送方，也是接收方。此处仅考虑A发送数据，B接收数据并确认发送的情况。传送的数据单元称为分组，停止等待协议是每次发完一个分组就停止发送，等待接收方的确认，收到确认后再发送下一个分组。


在传送过程中避免不了出现错误，接收方在检测出数据出现差错就丢弃分组，并且不会通知A收到的分组有差错。另外，分组在传送过程中也可能丢失，这时接收方什么都不知道。

而发送方如果超过了一定的时间没有收到接收方的确认，则会进行超时重传。实现超时重传需要发送完一个分组后设置一个超时计时器，如果收到接收方的确认则移除计时器。

等待协议的优点是简单，但缺点是信道利用率太低。为了提高效率，发送方一般采用**流水线传输。** 流水线传输就是发送方可连续发送多个分组，不必每发完一个分组就等待确认。这样使得信道上的数据可以不间断的传送。当使用流水线传输时，需要使用到连续**ARQ协议** 和**滑动窗口协议**

#### 连续ARQ协议

![WechatIMG25](https://user-images.githubusercontent.com/19853475/129452195-cb3df4cf-2de6-48c6-99a8-0b4a93928c50.jpeg)

连续ARQ协议规定，发送方没收到一个确认就把发送窗口向前滑动一个分组的位置。如上图，收到了第一个分组的确认就把发送窗口先前移动了一个分组位置。

而接收方一般都是采用累计确认的方式，就是说接收方不会对收到的分组逐个发送确认，而是收到几个分组后，按序到达最后一个分组发送确认。

累计确认的优点是容易实现，即使丢失也不用重传。但是，缺点是不能向发送方反映出接收方已经正确收到的分组信息，例如，如果发送方发送了前五个分组，而中间第三个分组丢失了。这时接收方只能对前两个分组发送确认，发送方无法知道后面三个分组的下落，只好把后面三个分组都再重传一次。

#### TCP报文段的首部格式

TCP传输的数据单元是报文段，一个TCP报文段分为首部和数据两部分，而TCP的全部功能体现在它首部中各个字段的作用。TCP报文首部的前20个字节是固定的，后面有4n字节是根据需要增加的选项。因此，TCP首部最小长度是20字节。如下：

![WechatIMG26](https://user-images.githubusercontent.com/19853475/129452638-596e60ba-6ccd-4ce1-9d35-c07861b1fbe6.jpeg)



## TCP与UDP常见面试题

### 什么是连接会话？

连接是通信双方的一个约定，目标是让两个在通信的程序之间产生一个默契，保证两个程序都在线，而且尽快的相应对方的请求，这就是连接。设计上连接是一种传输数据的行为，传输之前先建立一个连接，具体的说就是数据收发双方的内存中都建立一个用于维护数据传输状态的对象，比如对方的IP地址和端口号是多少，当前发送了多少数据，传输速度如何等。

和连接相关的一个名词叫**会话（Session）。** 会话是应用的行为。比如，微信里和别人聊天创建了一个聊天窗口，这个就是会话，开始Typing，发送的时候就和微信服务器之间建立了一个连接。聊天结束后不关闭微信，那么连接断开，但因为窗口没关，所以会话还在。

总的来说，会话是应用层的概念，而连接是传输层的概念。



### 什么是单工和双工？

单工是指在任何一个时刻，数据只能单向发送，即A可以向B发送数据，但B不能向A发送数据。

半双工是指数据可以向一个方向传输，也可以反向传输，但是是交替进行的。

全双工是指任何时刻数据都可以双向发送。

TCP与UDP都是双工协议。数据任何时候都可以双向传输。



###  TCP协议与UDP协议的区别

TCP/IP 中有两个具有代表性的传输层协议，分别是 TCP 和 UDP。

TCP 是面向连接的、可靠的流协议。流就是指不间断的数据结构，当应用程序采用 TCP 发送消息时，虽然可以保证发送的顺序，但还是犹如没有任何间隔的数据流发送给接收端。TCP 为提供可靠性传输，实行“顺序控制”或“重发控制”机制。此外还具备“流控制（流量控制）”、“拥塞控制”、提高网络利用率等众多功能。
UDP 是不具有可靠性的数据报协议。细微的处理它会交给上层的应用去完成。在 UDP 的情况下，虽然可以确保发送消息的大小，却不能保证消息一定会到达。因此，应用有时会根据自己的需要进行重发处理。
TCP 和 UDP 的优缺点无法简单地、绝对地去做比较：TCP 用于在传输层有必要实现可靠传输的情况；而在一方面，UDP 主要用于那些对高速传输和实时性有较高要求的通信或广播通信。TCP 和 UDP 应该根据应用的目的按需使用。

### TCP协议的三次握手

TCP有几个基本操作：

- 如果一个Host主动向另一个Host发起连接，称为SYN(Synchronization),即请求同步
- 如果一个Host主动断开连接，称为FIN（Finish），即请求完成。
- 如果一个Host给另一个Host发送数据，称为PSH（Push），即数据推送。

以上三种情况，接收方收到数据后，都需要给对方一个ACK（Acknowledge）相应。请求/响应的模型是可靠性的要求。如果没有响应，发送方可能认为自己需要重发这个请求。

TCP建立连接的过程（三次握手）

TCP 提供面向有连接的通信传输。面向有连接是指在数据通信开始之前先做好两端之间的准备工作。
所谓三次握手是指建立一个 TCP 连接时需要客户端和服务器端总共发送三个包以确认连接的建立。在socket编程中，这一过程由客户端执行connect来触发。

![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/d386163fa4b8a84241e693e2084cfcac.png)

建立连接的具体流程如下：

1. 客户端发送一个SYN消息给服务端，请求与服务端建立连接。
2. 服务端准备就绪，并向客户端发送一个ACK，确认可以连接。
3. 服务端发送一个SYN消息给客户端，请求与客户端建立连接。
4. 客户端准备就绪，并向服务端发送一个ACK，确认可以连接。

乍一看，这个连接过程是4步，但其实2，3步是同时发生的，即可以合并成一个ACK+SYN的响应一起发送给客户端。因此实际上双方只进行了三次握手，如下图所示：

![CioPOWB-RYSASfPkAAEen4ZR3gw297](https://user-images.githubusercontent.com/19853475/129468667-fc907302-06c2-46f5-84d5-939f8936b9b2.png)


### TCP协议的四次挥手

四次挥手即终止TCP连接，就是指断开一个TCP连接时，需要客户端和服务端总共发送4个包以确认连接的断开。在socket编程中，这一过程由客户端或服务端任一方执行close来触发。

由于TCP连接是全双工的，因此，每个方向都必须要单独进行关闭，这一原则是当一方完成数据发送任务后，发送一个FIN来终止这一方向的连接，收到一个FIN只是意味着这一方向上没有数据流动了，即不会再收到数据了，但是在这个TCP连接上仍然能够发送数据，直到这一方向也发送了FIN。首先进行关闭的一方将执行主动关闭，而另一方则执行被动关闭。
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/afa72d9ea7ed4a5853fc724ce84767be.png)

中断连接端可以是客户端，也可以是服务器端。以下假设客户端先进行断开连接请求：

1. 客户端请求断开连接，发送一个断开请求FIN。
2. 服务端收到请求，然后向客户端发送一个ACK，作为客户端断开连接请求的响应。
3. 服务端经过一个等待，确定服务端的数据已经发送完成，则向客户端发送一个FIN，请求与客户端断开连接。
4. 客户端收到服务端的FIN后，知道可以关闭连接了，但是可能有自己的事情还没处理，比如客户端发送给服务端没有收到ACK的请求，客户端处理完自己的事情后，会再次发送一个ACK响应服务端的断开请求。

断开连接的过程与建立连接不同的是第2，3步不是合并发送的。主要是因为服务端可能还要处理很多问题，比如服务端还有发送出去的消息没有得到ACK，也可能是自身资源要释放。因此，断开连接不能像握手那样将两条消息合并发送。

### 为什么TCP的连接需要三次握手，而断开连接需要四次挥手？

参考TCP协议的三次握手和四次挥手。

### TCP的滑动窗口与流速控制是什么？

滑动窗口是TCP协议控制可靠传输的核心，发送方将数据拆包，变成多个分组，然后将数据放入一个拥有滑动窗口的数据以此发出，仍然遵循先进先出的顺序，但是窗口中的分组会一次性发送。窗口中序号最小的分组如果收到ACK，窗口就发生滑动，如果最小序号的分组长时间没有收到ACK，就会触发整个窗口的重新发送。


