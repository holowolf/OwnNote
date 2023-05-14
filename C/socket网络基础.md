### OSI七层模型与TCP/IP四层模型对应关系

| OSI 七层模型 | TCP/IP 四层模型 |
| :---------: | :-------------: |
| 物理层 |   |
| 数据链路层 | 数据链路层 |
| 网络层 | 网络层 |
| 传输层 | 传输层 |
| 会话层 |  |
| 表示层 |  |
| 应用层 | 应用层 |


#### TCP/IP四层模型数据链路层的以太网帧协议
以太网帧协议格式 借助mac地址完成数据报传输  

| 目的地址 | 源地址 | 类型 |   数据   | CRC |
| :---:  | :----: | :---: | :----: | :----: |
| 6 | 6 | 2 | 46- 1500 | 4 |

arp数据报 -- 根据IP获取mac地址

| 以太网<br>目的地址 | 以太网<br>源地址 | 帧类型 | 硬件类型 | 协议类型 | 硬件<br>地址长度 | 协议<br>地址长度 | op | 发送端<br>以太网地址 | 发送端<br>ip地址 | 目的<br>以太网地址 | 目的<br>ip地址 |
| :---------: | :------: | :-------: | :---: | :---: | :---: | :----: | :----: | :---: | :---: | :----: | :----: |
| 6 | 6 | 2 | 2 | 2 | 1 | 1 | 2 | 6 | 4 | 6 | 4 |
以太网首部 包含以太网地址,以太网源地址,帧类型. 剩下28字节为ARP应答请求


4位版本号: ipv4, ipv6  
8位生存时间(TTL):  最多能经过多少跳  
32位源IP地址 : 数据发送端地址  
32位的IP地址 : 数据接收端地址  

IP 报文
<table>
    <tr>
        <td colspan="8" style="text-align:left">0</td>
        <td colspan="8" style="text-align:right">15</td>
        <td colspan="8" style="text-align:left">16</td>
        <td colspan="8" style="text-align:right">31</td>
    </tr>
    <tr>
        <td colspan="4" style="text-align:center">4位版本</td> 
        <td colspan="4" style="text-align:center">4位首部长度</td> 
        <td colspan="8" style="text-align:center">8位服务类型(TOS)</td> 
        <td colspan="16" style="text-align:center">16位总长度(字节数)</td> 
   </tr>
    <tr>
        <td colspan="16" style="text-align:center">16位标识</td>    
        <td colspan="3" style="text-align:center">3位标志</td> 
        <td colspan="13" style="text-align:center">13位片偏移</td> 
    </tr>
    <tr>
           <td colspan="8" style="text-align:center">8位生存时间</td>    
        <td colspan="8" style="text-align:center">8位协议</td> 
        <td colspan="16" style="text-align:center">16位首部检验和</td> 
    </tr>
    <tr>
           <td colspan="32" style="text-align:center">32位源IP地址</td>    
    </tr>
    <tr>
           <td colspan="32" style="text-align:center">32位目的IP地址</td>    
    </tr>
    <tr>
           <td colspan="32" style="text-align:center">选项(如果有)</td>    
    </tr>
    <tr>
           <td colspan="32" style="text-align:center">数据</td>    
    </tr>
</table>


16位源端口  
16位目的端口  
UDP报文格式
<table>
    <tr>
        <td colspan="8" style="text-align:left">0</td>
        <td colspan="8" style="text-align:right">15</td>
        <td colspan="8" style="text-align:left">16</td>
        <td colspan="8" style="text-align:right">31</td>
    </tr>
    <tr>
        <td colspan="16" style="text-align:center">16位源端口号</td> 
        <td colspan="16" style="text-align:center">16位目的端口号</td> 
   </tr>
    <tr>
        <td colspan="16" style="text-align:center">16位UDP长度</td>    
        <td colspan="16" style="text-align:center">16位UDP检验和</td> 
    </tr>
    <tr>
           <td colspan="32" style="text-align:center">数据(如果有)</td>    
    </tr>
</table>

16位源端口  
16位目的端口  
32位序列号  
32位确认序号  
6个标志位  
16位滑动窗口  

TCP 报文格式
<table>
    <tr>
        <td colspan="8" style="text-align:left">0</td>
        <td colspan="8" style="text-align:right">15</td>
        <td colspan="8" style="text-align:left">16</td>
        <td colspan="8" style="text-align:right">31</td>
    </tr>
    <tr>
        <td colspan="16" style="text-align:center">16位源端口</td> 
        <td colspan="16" style="text-align:center">16位目的端口</td> 
   </tr>
    <tr>
        <td colspan="32" style="text-align:center">32位序号</td>    
    </tr>
   <tr>
        <td colspan="32" style="text-align:center">32位确认序号</td>    
    </tr>
    <tr>
           <td colspan="4" style="text-align:center">4位首部长度</td>    
        <td colspan="6" style="text-align:center">保留(6位)</td> 
        <td colspan="1" style="text-align:center">U<br>R<br>G</td> 
        <td colspan="1" style="text-align:center">A<br>C<br>K</td> 
        <td colspan="1" style="text-align:center">P<br>S<br>H</td> 
        <td colspan="1" style="text-align:center">P<br>S<br>T</td> 
        <td colspan="1" style="text-align:center">S<br>Y<br>H</td> 
        <td colspan="1" style="text-align:center">F<br>I<br>H</td> 
        <td colspan="16" style="text-align:center">16位窗口大小</td> 
    </tr>
    <tr>
           <td colspan="16" style="text-align:center">16位检验和</td>    
           <td colspan="16" style="text-align:center">16位紧急指针</td>    
    </tr>
    <tr>
           <td colspan="32" style="text-align:center">选项</td>    
    </tr>
    <tr>
           <td colspan="32" style="text-align:center">数据</td>    
    </tr>
</table>


* tcp, udp 传输层协议
    * tcp:面向连接的安全的流式传输协议
        * 连接的时候进行三次握手
        * 数据发送的时候会进行数据确认`(数据丢失后会进行数据重传)`

    * udp:面向无连接的不安全的报式传输
        * 连接时候不会握手
        * 数据发送出去之后就不管了 `(如果数据报丢失会全丢)`



* socket编程
    * 什么是socket
        * 网络通信的函数接口
        * 封装传输层协议
            * tcp
            * udp
    * 网络IO编程
        * 读写操作
        * read/write
            * 文件描述符
        * 创建一个套接字, 得到文件描述符
    * 匿名管道
        * 在内存中创建一块存储空间
        * 管道的读写两端分别为一个文件描述符
    * 套接字
        * 创建成功后得到一个文件描述符 fd
        * fd 操作的是一块内核缓冲区
        * 当套接字被创建时分为两个部分 (1. 读缓冲区, 2. 写缓冲区) 内核缓冲区 
        * 客户端对应的两部分 (1.写缓冲区, 2. 度缓冲区)
        * 当客户端写缓冲区有数据时会自动通过网络传递到服务端的读缓冲区, 同理服务端的写缓冲区也会传递到客户端读缓冲区
        * 套接字通讯默认是阻塞的
    * socket tcp server
        * 创建套接字
            * int fd = socket
        * 绑定本地ip和端口
            * struct sockaddr_in_serv;
            * serv, port = htons(prot);
            * serv.IP = htonl(INADDR_ANY);
            * bind(fd, &serv, sizeof(serv));
        * 监听
            * listen(fd, 128);
        * 等待并接收连接请求
            * struct sockaddr_in_client;
            * int len = sizeof(client);
            * int cfd = accept(fd, &client, &len);
            * cfd-用于通信的
        * 通信
            * 接收数据 : read/recv
            * 发送数据 : write/send
        * 关闭
            * close(fd);
            * close(cfd);

    * socket tcp client
        * 创建套接字
            * int fd = socket
        * 连接服务器
            * struct sockaddr_in_server;
            * server.port;
            * server.ip = int;
            * server.family;
            * connect(fd, &server, sizeof(server));
        * 通信
            * 接收数据 : read/recv
            * 发送数据 : write/send
        * 端口连接
            * close(fd);
    * 滑动窗口(缓冲区)
        * win 4096 (此标志位就代表滑动窗口)
        * 代表滑动窗口对应的缓冲区的大小
        * \<mss 1460> 代表最大的数据量 1460
        * eg : SYN 0(0) [发起连接序号为0, 数据大小为0] win 4096 [数据缓冲区大小为4096] \<mss 1024>[发送数据时,一条数据最大只能有1024]


#### 多进程服务器, 连接解决
* 使用多进程方式, 解决服务器处理多连接问题.
    * 共享 
        * 读时共享, 写时复制
        * 文件描述符
        * 内存映射区
    * 父进程角色
        * 等待客户端接收连接 accept
        * 有连接的时候
            * 创建一个子进程 fork()
    * 子进程角色  
        * 通信
            * 使用accept 返回 -fd
        * 关掉监听的文件描述符
            * 浪费资源
   * 创建进程的个数受硬件限制
       * 受硬件限制
       * 文件描述符默认也是有上限个数 根据系统硬件决定
   * 子进程资源回收
       * 使用wait/waitpid
   * 使用信号进行回收
       * 信号捕捉
           * signal
           * sigaction - 推荐
           * sigchld 捕捉信号