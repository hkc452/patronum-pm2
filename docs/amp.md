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

amp 的使用方法也很简单，encode 对数据进行编码，decode 对数据进行解码。

```js
var bin = amp.encode([Buffer.from('hello'), Buffer.from('world')]);
var msg = amp.decode(bin);
console.log(msg);
```

接下来我们就来看看 encode 是怎么实现的~ 首先我们需要确认，我们一共需要多少字节，上面我们已经讲到了协议的组成，所以我们需要一个字节存储协议的版本号和数据条数的，加上每条数据需要4个字节存储数据长度以及数据本身长度需要的字节数。而对于首个字节，我们通过 `version << 4 | argc`, 来实现版本在低四位，数据条数在高四位。

对于每条数据的存储，我们前面说过，首先把数据的长度32位无符号大端序写入，接着再把数据写入。循环结束之时，就是我们的封印，不，我们的数据写完之时~
```js
/**
 * Protocol version.
 */

var version = 1;

/**
 * Encode `msg` and `args`.
 *
 * @param {Array} args
 * @return {Buffer}
 * @api public
 */

module.exports = function(args){
  var argc = args.length;
  var len = 1;
  var off = 0;

  // data length
  for (var i = 0; i < argc; i++) {
    len += 4 + args[i].length;
  }

  // buffer
  var buf = Buffer.allocUnsafe(len);

  // pack meta
  buf[off++] = version << 4 | argc;

  // pack args
  for (var i = 0; i < argc; i++) {
    var arg = args[i];

    buf.writeUInt32BE(arg.length, off);
    off += 4;

    arg.copy(buf, off);
    off += arg.length;
  }

  return buf;
};
```

我们再来看看怎么把数据解码 decode ，不管是编码还是解码，需要牢记于心的是，我们前面通讯协议的结构是怎样的~

首先我们从 buf 从拿出首个字节，`meta >> 4` 拿出低四位的版本号，`meta & 0xf` 通过让高四位相与，拿到数据条数 argv。

接着开始循环，首先取出 `readUInt32BE` 数据的长度，接着通过 `buf.slice(off, off += len)` 拿到数据，最后把数据数组返回去，打完收工
~
```js
/**
 * Decode the given `buf`.
 *
 * @param {Buffer} buf
 * @return {Object}
 * @api public
 */

module.exports = function(buf){
  var off = 0;

  // unpack meta
  var meta = buf[off++];
  var version = meta >> 4;
  var argv = meta & 0xf;
  var args = new Array(argv);

  // unpack args
  for (var i = 0; i < argv; i++) {
    var len = buf.readUInt32BE(off);
    off += 4;

    var arg = buf.slice(off, off += len);
    args[i] = arg;
  }

  return args;
};
```