在讲 rpc 之前，先讲讲通信协议的设计。什么是通讯协议，简单地说，是指通信双方对数据传送控制的一种约定。通信不仅仅是把消息传过去，还要保证消息的准确和可靠。

可以先看看 [amp](https://github.com/tj/node-amp) 协议的设计, 对于首个字节，我们把版本存到它的低四位，多少条数据存到它的高四位，刚好消耗完一个字节。对于接下来的每个数据，都是以 `<length>` / `<data>` 这种形式存在，每个数据的长度以4个字节表示，而数据的长度是不固定的。
> All multi-byte integers are big endian. The `version` and `argc` integers
  are stored in the first byte, followed by a sequence of zero or more
  `<length>` / `<data>` pairs, where `length` is a 32-bit unsigned integer.

```
      0        1 2 3 4     <length>    ...
+------------+----------+------------+
| <ver/argc> | <length> | <data>     | additional arguments
+------------+----------+------------+
```

需要注意的一点是，字节的高地位和高低地址是两回事，这样就设计到采用大端序还是小端序存储数据的问题。当然，对于网络协议的设计，我们需要采用大端序。

![大端序和小端序](https://hkc.gitee.io/pics/pm2/bite.png)

> 它们的区别在于：Int8 只需要一个字节就可以表示，而 Short，Int32，Double 这些类型一个字节放不下，我们就要用多个字节表示，这就要引入「字节序」的概念，也就是字节存储的顺序。对于某一个要表示的值，是把它的低位存到低地址，还是把它的高位存到低地址，前者叫小端字节序（Little Endian），后者叫大端字节序（Big Endian）。大端和小端各有优缺点，不同的CPU厂商并没有达成一致，但是当网络通讯的时候大家必须统一标准，不然无法通讯了。为了统一网络传输时候的字节的顺序，TCP/IP 协议 RFC1700 里规定使用「大端」字节序作为网络字节序，所以，我们在开发网络通讯协议的时候操作 Buffer 都应该用大端序的 API，也就是 BE 结尾的。参考来自：[聊聊 Node.js RPC（一）— 协议](https://zhuanlan.zhihu.com/p/38012481)

