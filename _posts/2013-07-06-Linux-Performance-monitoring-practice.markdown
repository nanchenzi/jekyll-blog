---
layout: post
title:  "Linux 常用性能监控解析"
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

  * -x 显示扩展的统计项.默认的显示是从{tps-Blk _ wrtn},而扩展项是从{rrqm/s-util} 用法iostat -x sda


### I/O iotop
iotop 较iostat能提供细粒度到线程或者进程的I/O资源使用情况，而且通过r,o,p,a等快捷键提供更好的数据展示方式

 * -o 仅显示当前真实在做I/O操作的线程或者进程，方便的是可通过按键o热切换是显示线程还是进程

 * -b 数据以非交互模式的形式展示，说白了就是一定的时间才刷数据，而不是有数据变动时才刷屏幕,当然正常的显示的变动也不是绝对意义上的变动

 * -n 指定数据刷新的次数

 * -d 设定刷新的延迟时间
 
 * -p 指定iotop需要监控的pid号，默认是所有的进程,可通过p切换需要监控进程PID,还是线程TID;
  
 * -u 指定iotop监控的用户

 * -P 指定iotop监控的pid号，但是只是监控进程，一般的iotop是监控的所有的线程

 * -a 指定iotop的显示监控数据是自iotop命令输入后累加起来的监控数据，而不是一般的benchmark数据  //可热切换的快捷键

 * -k 展示数据的单位是kilobytes，一般的是Byte

 * -t 在每行的显示数据前加时间

 * -q 推出的快捷键

快捷键r用于reverse排序，可通过左右键选择排序列;其他快捷键o,p,a,q上边的命令提到了;

### 网络 nload
nload 用于实时的显示网络流量状况的工具,显示的方式分为两部分，上部分为进入的流量，下部分为发出的流量
 
  * -a 指定网络流量算入的时间ms,默认为300,那么我要指定为3秒为单位计算一次，就这样nload -a 3000 
  
  * -i nload命令滚动显示的数据图例的比例
 
  * -t 指定数据刷新的时间间隔

  * -m 把多个网络接口设备数据一起显示

  * -u  h|H|b|B|k|K|m|M|g|G  指定显示的单位 x(units)/s 
  
  * -U  h|H|b|B|k|K|m|M|g|G  同上不过显示的总的数据量而不包含平均时间单位
 
  * devices 指定需要监控的网络端口，例如我的本用的是无线连接网络,接口的名称为eth1,那么我就可以这样 nload devices eth1

### system top
top 实时的显示系统当前各个任务整体运行情况,是一个较为常用的命令

 * -c 交互式的命令快捷键，显示程序的完整启动命令

 * -d 指定显示数据的刷新时间,单位为秒

 * -H 交互的命令是否开启所有线程的总量的监控

 * -i 交互的命令是否显示idle线程和僵尸线程

 * -n 交互的命令指定页显示数据的量

 * -u 指定监控用户的UID

 * -U 指定监控用户的UID或者username

 * -p 指定监控进程的pid号

 top的常用输出解释:

 * PID 唯一的进程ID号
 
 * USER 任务的拥有者

 * PR 任务的优先级别

 * NI 负数意味着任务拥有更好的优先级

 * VIRT 任务所使用的虚拟内存的总量,包含所有换进换出的内存

 * RES 任务所使用的物理内存的总量,不包含swap的

 * S 任务的状态 'D' = 不能被中断的睡眠状态;'R'运行状态;'S'睡眠状态'T'被追踪的或者停止的状态;'Z'僵尸
 
 * %CPU cpu的利用率

 * %MEM 物理内存的占用率

 * TIME+ 自任务启用起所使用的cpu片段时间总和

 
### system vmstat 

vmstat 也是常用的监控内存,IO和CPU的工具

-a 显示内存是否活跃的内存占用量

-f 显示开机后启动的fork数量

-m 显示slabinfo的信息

-n 周期的显示数据

-s 显示内存的所有统计数据

-d 显示硬盘的统计数据

-D 以和-s类似的方式显示硬盘的总和的统计数据

-p 指定硬盘的分区显示磁盘的读写情况

-S 指定单位是k,K,m,M 

vmstat 的输出解释:

 * r 等待运行的线程数

 * b 多少个线程是处在不可中断的休眠中

 * swpd 虚拟内存的使用量

 * free和buffer和cache就不解释了

 * si 从磁盘swapin的内存量

 * so swapout到磁盘的内存量

 * bi 从磁盘设备收到的块数

 * bo 发送到磁盘设备的块数

 * in 每秒钟系统的中断数

 * cs 每秒中系统的上下文环境切换的数

 * us cpu时间片花费在非内核代码上的比例,即用户占用的cpu时间片

 * sy 内核占用的cpu时间片

 * id cpu时间片的空闲比例

 * wa cpu时间片花费在等待I/O的比例

 * st cpu时间片用在等待被虚拟化调用上的比例

大致的命令就介绍这些,监控的命令还有些很好用的htop和lsof,netstat,还有专门用于内存数据分析的memdump等

相关链接   
 *  [系统性能分析的实践方法][1]
 *  [鸟哥的Linux私房菜.基础学习篇][鸟哥的Linux私房菜.基础学习篇] 
 *  [UNIX and Linux System Administration Handbook][UNIX and Linux System Administration Handbook]
[1]:http://www.binospace.com/index.php/system-performance-analysis-practices/
[鸟哥的Linux私房菜.基础学习篇]: http://book.douban.com/subject/4889838/ 
[UNIX and Linux System Administration Handbook]: http://book.douban.com/subject/1779429/
