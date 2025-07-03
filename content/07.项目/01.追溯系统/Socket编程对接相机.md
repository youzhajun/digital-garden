---
title: Socket编程对接相机
draft: false
tags:
  - java
  - "#项目"
date: 2024-10-15
---
# 项目介绍
基于条码的工厂产品全流程（生产、装箱、经销）溯源系统，对接高速工业扫码相机，实时采集数据，防止经销商跨区域串货。

# 问答
## 采用了哪种通信协议，为什么？

在 `TCP` 与 `UDP` 之间选择了 `TCP`。

>`TCP` 的特点：

- **面向连接** ： 三次握手，四次挥手
- **可靠传输：** 
	- 数据包确认
	- 超时重传
	- 流量控制
	- 拥塞控制
- **面向字节流：**
	- `TCP` 将应用程序交付的数据视为一个没有明确边界的、连续的字节序列。应用程序需要自己处理消息的“边界”问题。举个🌰：通过 TCP 发送了 “`Hello`” 和 “`World`” 两个字符串，接收方可能会一次性收到 “`HelloWorld`”，也可能先收到 “`HelloW`”，再收到 “`orld`”。

> `UDP` 的特点

 - **无连接：**
	 - 发送消息前不需要建立链接，只知道目标地址，直接扔。
 - **不可靠传输：**
	 - 不一定保证送到地方
	 - 没有 TCP 那套可靠的传输机制
 - **面向数据报：**
	 - UDP 以**独立的数据报**为单位进行数据传输。每个数据报都包含了完整的源地址、目标地址等信息。接收方收到的是一个个独立的数据报，保留了消息的边界。举个🌰，发什么消息对方只有没接到消息或接收到了全部消息。
 - **开销小，传输效率高**

### 为什么？

主要是因为业务需要，协议中需要包含扫码数据（如条码字符串）、相机ID、时间戳、流水线位置等特定字段，自定义协议可以精确地控制每个字段的长度和类型，避免不必要的开销。


## 这个项目中是否采用了自定义协议格式？为什么？怎么定义的？

是的，该项目采用了自定义协议格式。采用自定义协议格式主要是为了结合业务，让数据更加紧凑，解析速度更快，减少了网络带宽和CPU的消耗。
对数据包做了以下规定：
- **魔数 (Magic Number - 4字节)：** 固定值，用于快速识别这是一个有效的协议包，防止脏数据。
- **版本号 (Version - 1字节)：** 协议版本，用于未来升级。
- **消息类型 (Message Type - 1字节)：** 指示数据包的类型，例如：心跳包、扫码数据包、配置请求包、响应包等。
- **数据体长度 (Body Length - 4字节)：** 指示后续数据体的字节长度，这是解决粘包/拆包问题的关键。
- **数据体：(真正的业务数据)**
	- 相机ID （固定字节）
	- 时间戳（毫秒值，固定字节）
	- 数据条码
- **结束符 (Delimiter - 4字节)**


## 如何处理的粘包/拆包问题？

拆包、粘包问题产生的原因：
- 粘包/拆包是TCP协议中常见的现象，因为TCP是面向字节流的，它不保留消息边界。

主要有两种主流的解决方法：
1. 定长消息
2. 规定消息结束标识（本项目采用的方案）

具体如何实现呢？
1. 根据定义的结束符的字节长度，循环读取数据
2. 当出现了规定的结束符时，表明已经接收到了一个完整的消息

第三方 socket sdk 中都提供了完善的解决方案，像国产化的 `smart-socket` sdk 中，提供了`DelimiterFrameDecoder` （按照标识）与 `FixedLengthFrameDecoder` （按照长度）编解码器，只需要实现上述接口，重写 `decode` 方法即可自定义编解码器。
```java
	@Slf4j  
	public class YlProtocol implements Protocol<String> {  
	  
	    //结束符 DONE 解决半包 沾包问题  
	    private static final byte[] DELIMITER_BYTES = new byte[]{'D', 'O', 'N', 'E'};  
	  
	    @Override  
	    public String decode(ByteBuffer readBuffer, AioSession session) {  
	        log.info("数据解码，值为：{}", readBuffer);  
	        DelimiterFrameDecoder delimiterFrameDecoder;  
	        if (session.getAttachment() == null) {  
	            //构造指定结束符的临时缓冲区  
	            delimiterFrameDecoder = new DelimiterFrameDecoder(DELIMITER_BYTES, 256);  
	            //缓存解码器已应对半包情况  
	            session.setAttachment(delimiterFrameDecoder);  
	        } else {  
	            delimiterFrameDecoder = session.getAttachment();  
	        }  
	  
	        //未解析到DELIMITER_BYTES则返回null  
	        if (!delimiterFrameDecoder.decode(readBuffer)) {  
				//log.info("未解析到结尾标识DONE");  
	            return null;  
	        }  
	        //解码成功  
	        ByteBuffer byteBuffer = delimiterFrameDecoder.getBuffer();  
	        byte[] bytes = new byte[byteBuffer.remaining()];  
	        byteBuffer.get(bytes);  
	        session.setAttachment(null);//释放临时缓冲区  
	        String clientData = new String(bytes);  
	        return clientData;  
	    }  
	}
	
```


## 引入 RocketMQ 进行流量消峰。扫码相机产生的原始数据是直接发到 MQ 了吗？
是经过了少量但必要的前置处理后，才发送到 RocketMQ 的。直接将原始二进制数据发送到MQ是不太理想的，主要原因如下：
- **业务解耦：** MQ应该传输的是具有业务意义的数据，而不是底层的协议字节流。
- **数据格式标准化：** 原始二进制数据需要解析，如果直接发到MQ，意味着每个消费者都要负责解析，这不符合“职责单一”原则。
- **减少无效数据：** 在发送到MQ之前，我们可以进行初步的合法性校验，过滤掉明显错误或不完整的帧，减少MQ的无效负载。

具体的处理如下：
1. Socket层接收原始字节流
2. 协议解析与初步校验：将这个字节数组解析成具体的业务数据对象
3. 封装成业务消息体
4. 发送到 MQ

## 消费者的消费速度如何保证能跟上生产者的高峰？
![[RocketMQ-常见问题#如何处理消息堆积问题]] 



