---
layout: post
title:  "Linux 性能监控实践[1]"
date:   2013-07-06 19:07:50
---
搞Hadoop仅依靠ganglia或其他监控管理工具是远远不够的,真正了解熟悉各种mstat,nload,top等才靠谱,所以写篇linux的性能监控方面的文章搞清各个监控命令的细枝末节

###内存 free
`free`显示系统内存当前空闲和占用量以及swap内存交换使用情况,free实质是显示/proc/meminfo的格式化信息
{% highlight ruby %}
ly@ly-Latitude-E5400:~$ free
                     total      used      free      shared  buffers   cached
Mem:                 3089212    2049072   1040140   0       13784     1231952
-/+ buffers/cache:   803336     2285876     
Swap:                3134460    7488      3126972
{% endhighlight %}
free命令默认显示的是kilobytes字节的大小   `free -m` 按照megabytes字节的大小显示


* total   系统的总共内存大小(这里有个注意的地方,total的大小与机器实质配置的内存要略微小一点,这是因为kernel在启动时永久的占据了一部分内存的原因)

* used    当前运行的application和os所占用的内存大小

* free = total - used 系统空闲的内存大小

* shared  已经deprecated了,指多个processes共享的内存大小

* buffers 系统用于IO传输操作,例如数据等待写入磁盘

* cached  最近使用的文件被cached在了内存中,所以application跑起来更快


要特别注意的是NR=1和NR=2的区别,Mem和-/+buffers/cache的区别在于buffers和cached的大小在Mem行被算入used中,而实质上buffers和cached所占用的内存是可被分配给系统新的应用需求,即可被剥夺的,所以-/+buffers/cache行才是系统'实质'的占用和空闲内存使用情况.ps:{Mem:total}={Mem:used}+{Mem:free}={-/+buffers/cache:used}+{-/+buffers/cached:free},Swap即内存不足时,系统将一部分内存交换写入disk中,当系统开始使用swap交换分区则说明系统负载过重,对于我们的集群应用来说是不可接受的

命令行参数简介:

 *  -s {num} 以num秒的延迟持续刷新free的值

 *  -c {num} 和-s组合使用,限制刷新的次数

 *  -l 统计Mem的最大值和最小值

 *  -o 仅显示Mem和Swap行

 *  -t 多输出了一行是Mem行和Swap的sum值

`free -m -s 2` free以megabytes的大小显示内存,每2秒刷新一次

buffers和cached的大小你可以通过开多个虚拟机发现越变越小,当你立即释放虚拟机占据的内存后,buffers和cached的大小是伴着系统后续文件的使用逐渐增大的,猜想:buffers和cached内存占用的大小应该直接影响系统的性能,这部分内存占用越小系统的性能越低?


###I/O iostat
iostat是用来监测硬盘的读写速率,也就是IOPS.iostat report了三个方面的信息{cpu,device I/O,network},其中cpu和network的相关信息是和I/O操作相关的.iostat的report的原数据主要来自/proc/diskstats.先来看一下cpu方面的report信息

{% highlight ruby %}
Linux 2.6.32-279.el6.x86 _ 64 (hostname) 	07/31/2013 	_x86_64 (24 CPU)

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           2.61    0.00    1.23    0.42    0.00   95.75


{% endhighlight %}

 * %user  用户级别的cpu占用率,就是应用的占用率

 * %nice  用户级别的并且开始执行应用时指定了nice的优先级(priority)的cpu占用率

 * %system  当然是系统级别的kernel的cpu占用率

 * %iowait 硬盘的I/O很容易成为系统的瓶颈,此项目就显示了cpu在等待I/O操作所占的比重,我们当然希望这个值越低系统运行的越健康

 * %steal  虚拟化的cpu等待的处理器服务于其他虚拟的cpu的时间.例如,虚拟的两个cpu Va和Vb, steal就是Va等待处理器被Vb占用的执行指令的时间.

 * %idle 在没有磁盘I/O操作状态下CPU空闲的占用率信息


再来看一下iostat的主要作用监控磁盘设备的状态
  
 * tps 表示一秒钟内设备的传输次数,一次传输指的是一次设备上的I/O请求,多个逻辑上的I/O请求合并为一次I/O请求,一次I/O请求传输的大小不确定.

 * Blk_read/s(kB_read/s, MB_read/s) 每秒钟从设备读取的块的数量(kB或者MB),KB块的大小为扇区的大小,即512k
  
 * Blk_wrtn/s (kB_wrtn/s, MB_wrtn/s)每秒钟写入到设备的块的数量

 * Blk_read(kB_read, MB_read) 读取块的总数

 *  Blk_wrtn(kB_wrtn, MB_wrtn) 写入块的总数

 * rrqm/s 对于写队列中对相同的block的读取fileSystem会合并为一个读请求,rrqm/s表示的就是合并的个数

 * wrqm/s 对于相同块的写入合并为一次写操作,wrqm/s表示的就是合并的写个数

 * r/s 每秒完成的读操作次数(合并后的读操作)

 * w/s 每秒完成的写操作的次数(也是合并后的操作)

 * rsec/s (rkB/s, rMB/s) 每秒读的扇区的数量

 * wsec/s (wkB/s, wMB/s) 每秒写的扇区的数量
 
 * avgrq-sz 每秒I/O设备平均的请求次数

 * avgqu-sz 每秒I/O设备的平均请求队列长度

 * await 每毫秒秒平均的请求等待和服务时间.包括请求到队列中,和I/O设备完成请求服务的时间

 * r _ await 毫秒级的读请求的等待和服务时间  

 * w _ await 毫秒级的写请求的等待和服务时间

 * svctm  毫秒级的平均服务时间I(将要被移除,man文档的描述是不推荐的参考值)

 * %util I/O操作占用CPU的百分率,当I/O操作过于繁忙的时候值会达到100%


  至于网络的话就不写了.太多了>.<


命令行参数的使用:

  * -c 仅打印CPU的report

  * -d 打印I/O设备的report 

  * -h report输出格式化

  * -k 输出的大小单位为kilobytes

  * -m 输出的大小单位为megabytes

  * -p 输出一个指定I/O设备的report,输出的结果中还包行这个设备下的所有分区表的分别的repor的统计信息 ex: iostat -dx -p sda 2 6 指定输出sda这个设备下的扩展的统计信息,每两秒输出一次,一共输出6次
  
  * -t 指定report的统计的间隔时间(可忽略直接上数字)

  * -V 打印iostat的版本号

  * -x 显示扩展的统计项.默认的显示是从{tps-Blk_wrtn},而扩展项是从{rrqm/s-util} 用法iostat -x sda

  

参考书籍   
 *  [鸟哥的Linux私房菜.基础学习篇][鸟哥的Linux私房菜.基础学习篇] 
 *  [UNIX and Linux System Administration Handbook][UNIX and Linux System Administration Handbook]
[鸟哥的Linux私房菜.基础学习篇]: http://book.douban.com/subject/4889838/ 
[UNIX and Linux System Administration Handbook]: http://book.douban.com/subject/1779429/
