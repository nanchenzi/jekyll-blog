---
layout: post
title:  "zookeeper 服务端启动过程解析"
date:   2013-12-08 15:31:16
---
zookeeper 服务端入口程序位于:org.apache.zookeeper.server.quorum.QuorumPeerMain

###1)初始化配置和清理旧日志和旧数据文件

初始化QuorumPeerMain实例调用initializeAndRun初始化zk服务端配置参数

参数配置对应的解析封装类为QuorumPeerConfig,调用该类的parse函数解析参数配置文件;即zoo.cfg文件

*常规的zoo.cfg配置文件*
{% highlight java %}
dataDir=/var/zookeeper   //datalog和snaplog的存放路径
clientPort=2181         //面向客户端服务的端口
server.0=DataCenter09:2888:3888   //0代表server的sid;DataCenter09对应机器的hostname;2888服务端的通信端口;3888leader的选举端口
server.1=DataCenter10:2888:3888
server.2=DataCenter11:2888:3888
initLimit=10
syncLimit=5
tickTime=9000
{% endhighlight %}

DatadirCleanupManager 负责旧日志和数据文件的清理类
{% highlight java %}
DatadirCleanupManager purgeMgr = new DatadirCleanupManager(config.getDataDir(),
	 config.getDataLogDir(), 
	 config.getSnapRetainCount(),  //保留的旧日志和数据文件数目
         config.getPurgeInterval());  //任务运行的周期，单位为小时;默认为0所以不运行清理任务;
{% endhighlight %}
zoo.cfg中配置autopurge.purgeInterval=x,设置清理程序在x小时间隔运行

###2)运行服务端server实例

服务端采用NIO组建服务端通,即NIOServerCnxnFactory

`cnxnFactory.configure(config.getClientPortAddress(),config.getMaxClientCnxns());`  

默认的连接客户端port为2181,允许的最大客户端连接数为配置的60;

初始化QuorumPeer实例,this class manages the quorum protocol.There are three states this server
1. Leader election - each server will elect a leader (proposing itself as a leader initially).
2. Follower - the server will synchronize with the leader and replicate any transactions.
3. Leader - the server will process requests and forward them to followers.

QuorumPeer实现QuorumStats.Provider接口,用于提供当前Quorum的状态,初始化的状态为QuorumStats.Provider.UNKNOWN _ STATE,QuorumPeer实例从QuorumPeerConfig读取配置参数,运行start函数

{% highlight java %}
public synchronized void start() {
loadDataBase();  //加载数据
cnxnFactory.start();  //运行服务端实例
startLeaderElection(); //开始选举
super.start();  //运行线程run函数
}
{% endhighlight %}

###3)loadDataBase()从snap和log目录加载数据

ZKDatabase维护zookeeper在内存中的数据，启动时分别从数据和日志目录加载数据

DataTree对应数据的树型结构

sessionsWithTimeouts实例维护客户端的session

`snapLog.deserialize(dt, sessions);`从snap磁盘存放目录读取最近的100个有效文件，从旧至新的恢复到DataTree和sessionWithTimeouts当中;

`FileTxnLog txnLog = new FileTxnLog(dataDir);TxnIterator itr = txnLog.read(dt.lastProcessedZxid+1);long highestZxid = dt.lastProcessedZxid;`
根据snap文件读取的最大dt.lastProcessedZxid，加载日志目录中所有大于该id的文件,根据zxid递增排序,迭代的进行日志恢复操作;
{% highlight java %}
        ...
        TxnHeader hdr;
        while (true) {
	  ...
            try {
                processTransaction(hdr,dt,sessions, itr.getTxn());//根据记录的日志对sessions和dataTree做相应的修改
            } catch(KeeperException.NoNodeException e) {
               throw new IOException("Failed to process transaction type: " +
                     hdr.getType() + " error: " + e.getMessage(), e);
            }
            listener.onTxnLoaded(hdr, itr.getTxn());
            if (!itr.next()) //the iterator that moves to the next transaction
                break;
        }
{% endhighlight %}

###4)启动NIO服务端监听客户端连接端口

NIOServerCnxnFactory线程轮寻的从selecter中取得准备好的管道,ipMap维护客户端地址和对应的NIOServerCnxn实例,单个的ip地址客户端限制的服务数为maxClientCnxns数(默认为60);

{% highlight java %}
if ((k.readyOps() & SelectionKey.OP _ ACCEPT) != 0) {//新的连接请求创建对应的NIOServer和注册新管道
...
SelectionKey sk = sc.register(selector,SelectionKey.OP _ READ);
NIOServerCnxn cnxn = createConnection(sc, sk);
sk.attach(cnxn);
addCnxn(cnxn);
}else if ((k.readyOps() & (SelectionKey.OP _ READ | SelectionKey.OP _ WRITE)) != 0) {
   NIOServerCnxn c = (NIOServerCnxn) k.attachment();
   c.doIO(k);//通过针对客户端的NIOServerCnxn处理read或write数据buffer
 }
{% endhighlight %}

###5)leader选举

zk默认采用的选举类型(electionType)为FastLeaderElection

fastLeader的实现代码主要由FastLeaderElection和QuorumCnxManager两个类实现

<p><iframe id="embed_dom" name="embed_dom" frameborder="0" style="border:1px solid #000;display:block;width:720px; height:540px;" src="http://www.processon.com/embed/52a15fd80cf219c22501cde5">&nbsp;</iframe></p>

1. 启动QuorumCnxManager.Listener线程监听zk的选举端口，对于accept得到的sock，创建SendWorker和RecvWorker放入SenderWorkerMap中;zk集群选举过程中每台机器都会创建sock连接至其他peer，过程中两台peer之间存在两个管道互相连接至对方，例如A至B的sock，B至A的sock,两台peer之间的通信通过一个sock就足够进行互相之间的读写操作，所以listener在端口上监听到sock后，读取该sock的源sid，判定该sid和my.sid的大小，如果my.sid大，则关闭这个sock，本机重新发起到该sid的sock放入senderworkermap中,也就是zk选举过程peer的两两通信sock都由大的sid发起至小的sid;

2. 每个senderworker都有一个唯一的sid，即表明此senderworker是针对该peer的发送线程，运行的过程中根据该sid从queueSendMap得到ArrayBlockingQueue阻塞队列，从队列读取buffer写入管道中;而该sid的RecvWorker则负责从sock读取数据，封装成Message(包含sid和buffer)放入QuorumCnxManager类的recvQueue中(FastLeaderElection也包含一个类似的变量名recvqueue)

3. 初始化FastLeaderElection实例,启动该实例中的WorkerSender和WorkerReceiver线程;WorkerSender线程从sendqueue(LinkedBlockingQueue)取出数据根据sid放入queueSendMap中;WorkerReceiver从QuorumCnxManager的recvQueue队列中取出Message转换成Notification放入FastLeaderElection的recvqueue阻塞队列中

4. 由选举入口程序lookForLeader()方法开始选举过程peer之间的互相通信;变量recvset集合保存所有收到的peer的vote;方法轮寻的从recvqueue中取Notification;logicalclock代表每个选举阶段的epoch，如果此次选举过程完成，开始下一次选举过程，则logicalclock++;当peer落后于本次epoch则清空recvset中收到的vote,并更新提议的票为当前的Notification的sid，调用sendNotifications(),把peer的leader提议发给集群中的所有peer;在electionEpoch相同的情况下分别比较peerEpoch-->Zxid-->sid,如果当前peer的提议小于收到得proposal，则更新提议的票为当前的Notification的sid，把该提议发给zk集群中的所有peer;如果本次Notification并未更改peer原先的propsor，则把该消息转换成投票放入集合recvset中,否则continue取下一个Notification

5. 在recvset放入一条新vote后判定是否达到leader的条件;判断收到的vote集合中与peer自己提议的propose相同的vote数是否达到机器中的一半以上，未达到条件则继续continue取下一个Notification;当达到条件时，则要稍等finalizeWait的时间，取recvqueue剩下的数据，这样做的目的是可能存在某台更适合做leader机器延迟启动或者发送消息阻塞等情况导致该Notification再判定条件满足后到达，当判定该Notification比当前propsor更适合做leader时，把该Notification放回recvqueue，continue进行4过程的操作更改peer的提议leader;如果所有recvqueue取出的剩余的Notification没有更合适的提议，则根据提议的sid判定是否与本机的sid相同，当相同的时候则设置为leading，否则设置为FOLLOWING或者OBSERVING;

