---
layout: post
title:  "MySQL性能优化之linux系统参数优化"
date:   2018-06-03 11:11:13 +0000
categories: xiaochang update
---

我们以CentOs为例，改变一些对数据库影响较大的参数配置来优化数据库性能

一、内核相关的参数配置

1、网络相关参数

说起网络，首次先来了解一下tcp链接，对于tcp链接来说服务器与客户端需要三次握手来建立网络的链接，当三次握手成功之后，端口由监听转换成了可连接，然
后这条链路就可以传输数据了，对于一个监听的端口都会有自己的监听队列。

      net.core.somaxconn = 65535   
      net.core.netdev_max_backlog = 65535
      net.ipv4.tcp_max_syn_backlog = 65535
      
   配置一：每个端口监听队列的最大长度
   作用：默认值比较小，对于负载大的时候需要修改更大的值

   配置二：在每个网络接口接收数据包的速率比内核处理数据包的速率快的时候允许未发送到队列的数据包最大的数目
     
   配置三：还未获得对方链接的请求和保存在列队中的最大数目    
   作用：对超过这些数值的链接会被抛弃，所以要调大一些 
      
      
      net.ipv4.tcp_fin_timeout = 10 
      net.ipv4.tcp_tw_reuse = 1
      net.ipv4.tcp_tw_recycle = 1
      
   配置一：用于控制tcp链接处理的等待状态的时间，对于链接比较频繁的系统，通常会有大量的链接处于等待状态的，较少这种等待的时间，加快tcp的回收速度
   配置二、配置三：用于加快tcp链接的回收，在高负载的情况下是非常重要的，如果有大量的数据库链接请求，如果tcp链接呗占满的话，就会出现数据库无法链接的现象。
    
    
      net.core.wmem_default = 87380
      net.core.wmem_max = 16777216
      net.core.rmem_default = 87380
      net.core.rmem_max = 16777216
      
   以上四个参数决定了tcp接连接收和发送缓冲区的默认值和最大值，对于数据库应用来说，也需要吧这几个值调的稍微大一点
   
      
      net.ipv4.tcp_keepalive_time = 120   
      net.ipv4.tcp_keepalive_intvl = 30  
      net.ipv4.tcp_keepalive_probes = 3  
      
   配置一：tcp发送探测消息时间的间隔
   配置二：tcp未获得响应时，重发该消息时间的间隔
   配置三：在tcp链接失效之前，最多发送多少个探测消息
      
   用于减少失效链接所占用的tcp系统资源的数量,加快资源回收的效率。
   以上三个值的意思就是如果一个tcp链接120秒中没有数据传输则开始探测有效性，每次探测的时间间隔是30秒，最多探测3次
    
    
 2、内存的相关参数
      
        kernel.shmmax = 4294967295  
      
   //用于定义单个共享内存段的最大值 
     
   这个参数应该设置的足够大，以便于在一个共享内存段下容纳下整个的Innodb缓冲池的大小
    
   这个值的大小对于64位的Linux系统，可取的最大值为物理内存值-1byte，建议值大于物理内存的一半，一般取值大于Innodb缓冲池的大小即可，可以取值物理内存-1byte.

        vm.swappiness = 0  
        
   //当内存不足时会对性能产生比较明显的影响，  
    
   在mysql服务器上保留交换区很有必要的，但是要控制何时使用交换分区，vm.sawppiness =0就是告诉linux内核除非内存完全满了，否则不要使用交换区。(Linux系统内存交换区：在linux系统安装时都会有一个特殊的磁盘分区，称之为系统交
换分区  swap就是交换分区 ，当操作系统因为没有足够的内存时就会将一些虚拟内存写到磁盘的交换区中这样就会发生内存交换。)
    
 
二、增加资源文件限制（etc/security/limit.conf）
    
   这个文件是linux PAM也就是插入式认证模块配置文件，即打开文件数的限制

        *soft nofile 65535
        *hard nofile 65535 
        
   以上两个配置是用于控制打开文件的数量的限制

   * 表示对所有用户有效
   soft     指的是当前系统生效的设置         对与同一资源 soft设置不能比hard高
   hard     表明系统中所能设定的最大值
   nofile   表示所限制的资源是打开文件的最大数目
   65535    就是限制的数量

  可打开的文件数量增加到了65535个以保证可以打开足够多的文件句柄
  注意：这个文件的修改需要重启系统才能生效
  
三、磁盘调整策略（sys/block/decname/queue/scheuler）

       cat  /sys/block/sda/queue/scheduler //查看磁盘调度策略命令
       noop anticipatory deadline [cfq]   //磁盘调整策略位cfq
       
 cfq  按进程创建多个列队，实图保持多个进程的公平，这就没有考虑到读与写操作的不同耗时。
 
 noop（电梯式调度策略） noop实现了一个FIFO列队，它像电梯的工作方法一样对I/O请求进行组织，当有一个新的请求到来时，它将请求合并到最近的请求之后，以此来保证请求同一介质。NOOP倾向饿死读而利于写，因此NOOP对于闪存设备、RAM及嵌入式系统是最好的选择。
 
 deadline (截止时间调度策略) 确保了在一个截止时间内服务请求，这个截止时间是可调整的，而默认读期限短于写期限。这样就防止了写操作因为不能被读取而饿死的现象，Deadline对数据库类应用是最好的选择。

 anticipatory(预料I/O调整策略)  本质上与Deadline一样，但是最后一次读操作，要等待6ms，才能继续对其他I/O请求进行调度。它会在每个6ms中插入新的I/O操作，而会将一些小写入流合并成一个大写入流，用写入延时换取量大的写入吞吐
量。AS适合于写入较多的环境，比如文件服务器，AS对数据库环境表现很差。


      echo deadline > /sys/block/sda/queue/scheduler  //改变磁盘调整策略命令
      
      
      
以上是系统配置设置来优化数据库的
      
