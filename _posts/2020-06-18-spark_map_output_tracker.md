---
layout: post
title:  "Spark源码学习笔记（八）"
date:   2020-06-18 21:47:56 +0800
tags:
- Spark
color: rgb(207, 207, 207)
cover: '/blog/Halloween/2020_05_14_10.webp'
subtitle: 'MapOutputTracker'
---
> MapOutputTracker: 用于跟踪shuffle阶段map输出的结果，序列化MapStauses与反序列化MapStatus，序列化时尝试使用直接序列化的方式，如果序列化后的结果满足广播的最小Size要求，则使用广播的方式序列化，不管是直接序列化还是通过广播的方式，都会使用Gzip压缩，最后返回字节数组，askTracker/sendTracker用于发送rpc请求获取MapStatus

```scala
def serializeMapStatuses(statuses: Array[MapStatus], broadcastManager: BroadcastManager,
      isLocal: Boolean, minBroadcastSize: Int): (Array[Byte], Broadcast[Array[Byte]]) = {
    val out = new ByteArrayOutputStream
    out.write(DIRECT)
    val objOut = new ObjectOutputStream(new GZIPOutputStream(out))
    Utils.tryWithSafeFinally {
      // Since statuses can be modified in parallel, sync on it
      statuses.synchronized {
        objOut.writeObject(statuses)
      }
    } {
      objOut.close()
    }
    val arr = out.toByteArray
    if (arr.length >= minBroadcastSize) {
      // Use broadcast instead.
      // Important arr(0) is the tag == DIRECT, ignore that while deserializing !
      val bcast = broadcastManager.newBroadcast(arr, isLocal)
      // toByteArray creates copy, so we can reuse out
      out.reset()
      out.write(BROADCAST)
      val oos = new ObjectOutputStream(new GZIPOutputStream(out))
      oos.writeObject(bcast)
      oos.close()
      val outArr = out.toByteArray
      logInfo("Broadcast mapstatuses size = " + outArr.length + ", actual size = " + arr.length)
      (outArr, bcast)
    } else {
      (arr, null)
    }
  }

  // Opposite of serializeMapStatuses.
  def deserializeMapStatuses(bytes: Array[Byte]): Array[MapStatus] = {
    assert (bytes.length > 0)

    def deserializeObject(arr: Array[Byte], off: Int, len: Int): AnyRef = {
      val objIn = new ObjectInputStream(new GZIPInputStream(
        new ByteArrayInputStream(arr, off, len)))
      Utils.tryWithSafeFinally {
        objIn.readObject()
      } {
        objIn.close()
      }
    }

    bytes(0) match {
      case DIRECT =>
        deserializeObject(bytes, 1, bytes.length - 1).asInstanceOf[Array[MapStatus]]
      case BROADCAST =>
        // deserialize the Broadcast, pull .value array out of it, and then deserialize that
        val bcast = deserializeObject(bytes, 1, bytes.length - 1).
          asInstanceOf[Broadcast[Array[Byte]]]
        logInfo("Broadcast mapstatuses size = " + bytes.length +
          ", actual size = " + bcast.value.length)
        // Important - ignore the DIRECT tag ! Start from offset 1
        deserializeObject(bcast.value, 1, bcast.value.length - 1).asInstanceOf[Array[MapStatus]]
      case _ => throw new IllegalArgumentException("Unexpected byte tag = " + bytes(0))
    }
  }

```

> MapStatus: ShuffleMapTask返回给scheduler的结果，其中包含了task运行位置信息，输出block块大小等信息
> 
> ShuffleStatus: 维护mapId到MapStatus的映射，通过addMapOutput/removeMapOutput添加/移除数组中的映射，在这里可能会有疑问，为什么使用数组存储映射，而不是map，因为rpc获取到结果反序列化后是MapStatus数组，使用了ZipWithIndex生成的mapId
> 
> MapOutputTrackerWorker使用getStatuses方法获取指定shuffleId的MapStatus信息，当本地获取不到，则发送RPC请求获取
> 
> MapOutputTrackerMasterEndpoint收到rpc请求后，会使用MapOutputTrackerMaster的post方法将GetMapOutputMessage请求加入队列中，等待消费
> 
> MapOutputTrackerMaster：创建threadpool消费队列中的GetMapOutputMessage请求，获取指定shuffleId的ShuffleStatus对象，将shuffleStatus对象中的MapStatus数组序列化后答复给请求方，我都怀疑这些代码是一个人写的了，每次都是MessageLoop + threadpool组合
> 
> MapOutputTrackerWorker获取到Master的答复后，反序列化数据得到MapStatus，并存到map中


```scala
private class MessageLoop extends Runnable {
    override def run(): Unit = {
      try {
        while (true) {
          try {
            val data = mapOutputRequests.take()
             if (data == PoisonPill) {
              // Put PoisonPill back so that other MessageLoops can see it.
              mapOutputRequests.offer(PoisonPill)
              return
            }
            val context = data.context
            val shuffleId = data.shuffleId
            val hostPort = context.senderAddress.hostPort
            logDebug("Handling request to send map output locations for shuffle " + shuffleId +
              " to " + hostPort)
            val shuffleStatus = shuffleStatuses.get(shuffleId).head
            context.reply(
              shuffleStatus.serializedMapStatus(broadcastManager, isLocal, minSizeForBroadcast))
          } catch {
            case NonFatal(e) => logError(e.getMessage, e)
          }
        }
      } catch {
        case ie: InterruptedException => // exit
      }
    }
  }
```
> 简单来说，就是MapOutputTrackerWorker发送RPC请求向MapOutputTrackerMaster获取指定shuffleId的MapStatus数据，MapOutputTrackerMaster消费请求消息，获取数据并序列化传输给MapOutputTrackerWorker的一个流程