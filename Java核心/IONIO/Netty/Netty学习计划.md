# 目标

能熟练使用Netty开发网络程序。

1. 掌握Netty异步事件处理机制，掌握Inbound \ Outbound流，掌握NettyIO事件及处理，以及Netty的异步处理相关的**Future/Promise**类
2. 掌握Netty对buffer的使用，掌握HeapByteBuf\DirectByteBuf\CompositeByteBuf的区别和作用，了解Pooled和UnPooled的区别，以及Pooled实现原理
3. 掌握Netty对TCP协议粘包半包处理，以及相关的处理类
4. 掌握常用的编解码类，尤其是HTTP协议的编解码
5. 掌握Netty线程模型，了解Netty如何工作
6. 了解TCP协议相关知道
7. 学习Netty源码精华，提高编码质量

# 结果

1. 使用Netty开发一个简易的IM聊天工具，自己设计协议
2. 使用Netty开发一个异步HttpClient工具类
3. 使用Netty整合SpringMvc实现比较简单的HttpServer+SpringMvc功能
4. 产出Netty学习PPT，讲述Netty的设计，功能及使用

# 学习资料

电子书：Netty权威指南 



Netty权威指南 第2版 带书签目录 完整版.pdf



源码：https://github.com/netty/netty

# 学习计划

## 第一阶段 - 了解Netty （19.4.23 ~19.5.5）

- 熟悉Netty经常使用的类，以及作用
- 常见的server\client 模式编写方式
- 常见的编解码类的作用以及使用，包括组合使用
- 熟悉pipeline流的对IO inbound 和outbound的处理顺序，及自定义控制
- Handler常见方法的作用及使用

## 第二阶段 - 理解原理（19.5.5 ~19.5.17）

在对Netty有初步的理解后，深入学习

- 理解Netty异步事件处理机制
- 掌握Netty对buffer的使用
- 掌握Netty对TCP协议粘包半包处理，以及相关的处理类
- 掌握Netty线程模型，了解Netty如何工作

## 第三阶段 - 源码阅读（19.5.17 ~19.6.9）

- 学习EventLoop的相关实现，学习Netty线程池的管理，了解实现细节
- 学习Future/Promise的实现原理，理解异步事件的实现细节
- 学习Channel&Unsafe对java原生IO的包装，了解实现细节
- 学习ChannelPipeline对IO流的控制，了解实现细节
- 学习ChannelHandler对IO流的拦截和处理，了解其实现细节
- 了解Bytebuf的相关实现





https://km.sankuai.com/page/151522790