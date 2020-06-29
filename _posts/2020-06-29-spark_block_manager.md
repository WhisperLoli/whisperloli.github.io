---
layout: post
title:  "Spark源码学习笔记（十一）"
date:   2020-06-29 21:55:25 +0800
tags:
- Spark
color: rgb(207, 207, 207)
cover: '/blog/Halloween/2020_05_14_07.webp'
subtitle: 'BlockManager'
---

> BlockManagerMaster作用：使用driverEndpoint(RpcEndpointRef)发送请求处理某件事物registerBlockManager，updateBlockInfo等等，BlockManagerMasterEndpoint中会收到消息并做出对应的处理，有些处理（比如removeRdd、removeShuffle等操作）会涉及executor，就会调用到slaveEndpoint(RpcEndpointRef)发送请求给executor，BlockManagerSlaveEndpoint收到消息会在executor上做出对应的处理
> 
> BlockManagerMaster中的部分方法会被BlockManager调用到，例如registerBlockManager、updateBlockInfo之类，其他的方法会被外界调用，如dagScheduler等
> 
> BlockId用于标识一个block是何种类型的block数据，会给每种不同类型的block分配不同的id，产生唯一的一组键，我们常见的就是RDD block,SHUFFLE block,BROADCAST block,BlockId也提供了方法可以判断是一个blockId是不是RDDBlockID、ShuffleBlockId、BroadcastBlockId
> 
> BlockManagerId：BlockManager的ID，唯一标识符，类似BlockId与Block的关系，每个executor都会有一个单独的blockManager
> 
> BlockInfoManager：用于跟踪每个block metadata及管理block的锁，readLocksByTask/writeLocksByTask会记录所有task读写加锁操作
> 
> NettyBlockTransferService：用于fetch/upload数据，会使用到NettyBlockRpcServer，NettyBlockRpcServer也继承自RpcHandler，发送的上传/下载block消息都会被NettyBlockRpcServer处，NettyBlockTransferService中的fetchBlocks/uploadBlock都会使用到NettyBlockRpcServer生成的client发送消息

```scala
  override def receive(
      client: TransportClient,
      rpcMessage: ByteBuffer,
      responseContext: RpcResponseCallback): Unit = {
    val message = BlockTransferMessage.Decoder.fromByteBuffer(rpcMessage)
    logTrace(s"Received request: $message")

    message match {
      case openBlocks: OpenBlocks =>
        val blocksNum = openBlocks.blockIds.length
        val blocks = for (i <- (0 until blocksNum).view)
          yield blockManager.getBlockData(BlockId.apply(openBlocks.blockIds(i)))
        val streamId = streamManager.registerStream(appId, blocks.iterator.asJava)
        logTrace(s"Registered streamId $streamId with $blocksNum buffers")
        responseContext.onSuccess(new StreamHandle(streamId, blocksNum).toByteBuffer)

      case uploadBlock: UploadBlock =>
        // StorageLevel and ClassTag are serialized as bytes using our JavaSerializer.
        val (level: StorageLevel, classTag: ClassTag[_]) = {
          serializer
            .newInstance()
            .deserialize(ByteBuffer.wrap(uploadBlock.metadata))
            .asInstanceOf[(StorageLevel, ClassTag[_])]
        }
        val data = new NioManagedBuffer(ByteBuffer.wrap(uploadBlock.blockData))
        val blockId = BlockId(uploadBlock.blockId)
        blockManager.putBlockData(blockId, data, level, classTag)
        responseContext.onSuccess(ByteBuffer.allocate(0))
    }
  }

```

> BlockManager：需要注册到BlockManagerMaster中，该类会运行在每一个节点（driver和executor）中
，并对外提供了接口供get/put blocks，block在本地或者远程均可。还有就是对BlockInfo中的方法做了一层封装，通过该类才能调用BlockInfo中的方法对block加锁

> BlockManager中虽然方法很多，但是很多都是会被spark内部其他部分调用，类似于SparkContext提供很多方法，被用户调用一样，Spark内部其他部分就是BlockManager的用户