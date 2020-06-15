---
layout: post
title:  "Spark源码学习笔记（六）"
date:   2020-06-11 00:11:56 +0800
tags:
- Spark
color: rgb(207, 207, 207)
cover: '/blog/Halloween/2020_05_14_12.webp'
subtitle: 'Netty RPC机制'
---
> spark 1.6之前通信采用的是Akka框架，并不是netty，从2.0版本开始，切换成了netty，关于这方面的原因，官方给的回答是社区中建议比较多，因为akka版本兼容性差，不同版本之间可能会有无法通信情况，给使用者造成了很大的困扰，基于这个原因，切换成了netty。
> 
> RPC方面的知识点会比较多，我也看了挺久。按照之前的思路，我们跟着SparkContext创建流程走，在SparkEnv中会使用NettyRpcEnvFactory创建NettyRpcEnv，启动driver的rpc服务
> 
> 主要用到的类如下
> 
> Dispatcher: 用作消息转发，和Inbox一起用，发送InboxMessage消息，验证endpoint是否存在等，采用的是策略模式，InboxMessage为最上层抽象Message，OneWayMessage,RpcMessage,OnStart,OnStop,RemoteProcessConnected等Message均继承自InboxMessage
> 
> ```scala
> 	/** Thread pool used for dispatching messages. */
  private val threadpool: ThreadPoolExecutor = {
    val availableCores =
      if (numUsableCores > 0) numUsableCores else Runtime.getRuntime.availableProcessors()
    val numThreads = nettyEnv.conf.getInt("spark.rpc.netty.dispatcher.numThreads",
      math.max(2, availableCores))
    val pool = ThreadUtils.newDaemonFixedThreadPool(numThreads, "dispatcher-event-loop")
    for (i <- 0 until numThreads) {
      pool.execute(new MessageLoop)
    }
    pool
  }
  /** Message loop used for dispatching messages. */
  private class MessageLoop extends Runnable {
    override def run(): Unit = {
      try {
        while (true) {
          try {
            val data = receivers.take()
            if (data == PoisonPill) {
              // Put PoisonPill back so that other MessageLoops can see it.
              receivers.offer(PoisonPill)
              return
            }
            data.inbox.process(Dispatcher.this)
          } catch {
            case NonFatal(e) => logError(e.getMessage, e)
          }
        }
      } catch {
        case ie: InterruptedException => // exit
      }
    }
  }
 /** Inbox中process代码
   * Process stored messages.
   */
  def process(dispatcher: Dispatcher): Unit = {
    var message: InboxMessage = null
    inbox.synchronized {
      if (!enableConcurrent && numActiveThreads != 0) {
        return
      }
      message = messages.poll()
      if (message != null) {
        numActiveThreads += 1
      } else {
        return
      }
    }
    while (true) {
      safelyCall(endpoint) {
        message match {
          case RpcMessage(_sender, content, context) =>
            try {
              endpoint.receiveAndReply(context).applyOrElse[Any, Unit](content, { msg =>
                throw new SparkException(s"Unsupported message $message from ${_sender}")
              })
            } catch {
              case NonFatal(e) =>
                context.sendFailure(e)
                // Throw the exception -- this exception will be caught by the safelyCall function.
                // The endpoint's onError function will be called.
                throw e
            }
          case OneWayMessage(_sender, content) =>
            endpoint.receive.applyOrElse[Any, Unit](content, { msg =>
              throw new SparkException(s"Unsupported message $message from ${_sender}")
            })
          case OnStart =>
            endpoint.onStart()
            if (!endpoint.isInstanceOf[ThreadSafeRpcEndpoint]) {
              inbox.synchronized {
                if (!stopped) {
                  enableConcurrent = true
                }
              }
            }
          case OnStop =>
            val activeThreads = inbox.synchronized { inbox.numActiveThreads }
            assert(activeThreads == 1,
              s"There should be only a single active thread but found $activeThreads threads.")
            dispatcher.removeRpcEndpointRef(endpoint)
            endpoint.onStop()
            assert(isEmpty, "OnStop should be the last message")
          case RemoteProcessConnected(remoteAddress) =>
            endpoint.onConnected(remoteAddress)
          case RemoteProcessDisconnected(remoteAddress) =>
            endpoint.onDisconnected(remoteAddress)
          case RemoteProcessConnectionError(cause, remoteAddress) =>
            endpoint.onNetworkError(cause, remoteAddress)
        }
      }
      inbox.synchronized {
        // "enableConcurrent" will be set to false after `onStop` is called, so we should check it
        // every time.
        if (!enableConcurrent && numActiveThreads != 1) {
          // If we are not the only one worker, exit
          numActiveThreads -= 1
          return
        }
        message = messages.poll()
        if (message == null) {
          numActiveThreads -= 1
          return
        }
      }
    }
  }
> ```
> InBox、InBoxMessage：NettyRpcEnv中send/ask方法发送至本地rpc endpoint使用，由endpoint处理接收到rpc消息，EndpointRef中的ask、send方法底层就是调用NettyRpcEvn中的ask、send方法
> 
> OutBox、OutBoxMessage: NettyRpcEnv中send/ask方法发送至远程rpc endpoint使用，底层通过TransportClient发送，由RpcHandler处理接收到的rpc消息，当发送请求的client和server在同一台机器上则使用inbox，否则使用outbox
> 
> RequestMessage中包含了InBoxMessage，OutBoxMessage需要的属性，可以用RequestMessage中的属性生成后面的对象，其并不是继承关系
> 
> ```scala
>  private[netty] def send(message: RequestMessage): Unit = {
    val remoteAddr = message.receiver.address
    if (remoteAddr == address) {
      // Message to a local RPC endpoint.
      try {
        dispatcher.postOneWayMessage(message)
      } catch {
        case e: RpcEnvStoppedException => logDebug(e.getMessage)
      }
    } else {
      // Message to a remote RPC endpoint.
      postToOutbox(message.receiver, OneWayOutboxMessage(message.serialize(this)))
    }
  }
> ```
> TransportContext：创建TransportContext时需要参数NettyRpcHandler，而创建NettyRpcHandler需要Dispatcher
> 
> NettyRpcHandler继承RpcHandler，该类用于处理TransportClient中方法sendRPC()发送的RPC请求，即client通过netty rpc发送到server的请求
> 
> TransportClient用于和netty server通信
>
>NettyRpcEnv为主要入口类，类中有startServer方法用于启动服务，使用TransportContext创建server, Dispatcher注册RpcEndpoint， setupEndpoint方法也是用Dispatcher注册RpcEndpoint，调用Dispatcher的registerRpcEndpoint，asyncSetupEndpointRefByURI方法用于创建EndPointRef，在SparkEnv中有使用到，当不是driver端时，使用RpcUtils.makeDriverRef方法
>
>```scala
>  def registerOrLookupEndpoint(
	    name: String, endpointCreator: => RpcEndpoint):
	  RpcEndpointRef = {
	  if (isDriver) {
	    logInfo("Registering " + name)
	    rpcEnv.setupEndpoint(name, endpointCreator)
	  } else {
	    RpcUtils.makeDriverRef(name, conf, rpcEnv)
	  }
	}
	/**
	*  RpcUtils.makeDriverRef方法
   * Retrieve a `RpcEndpointRef` which is located in the driver via its name.
   */
  def makeDriverRef(name: String, conf: SparkConf, rpcEnv: RpcEnv): RpcEndpointRef = {
    val driverHost: String = conf.get("spark.driver.host", "localhost")
    val driverPort: Int = conf.getInt("spark.driver.port", 7077)
    Utils.checkHost(driverHost)
    rpcEnv.setupEndpointRef(RpcAddress(driverHost, driverPort), name)
  }
 /**
   * Retrieve the [[RpcEndpointRef]] represented by `address` and `endpointName`.
   * This is a blocking action.
   */
  def setupEndpointRef(address: RpcAddress, endpointName: String): RpcEndpointRef = {
    setupEndpointRefByURI(RpcEndpointAddress(address, endpointName).toString)
  }
 /**
   * Retrieve the [[RpcEndpointRef]] represented by `uri`. This is a blocking action.
   */
  def setupEndpointRefByURI(uri: String): RpcEndpointRef = {
    defaultLookupTimeout.awaitResult(asyncSetupEndpointRefByURI(uri))
  }
  // 会调用NettyRpcEnv中的实现
  def asyncSetupEndpointRefByURI(uri: String): Future[RpcEndpointRef]
   >```
>
>再讲讲EndPoint和EndPointRef吧，刚开始看我也有点懵，不知道是什么概念，其实就是EndPoint存在于driver端，EndPointRef存在于executor中，是EndPoint的一个引用，使用registerRpcEndpoint会将endpoint和endpointRef都封装到map中，可以根据endpoint找到与之对应的endpointRef
>
>```scala
>private class EndpointData(
      val name: String,
      val endpoint: RpcEndpoint,
      val ref: NettyRpcEndpointRef) {
    val inbox = new Inbox(ref, endpoint)
  }
private val endpoints: ConcurrentMap[String, EndpointData] =
    new ConcurrentHashMap[String, EndpointData]
>private val endpointRefs: ConcurrentMap[RpcEndpoint, RpcEndpointRef] =
    new ConcurrentHashMap[RpcEndpoint, RpcEndpointRef]
>def registerRpcEndpoint(name: String, endpoint: RpcEndpoint): NettyRpcEndpointRef = {
    val addr = RpcEndpointAddress(nettyEnv.address, name)
    val endpointRef = new NettyRpcEndpointRef(nettyEnv.conf, addr, nettyEnv)
    synchronized {
      if (stopped) {
        throw new IllegalStateException("RpcEnv has been stopped")
      }
      if (endpoints.putIfAbsent(name, new EndpointData(name, endpoint, endpointRef)) != null) {
        throw new IllegalArgumentException(s"There is already an RpcEndpoint called $name")
      }
      val data = endpoints.get(name)
      endpointRefs.put(data.endpoint, data.ref)
      receivers.offer(data)  // for the OnStart message
    }
    endpointRef
  }
  def getRpcEndpointRef(endpoint: RpcEndpoint): RpcEndpointRef = endpointRefs.get(endpoint)
>```
>
>整个的大体流程就是EndpointRef发送消息出去，会调用NettyRpcEnv中的send/ask方法，在NettyRpcEnv中会判断当前是否是消息接收者和当前的address是否相同，如果相同则使用inbox消息，dispather直接转发给inbox处理；如果address不同，则使用outbox消息，底层使用TransportClient发送消息，NettyRpcHandler会处理接收到的消息，然后使用dispather转发给inbox处理
> outbox用于client端，inbox用于server端