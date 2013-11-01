---
layout: post
title:  "zookeeper client端实现解析"
date:   2013-10-31 21:50:16
---
zookeeper 客户端的实现主要由以下三个类完成:
 * org.apache.zookeeper.ZooKeeper
 * org.apache.zookeeper.ClientCnxn 
 * org.apache.zookeeper.ClientCnxnSocketNIO 

org.apache.zookeeper.ZooKeeper主要是一层api的封装,客户端程序用到一个Zookeeper实例就可以进行所有的操作

ZKWatchManager是在org.apache.zookeeper.ZooKeeper下的内部类,包含三个私有属性dataWatches、existWatches、childWatches, ZKWatchManager主要负责管理所有ClientCnxn从server集群上得到Watch事件

ClientCnxn是client端的核心实现,其中包含了两个轮寻的线程SendThread和EventThread,SendThread主要轮循从outgoingQueue队列中取得Zookeeper塞入的Packet包,通过ClientCnxnSocketNIO发送给服务器,并把发送的packet塞入pendingQueue队列中等待服务端的response,同时也从同服务端建立的管道中读取response把相应的packet移出pendingQueue,放入EventThrad负责处理的waitingEvents队列中,SendThread也负责和集群连接的建立、断开和session的ping连接,EventThread负责处理waitingEvent队列中packet,把packet中finished标识为true，使得阻塞的客户端函数返回并且取得packet中的response,根据不同的response调用不同的回调实现方法处理事件，其中waitingEvent队列采用LinkedBlockingQueue

ClientCnxnSocketNIO则是负责网络的通信，管道连接的建立，选择器的select操作,read和write的管道读写操作

<p><iframe id="embed_dom" name="embed_dom" frameborder="0" style="border:1px solid #000;display:block;width:510px; height:600px;" src="http://www.processon.com/embed/5271f9130cf22f64f63475b2" >&nbsp;</iframe></p>

### 连接的建立
参考org.apache.zookeeper.ZooKeeperMain中的main函数作为入口跟踪连接的建立
{% highlight java %}
 //ZooKeeper的构造函数
 public ZooKeeper(String connectString, int sessionTimeout, Watcher watcher,
            boolean canBeReadOnly)
        throws IOException
    {

        watchManager.defaultWatcher = watcher;  //默认的实现了process方法的watch

        ConnectStringParser connectStringParser = new ConnectStringParser(
                connectString);  //解析传入的hostsStr,用于指定chrootPath和生成多个InetSocketAddress集合
        HostProvider hostProvider = new StaticHostProvider(
                connectStringParser.getServerAddresses());  //提供InetSocketAddress的工具类
                                                            //其中的Collections.shuffle(this.serverAddresses)
                                                            //保证客户端请求集群中不同的机器，避免羊群效应
        cnxn = new ClientCnxn(connectStringParser.getChrootPath(),
                hostProvider, sessionTimeout, this, watchManager,
                getClientCnxnSocket(), canBeReadOnly);
	cnxn.start(); //启动线程sendThread和eventThread
    }
  
   //ClinetCnxn的构造函数
    public ClientCnxn(String chrootPath, HostProvider hostProvider, int sessionTimeout, ZooKeeper zooKeeper,
            ClientWatchManager watcher, ClientCnxnSocket clientCnxnSocket,
            long sessionId, byte[] sessionPasswd, boolean canBeReadOnly) {
        this.zooKeeper = zooKeeper;
        this.watcher = watcher;
        this.sessionId = sessionId;  //初始为0 
        this.sessionPasswd = sessionPasswd; //初始为new byte[16]
        this.sessionTimeout = sessionTimeout; //设置为3000ms
        this.hostProvider = hostProvider;
        this.chrootPath = chrootPath;

        connectTimeout = sessionTimeout / hostProvider.size();  //连接的timeout设置为sessionTimeOut除以InetSockAddress集合大小
                                                                //size越大，连接timeout的值越小
        readTimeout = sessionTimeout * 2 / 3;     //读的timeout设为sessionTimeout的三分之二
        readOnly = canBeReadOnly;

        sendThread = new SendThread(clientCnxnSocket);
        eventThread = new EventThread();

    }   

{% endhighlight %}

在sendThread中States属性用于标识客户端与集群的连接状态,初始为NOT-CONNECTED,在线程的run方法中创建SocketChanel,并向服务端发送connect的请求消息，在read到服务端的response消息后将state修改为CONNECTED或者CONNECTEDREADONLY

{% highlight java %}
                   //sendThread轮循的代码
                    if (!clientCnxnSocket.isConnected()) {  //判断ClinetCnxnSocketNIO实现类中的管道选择建是否创建，第一次运行为空进入函数
                        if(!isFirstConnect){ //如果不是第一建立连接则休眠一定的时间
                            try {
                                Thread.sleep(r.nextInt(1000));
                            } catch (InterruptedException e) {
                                LOG.warn("Unexpected exception", e);
                            }
                        }
                        // don't re-establish connection if we are closing
                        if (closing || !state.isAlive()) {
                            break;
                        }
                        startConnect(); //将state置为CONNECTING,表示连接进行中，并且通过hostProvider提供的InetSockAddress建立管道
                                        //向selector中注册关心OP_CONNECT的管道
                        clientCnxnSocket.updateLastSendAndHeard(); //更新客户端发送和接收的时间搓
                    }

                    if (state.isConnected()) {
                        //...省略了zooKeeperSaslClient的部分代码
			to = readTimeout - clientCnxnSocket.getIdleRecv(); //IdleRecv表示上次收到消息和now的间隔
                    } else {
                        to = connectTimeout - clientCnxnSocket.getIdleRecv();
                    }
                    
                    if (to <= 0) {  //小于0表示间隔大于timeout则session失效，抛出异常重新进行连接
                        throw new SessionTimeoutException(
                                "Client session timed out, have not heard from server in "
                                        + clientCnxnSocket.getIdleRecv() + "ms"
                                        + " for sessionid 0x"
                                        + Long.toHexString(sessionId));
                    }
                    if (state.isConnected()) {
                        int timeToNextPing = readTimeout / 2
                                - clientCnxnSocket.getIdleSend();  //在连接已经建立的条件下是否需要发送ping消息保持session
                        if (timeToNextPing <= 0) {
                            sendPing();
                            clientCnxnSocket.updateLastSend();
                        } else {
                            if (timeToNextPing < to) {
                                to = timeToNextPing;
                            }
                        }
                    }

                    // If we are in read-only mode, seek for read/write server
                    if (state == States.CONNECTEDREADONLY) { 
                        long now = System.currentTimeMillis();
                        int idlePingRwServer = (int) (now - lastPingRwServer);
                        if (idlePingRwServer >= pingRwTimeout) {
                            lastPingRwServer = now;
                            idlePingRwServer = 0;
                            pingRwTimeout =
                                Math.min(2*pingRwTimeout, maxPingRwTimeout);
                            pingRwServer(); //由hostProvider得到集群中的另一个InetSockAddress直接建立sock得到outputStream发送‘isro’
                                           //判断该地址是否是rw的服务器,在是的情况下抛出异常重新连接该地址rwServerAddress
                        }
                        to = Math.min(to, pingRwTimeout - idlePingRwServer);
                    }
            clientCnxnSocket.doTransport(to, pendingQueue, outgoingQueue, ClientCnxn.this);//调用clientCnxnSocketNIO发送消息
                
{% endhighlight %}

clientCnxnSocketNIO中的doTransport主要完成选择建的select()操作获得准备好的通道进行相应的操作，doIo则负责通道的读和写,这也是完成网络通信的主要方法

{% highlight java %}
  
 void doTransport(int waitTimeOut, List<Packet> pendingQueue, LinkedList<Packet> outgoingQueue,
                     ClientCnxn cnxn)
            throws IOException, InterruptedException {
        selector.select(waitTimeOut); //阻塞的等待相应的时间
        Set<SelectionKey> selected;
        synchronized (this) {
            selected = selector.selectedKeys();  //获得准备好的SelectionKey集合
        }
        // Everything below and until we get back to the select is
        // non blocking, so time is effectively a constant. That is
        // Why we just have to do this once, here
        updateNow();  //之所以在这更新now的时间是因为之前的所有操作都是非阻塞的
        for (SelectionKey k : selected) {
            SocketChannel sc = ((SocketChannel) k.channel());
            if ((k.readyOps() & SelectionKey.OP_CONNECT) != 0) { //第一次连接时设置的key仅关心连接
                if (sc.finishConnect()) {
                    updateLastSendAndHeard();
                    sendThread.primeConnection();  //添加conReq的Packet到outgoingQueue队列中等待下次发送
						 //并且enableReadWriteOnly,等待sendThread下一次调用doTransport,进而进入下面的doIO方法的调用
                }
            } else if ((k.readyOps() & (SelectionKey.OP_READ | SelectionKey.OP_WRITE)) != 0) {
                doIO(pendingQueue, outgoingQueue, cnxn); //调用doIo发送或者读取消息
            }
        }
        if (sendThread.getZkState().isConnected()) { //在连接的条件下保证outgoingQueue有数据时enableWrite
            synchronized(outgoingQueue) {
                if (findSendablePacket(outgoingQueue,
                        cnxn.sendThread.clientTunneledAuthenticationInProgress()) != null) {
                    enableWrite();
                }
            }
        }
        selected.clear();  //清楚已经处理的建
    }

   
   void doIO(List<Packet> pendingQueue, LinkedList<Packet> outgoingQueue, ClientCnxn cnxn)
      throws InterruptedException, IOException {
        SocketChannel sock = (SocketChannel) sockKey.channel();
        if (sock == null) {
            throw new IOException("Socket is null!");
        }
        if (sockKey.isReadable()) {  //是否有可读的数据
            int rc = sock.read(incomingBuffer);
            if (rc < 0) {
                throw new EndOfStreamException(
                        "Unable to read additional data from server sessionid 0x"
                                + Long.toHexString(sessionId)
                                + ", likely server has closed socket");
            }
            if (!incomingBuffer.hasRemaining()) {
                incomingBuffer.flip(); 
                if (incomingBuffer == lenBuffer) {
                    recvCount++;
                    readLength();  //读取数据的长度，调用ByteBuffer重新分配incomingBuffer的长度
                } else if (!initialized) {  //在连接未建立时,initialized为false
                    readConnectResult();   //读取response建立连接
                    enableRead();
                    if (findSendablePacket(outgoingQueue,
                            cnxn.sendThread.clientTunneledAuthenticationInProgress()) != null) {
                        // Since SASL authentication has completed (if client is configured to do so),
                        // outgoing packets waiting in the outgoingQueue can now be sent.
                        enableWrite();
                    }
                    lenBuffer.clear();
                    incomingBuffer = lenBuffer;
                    updateLastHeard();
                    initialized = true;  //初始化完成
                } else {
                    sendThread.readResponse(incomingBuffer); //当连接建立时直接读取消息
                    lenBuffer.clear();
                    incomingBuffer = lenBuffer;
                    updateLastHeard();
                }
            }
        }
        if (sockKey.isWritable()) {   //写入的管道可用
            synchronized(outgoingQueue) {
                Packet p = findSendablePacket(outgoingQueue,
                        cnxn.sendThread.clientTunneledAuthenticationInProgress());  //得到首个需要发送的Packet

                if (p != null) {
                    updateLastSend();
                    // If we already started writing p, p.bb will already exist
                    if (p.bb == null) {
                        if ((p.requestHeader != null) &&
                                (p.requestHeader.getType() != OpCode.ping) &&
                                (p.requestHeader.getType() != OpCode.auth)) {
                            p.requestHeader.setXid(cnxn.getXid());  //ping和auth的消息不需要发送xid
                        }
                        p.createBB();
                    }
                    sock.write(p.bb); //写入数据
                    if (!p.bb.hasRemaining()) {
                        sentCount++;
                        outgoingQueue.removeFirstOccurrence(p); //当消息完全写入后将Packet从outgoingQueue中移除
                        if (p.requestHeader != null
                                && p.requestHeader.getType() != OpCode.ping
                                && p.requestHeader.getType() != OpCode.auth) {
                            synchronized (pendingQueue) {
                                pendingQueue.add(p);  //如果不是ping和auth的消息则放入pendingQueue中
                            }
                        }
                    }
                }
                if (outgoingQueue.isEmpty()) { //判断outgoingQueue是否为空,空则disableWrite,反之亦然
                    disableWrite();
                } else {
                    enableWrite();
                }
            }
        }
    }
  
{% endhighlight %}

readConnectResult方法最终会调用sendThread中的onConnected完成连接

{% highlight java  %}

 void onConnected(int _negotiatedSessionTimeout, long _sessionId,
                byte[] _sessionPasswd, boolean isRO) throws IOException {
            negotiatedSessionTimeout = _negotiatedSessionTimeout;
	      ...
            if (!readOnly && isRO) { //客户端设置是可读写的但是服务端仅是只读记入错误
                LOG.error("Read/write client got connected to read-only server");
            }
            readTimeout = negotiatedSessionTimeout * 2 / 3;  //根据服务端回复的sessionTimeou重新设置这两个值
            connectTimeout = negotiatedSessionTimeout / hostProvider.size();
            hostProvider.onConnected();
            sessionId = _sessionId;  //客户端的sessionId设置为服务端分配的sessionId
            sessionPasswd = _sessionPasswd;  //密码也设置为服务端提供的
            state = (isRO) ?
                    States.CONNECTEDREADONLY : States.CONNECTED; //根据服务端的是否可读写设置state的状态
            seenRwServerBefore |= !isRO;
            KeeperState eventState = (isRO) ?
                    KeeperState.ConnectedReadOnly : KeeperState.SyncConnected;
            eventThread.queueEvent(new WatchedEvent( //将事件放入waittingQueue中待EventThread线程处理
                    Watcher.Event.EventType.None,
                    eventState, null));
        }

{% endhighlight %}


###客户端发送一个create请求

客户端程序通过调用Zookeeper的create函数发送create的Packet,函数等待Packet的完成

{% highlight java%}
      
    	RequestHeader h = new RequestHeader();//请求的头消息
        h.setType(ZooDefs.OpCode.create); //设置请求头消息的类型
        CreateRequest request = new CreateRequest(); //请求的消息封装
        CreateResponse response = new CreateResponse(); //返回消息的封装
        request.setData(data); //塞入创建的数据
        request.setFlags(createMode.toFlag());//创建节点的类型
        request.setPath(serverPath); //服务端路径
        if (acl != null && acl.size() == 0) {
            throw new KeeperException.InvalidACLException();
        }
        request.setAcl(acl); //acl控制权限
        ReplyHeader r = cnxn.submitRequest(h, request, response, null); //利用cnxn提交请求

	
    	public ReplyHeader submitRequest(RequestHeader h, Record request,
            Record response, WatchRegistration watchRegistration)
            throws InterruptedException {
        	ReplyHeader r = new ReplyHeader();  //返回的头消息封装
	        Packet packet = queuePacket(h, r, request, response, null, null, null,
                    null, watchRegistration); //封装好发送的Packet,往OutgoingQueue提交等待处理
        	synchronized (packet) {
	            while (!packet.finished) {  //调用函数等待直到服务端响应消息完成
        	        packet.wait();
	            }
	        }
        	return r;
	    }

{% endhighlight %}

接着进入上边sengThread提到的轮循处理的过程，待管道读到服务端的响应后进入sendThread.readResponse(incomingBuffer)方法，完成消息的响应的处理过程

{% highlight java%}
	
	void readResponse(ByteBuffer incomingBuffer) throws IOException {
        
	    ByteBufferInputStream bbis = new ByteBufferInputStream(
                    incomingBuffer);
            BinaryInputArchive bbia = BinaryInputArchive.getArchive(bbis);
            ReplyHeader replyHdr = new ReplyHeader();
            replyHdr.deserialize(bbia, "header"); //反序列化得到返回的头消息
            if (replyHdr.getXid() == -2) {
                // -2 is the xid for pings
		//-2 表示ping的消息回馈，再debug的情况下记录日志然后返回不进行其他操作
                return;
            }
            if (replyHdr.getXid() == -4) {
                // -4 is the xid for AuthPacket               
                if(replyHdr.getErr() == KeeperException.Code.AUTHFAILED.intValue()) {
                    state = States.AUTH_FAILED;   //向waittingQueue丢入授权失败的event                 
                    eventThread.queueEvent( new WatchedEvent(Watcher.Event.EventType.None, 
                            Watcher.Event.KeeperState.AuthFailed, null) ); //将会从WathcerManager中得到所有的wathch进行处理      		            		
                }
                return;
            }
            if (replyHdr.getXid() == -1) {
                // -1 means notification
                WatcherEvent event = new WatcherEvent();
                event.deserialize(bbia, "response");
		...
                WatchedEvent we = new WatchedEvent(event);
                eventThread.queueEvent( we );  //该方法会从WatcherManager中得到所管理的响应的event
					       //然后将event封装成WatcherSetEventPair丢入waittingQueue中等待EventThread的处理
                return;
            }
	   ...

            Packet packet;
            synchronized (pendingQueue) {
                if (pendingQueue.size() == 0) {
                    throw new IOException("Nothing in the queue, but got "
                            + replyHdr.getXid());
                }
                packet = pendingQueue.remove();  //从pendingQueue中移除等待响应的Packet
            }
            /*
             * Since requests are processed in order, we better get a response
             * to the first request!
             */
            try {
                if (packet.requestHeader.getXid() != replyHdr.getXid()) {  //当请求的xid与服务端的xid不相等时,标识错误,抛出失去连接的错误
                    packet.replyHeader.setErr(
                            KeeperException.Code.CONNECTIONLOSS.intValue());
                    throw new IOException("Xid out of order. Got Xid "
                            + replyHdr.getXid() + " with err " +
                            + replyHdr.getErr() +
                            " expected Xid "
                            + packet.requestHeader.getXid()
                            + " for a packet with details: "
                            + packet );
                }

                packet.replyHeader.setXid(replyHdr.getXid()); //将返回的头消息放回等待的packet中
                packet.replyHeader.setErr(replyHdr.getErr());
                packet.replyHeader.setZxid(replyHdr.getZxid());
                if (replyHdr.getZxid() > 0) {
                    lastZxid = replyHdr.getZxid();  //更新最后的lastZxid
                }
                if (packet.response != null && replyHdr.getErr() == 0) {
                    packet.response.deserialize(bbia, "response");
                }
            } finally {
                finishPacket(packet);  //调用该函数完成packet的最后一个步骤
            }
        }

    private void finishPacket(Packet p) {
        if (p.watchRegistration != null) {
            p.watchRegistration.register(p.replyHeader.getErr()); //当返回的消息正确的情况下将watch放入WatcherManager中
        }
        if (p.cb == null) {  //如果Packet未设置回调函数则标识完成通知等待的线程
            synchronized (p) {
                p.finished = true; 
                p.notifyAll();
            }
        } else {
            p.finished = true; //标识完成
            eventThread.queuePacket(p); //将packet丢入waittingQueue中等待EventThread调用相应的回调方法
        }
    }	
{% endhighlight %}

最后在看一下EventThread对waittingQueue所做的操作

{% highlight java%}
     private void processEvent(Object event) {
          try {
              if (event instanceof WatcherSetEventPair) {   //对于event的操作根据类型分为两类
							//第一是先前封装的WatcherSetEvetnPair针对返回的头消息是-4和-1所做的操作
						        //根据返回的WatchManager所管理的Watch分别调用各自的process函数处理
                  WatcherSetEventPair pair = (WatcherSetEventPair) event;
                  for (Watcher watcher : pair.watchers) {
                      try {
                          watcher.process(pair.event);
                      } catch (Throwable t) {
                          LOG.error("Error while calling watcher ", t);
                      }
                  }
              } else {  
                  //第二种是包含cb所进行的回调处理
                  //根据Packet中设置的返回消息回调类型通过cb来完成
                  Packet p = (Packet) event;
                  int rc = 0;
                  String clientPath = p.clientPath;
                  if (p.replyHeader.getErr() != 0) {
                      rc = p.replyHeader.getErr();
                  }
                  if (p.cb == null) {
                      LOG.warn("Somehow a null cb got to EventThread!");
                  } else if (p.response instanceof ExistsResponse
                          || p.response instanceof SetDataResponse
                          || p.response instanceof SetACLResponse) {
                      StatCallback cb = (StatCallback) p.cb;
                      if (rc == 0) {
                          if (p.response instanceof ExistsResponse) {
                              cb.processResult(rc, clientPath, p.ctx,
                                      ((ExistsResponse) p.response)
                                              .getStat());
                          } else if (p.response instanceof SetDataResponse) {
                              cb.processResult(rc, clientPath, p.ctx,
                                      ((SetDataResponse) p.response)
                                              .getStat());
                          } else if (p.response instanceof SetACLResponse) {
                              cb.processResult(rc, clientPath, p.ctx,
                                      ((SetACLResponse) p.response)
                                              .getStat());
                          }
                      } else {
                          cb.processResult(rc, clientPath, p.ctx, null);
                      }
                    ...
              }
          } catch (Throwable t) {
              LOG.error("Caught unexpected throwable", t);
          }
       }
    }

{% endhighlight %}

针对Zookeeper客户端的实现逻辑和主要代码段介绍完了,看似简单但真正把代码都介绍完才慢慢体会到里边很多的细节,也算是真正意义上的看懂了,总觉着这篇文章的代码贴的太多,看得不是很舒服,下篇介绍Zookeeper实现的方式看看是否能换一种更好的方式写出来,很多事总要经历那么一个过程...

相关链接   
 *  [Zookeeper全解析——Client端][1]
 *  [Java NIO][2]
 *  [Java网络编程][3]
 *  [Jekyll中引入iframe的问题][4]
[1]: http://www.spnguru.com/2010/08/zookeeper%E5%85%A8%E8%A7%A3%E6%9E%90%E2%80%94%E2%80%94client%E7%AB%AF/
[2]: http://book.douban.com/subject/1433583/
[3]: http://book.douban.com/subject/1438754/
[4]: http://log.adamwilcox.org/2013/03/25/fixing-soundcloud-embeds-on-github-pages-jekyll/                                                    
