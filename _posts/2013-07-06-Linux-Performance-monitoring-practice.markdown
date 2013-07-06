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

...未完待续

参考书籍   
 *  [鸟哥的Linux私房菜.基础学习篇][鸟哥的Linux私房菜.基础学习篇] 
 *  [UNIX and Linux System Administration Handbook][UNIX and Linux System Administration Handbook]
[鸟哥的Linux私房菜.基础学习篇]: http://book.douban.com/subject/4889838/ 
[UNIX and Linux System Administration Handbook]: http://book.douban.com/subject/1779429/
