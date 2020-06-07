[axon](https://github.com/Unitech/pm2-axon) 是 rpc 底层依赖的 socket 库，是讲 rpc 前最后的热身。

我们可以先看看他支持的特性~
> 1. message oriented 
> 2. automated reconnection
> 3. light-weight wire protocol
> 4. mixed-type arguments (strings, objects, buffers, etc)
> 5. unix domain socket support
> 6. fast (~800 mb/s ~500,000 messages/s)

支持多种数据类型、速度足够快、轻量的协议、支持 Unix socket 协议、支持 amp-message、 支持多种数据类型混合等~

axon 一共支持四种组类型的 socket，具体四对怎么使用，且听下回分解~ :溜)
- push / pull
- pub / sub
- req / rep
- pub-emitter / sub-emitter

axon 通过 socket 方法去初始化内部四组八种类型的 socket，type 就是具体的socket 类型，如果找不到，会直接抛出找不到 socket 类型的报错，否则使用传入的 options 值去初始化 socket。
```js
/**
 * Constructors.
 */

exports.PubEmitterSocket = require('./sockets/pub-emitter');
exports.SubEmitterSocket = require('./sockets/sub-emitter');
exports.PushSocket = require('./sockets/push');
exports.PullSocket = require('./sockets/pull');
exports.PubSocket = require('./sockets/pub');
exports.SubSocket = require('./sockets/sub');
exports.ReqSocket = require('./sockets/req');
exports.RepSocket = require('./sockets/rep');
exports.Socket = require('./sockets/sock');

/**
 * Socket types.
 */

exports.types = {
  'pub-emitter': exports.PubEmitterSocket,
  'sub-emitter': exports.SubEmitterSocket,
  'push': exports.PushSocket,
  'pull': exports.PullSocket,
  'pub': exports.PubSocket,
  'sub': exports.SubSocket,
  'req': exports.ReqSocket,
  'rep': exports.RepSocket
};

/**
 * Return a new socket of the given `type`.
 *
 * @param {String} type
 * @param {Object} options
 * @return {Socket}
 * @api public
 */

exports.socket = function(type, options){
  var fn = exports.types[type];
  if (!fn) throw new Error('invalid socket type "' + type + '"');
  return new fn(options);
};
```
接下来，让我们首先看看 [sock](https://github.com/Unitech/pm2-axon/blob/master/lib/sockets/sock.js) ，因为它是内置其他八种 socket 的基类。

从 Socket 上面的构造函数注释我们可以看到， Socket 可以是 client 或者 server ，取决于我们用了 connect 还是 bind 方法~

看看构造方法, this.opts 就是传入的可选参数， this.server 只有当前 socket 是服务器时才有值，this.socks 保存着我们链接上服务器的 socket 或者 服务器上链接的 socket，对于 this.setting ，需要提下 `Configurable(Socket.prototype)`, Configurable 为我们操作 setting 属性提供了方法，举个例子, `this.set('hwm', Infinity)`，这个就是讲 `setting.hwm` 赋值为 Infinity。而对于构造函数的四个 set 属性，可以看看 [Readme](https://github.com/Unitech/pm2-axon/blob/master/Readme.md)

继续往下看，可以看到 Socket 的原型的原型指向了 `Emitter.prototype`,也就是 Socket 支持事件监听~

`Socket.prototype.use` 方法主要是用于插件调用，把当前实例出传进去，同时返回 this 表示支持链式调用。对于 axon 来说，主要支持两个插件，一个 enqueue，用于当发送数据的，要接收端没有在线，这是把消息缓存起来，另外一个 round-robin，round-robin 算法都知道是啥回事，在这里是服务器采用 round-robin 向接收端发送信息~
``` js
/**
 * Expose `Socket`.
 */

module.exports = Socket;

/**
 * Initialize a new `Socket`.
 *
 * A "Socket" encapsulates the ability of being
 * the "client" or the "server" depending on
 * whether `connect()` or `bind()` was called.
 *
 * @api private
 */

function Socket() {
  var self = this;
  this.opts = {};
  this.server = null;
  this.socks = [];
  this.settings = {};
  this.set('hwm', Infinity);
  this.set('identity', String(process.pid));
  this.set('retry timeout', 100);
  this.set('retry max timeout', 5000);
}

/**
 * Inherit from `Emitter.prototype`.
 */

Socket.prototype.__proto__ = Emitter.prototype;

/**
 * Make it configurable `.set()` etc.
 */

Configurable(Socket.prototype);

/**
 * Use the given `plugin`.
 *
 * @param {Function} plugin
 * @api private
 */

Socket.prototype.use = function(plugin){
  plugin(this);
  return this;
};
```
pack 方法调用 amp-message 去打包数据，最后返回 msg.toBuffer()，还记得Message 内部在内部调用 amp.encode 之前，会调用自己的 pack 方法对数据加标识
```js
/**
 * Creates a new `Message` and write the `args`.
 *
 * @param {Array} args
 * @return {Buffer}
 * @api private
 */

Socket.prototype.pack = function(args){
  var msg = new Message(args);
  return msg.toBuffer();
};
```
closeSockets 看注释就知道干嘛的，关闭 this.socks 上面所有的 socket，调用  sock.destroy 方法，主要 this.socks 上面的 socket 都是原生的 socket，而不是这个 Socket 对象的实例。

再来看看 close 方法，首先讲 closing 状态置为 true，然后调用 closeSockets 关闭 socket，对于服务器，还要调用 closeServer 关闭服务器。

我们来看看 closeServer，close 会传入 fn 回调进去 closeServer, closeServer 内部 this.server 首先把 close 回调与 socket 上面的close 监听方法绑定在一起，然后显式调用 server.close,最后触发我们传入的回调 fn。
```js
/**
 * Close all open underlying sockets.
 *
 * @api private
 */

Socket.prototype.closeSockets = function(){
  debug('closing %d connections', this.socks.length);
  this.socks.forEach(function(sock){
    sock.destroy();
  });
};

/**
 * Close the socket.
 *
 * Delegates to the server or clients
 * based on the socket `type`.
 *
 * @param {Function} [fn]
 * @api public
 */

Socket.prototype.close = function(fn){
  debug('closing');
  this.closing = true;
  this.closeSockets();
  if (this.server) this.closeServer(fn);
};

/**
 * Close the server.
 *
 * @param {Function} [fn]
 * @api public
 */

Socket.prototype.closeServer = function(fn){
  debug('closing server');
  this.server.on('close', this.emit.bind(this, 'close'));
  this.server.close();
  fn && fn();
};
```

address 用于获取服务器的地址，如果不是服务器就直接返回，默认是 tcp 协议，当然我们前面说了也支持 unix 协议~
```js
/**
 * Return the server address.
 *
 * @return {Object}
 * @api public
 */

Socket.prototype.address = function(){
  if (!this.server) return;
  var addr = this.server.address();
  addr.string = 'tcp://' + addr.address + ':' + addr.port;
  return addr;
};
```

removeSocket、addSocket 就是讲连接上的 socket 存到一个数组里面，对于客户端来说，就是 connect 中回调函数中的 socket，对于服务端，就是 onconnect 回到参数中的 socket。对于 addSocket 我们需要注意的是，因为我们采用的是 amp 协议，所以需要从流中取出完整的 amp 数据，需要借助 amp.Stream，最后 parser 触发 data 事件时，会触发 onMessage 事件。
```js
/**
 * Remove `sock`.
 *
 * @param {Socket} sock
 * @api private
 */

Socket.prototype.removeSocket = function(sock){
  var i = this.socks.indexOf(sock);
  if (!~i) return;
  debug('remove socket %d', i);
  this.socks.splice(i, 1);
};

/**
 * Add `sock`.
 *
 * @param {Socket} sock
 * @api private
 */

Socket.prototype.addSocket = function(sock){
  var parser = new Parser;
  var i = this.socks.push(sock) - 1;
  debug('add socket %d', i);
  sock.pipe(parser);
  parser.on('data', this.onmessage(sock));
};
```
趁热打铁，我们来看看 onmessage 事件，需要注意的是，我们八种类型的socket，有些会根据自己的需求，覆盖内置的 onmessage。

我们知道 Message 初始化的时候，如果传参是 buffer，会将 buffer 解码，然后将解码后的数据和原生 socket 传给 socket 实例上面的 message 监听函数~
```js
/**
 * Handles framed messages emitted from the parser, by
 * default it will go ahead and emit the "message" events on
 * the socket. However, if the "higher level" socket needs
 * to hook into the messages before they are emitted, it
 * should override this method and take care of everything
 * it self, including emitted the "message" event.
 *
 * @param {net.Socket} sock
 * @return {Function} closure(msg, mulitpart)
 * @api private
 */

Socket.prototype.onmessage = function(sock){
  var self = this;
  return function(buf){
    var msg = new Message(buf);
    self.emit.apply(self, ['message'].concat(msg.args), sock);
  };
};
```
handleErrors 就是用于处理错误的，对于发生错误的 原生sokcet，首先触发 socket error 监听，同时将发生错误的 socket 移除出去，再看看发生的错误是否在忽略错误列表，如果不在，触发 error 监听，否则触发 ignored error 监听。
```js
/**
 * Errors to ignore.
 */

var ignore = [
  'ECONNREFUSED',
  'ECONNRESET',
  'ETIMEDOUT',
  'EHOSTUNREACH',
  'ENETUNREACH',
  'ENETDOWN',
  'EPIPE',
  'ENOENT'
];
/**
 * Handle `sock` errors.
 *
 * Emits:
 *
 *  - `error` (err) when the error is not ignored
 *  - `ignored error` (err) when the error is ignored
 *  - `socket error` (err) regardless of ignoring
 *
 * @param {Socket} sock
 * @api private
 */

Socket.prototype.handleErrors = function(sock){
  var self = this;
  sock.on('error', function(err){
    debug('error %s', err.code || err.message);
    self.emit('socket error', err);
    self.removeSocket(sock);
    if (!~ignore.indexOf(err.code)) return self.emit('error', err);
    debug('ignored %s', err.code);
    self.emit('ignored error', err);
  });
};
```
接下来，我们看看 connect 方法，这个就是当 Socket 成为客户端时，需要连接服务端的方法~

connect 首先会进行一系列的参数校验，但是如果我们采用 bind 方法过，type 会被置为 server ，这是我们不能再调用 connect 方法了。如果 port 是 字符串，会调用 url.parse, 尝试解析出 post 和 port。

sock.setNoDelay() 表示禁用 Nagle 的 socket 算法，因为我们希望数据延迟。

对于 sock 的 close 监听，这时 connected 为false，除了我们主动调用上面的 socket.close 方法关闭的，这是会触发 close 监听，否则会尝试进行重连。

对于 sock 的  conncet 监听，如果连接成功，这时 connected 为 true，同时将 sock 加入 this.socks, 触发 socket 上面的 conncet 监听。

`sock.connect(port, host); ` 就是主动发起连接~

最后返回 this ，方便链式调用。

```js
/**
 * Connect to `port` at `host` and invoke `fn()`.
 *
 * Defaults `host` to localhost.
 *
 * TODO: needs big cleanup
 *
 * @param {Number|String} port
 * @param {String} host
 * @param {Function} fn
 * @return {Socket}
 * @api public
 */

Socket.prototype.connect = function(port, host, fn){
  var self = this;
  if ('server' == this.type) throw new Error('cannot connect() after bind()');
  if ('function' == typeof host) fn = host, host = undefined;

  if ('string' == typeof port) {
    port = url.parse(port);

    if (port.pathname) {
      fn = host;
      host = null;
      fn = undefined;
      port = port.pathname;
    } else {
      host = port.hostname || '0.0.0.0';
      port = parseInt(port.port, 10);
    }
  } else {
    host = host || '0.0.0.0';
  }

  var max = self.get('retry max timeout');
  var sock = new net.Socket;
  sock.setNoDelay();
  this.type = 'client';
  port = port;

  this.handleErrors(sock);

  sock.on('close', function(){
    self.connected = false;
    self.removeSocket(sock);
    if (self.closing) return self.emit('close');
    var retry = self.retry || self.get('retry timeout');
    if (retry === 0) return self.emit('close');
    setTimeout(function(){
      debug('attempting reconnect');
      self.emit('reconnect attempt');
      sock.destroy();
      self.connect(port, host);
      self.retry = Math.round(Math.min(max, retry * 1.5));
    }, retry);
  });

  sock.on('connect', function(){
    debug('connect');
    self.connected = true;
    self.addSocket(sock);
    self.retry = self.get('retry timeout');
    self.emit('connect');
    fn && fn();
  });

  debug('connect attempt %s:%s', host, port);
  sock.connect(port, host);
  return this;
};
```
bind 方法就是让 Socket 成为服务器，同理，如果调用过 conncet 方法，则不能再次调用 bind 方法，不然 最明显的就是 this.socks 里面的 sock 都混了~

解析我们解析 port，需要主要的是注意 unix socket 文件，我们需要特别做个标记，unixSocket 为 true，这在后面会用到，对于 unix socket， `unix:///some/path` 的 port 就是 `/some/path`。

接着我们将 type 设置为 server，通过尝试 `net.createServer(this.onconnect.bind(this))` 建立服务器，并赋值给 server，需要注意的是，this.onconnect 方法，这个我们会接下来讲解。

`this.server.listen(port, host, fn)` 开始建立服务器监听，正式起服务器。

这里需要再提下 unixSocket，以为我们需要对 socket 文件监听错误时进行一个重试机制的处理。如果出现 EADDRINUSE 错误，我们会尝试建立客户端链接上去，如果链接成功了，说明已经确实有进程在用。否则删除 socket 文件，重试建立服务器监听。对于 unixSocket 的其他错误，直接删除 socket 文件，重新监听服务器监听。
```js
/**
 * Bind to `port` at `host` and invoke `fn()`.
 *
 * Defaults `host` to INADDR_ANY.
 *
 * Emits:
 *
 *  - `connection` when a client connects
 *  - `disconnect` when a client disconnects
 *  - `bind` when bound and listening
 *
 * @param {Number|String} port
 * @param {Function} fn
 * @return {Socket}
 * @api public
 */

Socket.prototype.bind = function(port, host, fn){
  var self = this;
  if ('client' == this.type) throw new Error('cannot bind() after connect()');
  if ('function' == typeof host) fn = host, host = undefined;

  var unixSocket = false;

  if ('string' == typeof port) {
    port = url.parse(port);

    if (port.pathname) {
      fn = host;
      host = null;
      port = port.pathname;
      unixSocket = true;
    } else {
      host = port.hostname || '0.0.0.0';
      port = parseInt(port.port, 10);
    }
  } else {
    host = host || '0.0.0.0';
  }

  this.type = 'server';

  this.server = net.createServer(this.onconnect.bind(this));

  debug('bind %s:%s', host, port);
  this.server.on('listening', this.emit.bind(this, 'bind'));

  if (unixSocket) {
    // TODO: move out
    this.server.on('error', function(e) {
      debug('Got error while trying to bind', e.stack || e);
      if (e.code == 'EADDRINUSE') {
        // Unix file socket and error EADDRINUSE is the case if
        // the file socket exists. We check if other processes
        // listen on file socket, otherwise it is a stale socket
        // that we could reopen
        // We try to connect to socket via plain network socket
        var clientSocket = new net.Socket();

        clientSocket.on('error', function(e2) {
          debug('Got sub-error', e2);
          if (e2.code == 'ECONNREFUSED' || e2.code == 'ENOENT') {
            // No other server listening, so we can delete stale
            // socket file and reopen server socket
            try {
              fs.unlinkSync(port);
            } catch(e) {}
            self.server.listen(port, host, fn);
          }
        });

        clientSocket.connect({path: port}, function() {
          // Connection is possible, so other server is listening
          // on this file socket
          if (fn) return fn(new Error('Process already listening on socket ' + port));
        });
      }
      else {
        try {
          fs.unlinkSync(port);
        } catch(e) {}
        self.server.listen(port, host, fn);
      }
    });
  }

  this.server.listen(port, host, fn);
  return this;
};
```
onconnect 主要是将 sock 加入 socks ，同时触发 connect 监听，处理 sock 出现的 error，同时监听到sock 上面的 close 事件时，触发 socket 的 disconnect 监听和将 sock 移除出 socks。
```js
/**
 * Handle connection.
 *
 * @param {Socket} sock
 * @api private
 */

Socket.prototype.onconnect = function(sock){
  var self = this;
  var addr = null;

  if (sock.remoteAddress && sock.remotePort)
    addr = sock.remoteAddress + ':' + sock.remotePort;
  else if (sock.server && sock.server._pipeName)
    addr = sock.server._pipeName;

  debug('accept %s', addr);
  this.addSocket(sock);
  this.handleErrors(sock);
  this.emit('connect', sock);
  sock.on('close', function(){
    debug('disconnect %s', addr);
    self.emit('disconnect', sock);
    self.removeSocket(sock);
  });
};
```

### 插件
我们说过， Socket 一共两个插件，enqueue 和 round-robin, 接下来让我们逐一击破

#### enqueue
enqueue 主要讲消息缓存起来，等下次 socket 连接的时候，让离线存储的信息发给客户端。

这里的 sock 表示 Socket 实例，首先将离线消息缓存在 queue 数组，但消息的缓存有个峰值 `sock.settings.hwm`, 我们看看 enqueue 方法，对于无法缓存的消息，直接丢弃，调用 drop 方法，里面会触发 drop 监听方法。

上面讲服务器 bind 的时候，我们知道服务器 onconnect 方法会触发 connect 方法，这是我们会看看 sock 上面有没有缓存的离线信息，如果有，我们循环 send 给客户端，最后触发 flush 监听。
```js
/**
 * Module dependencies.
 */

var debug = require('debug')('axon:queue');

/**
 * Queue plugin.
 *
 * Provides an `.enqueue()` method to the `sock`. Messages
 * passed to `enqueue` will be buffered until the next
 * `connect` event is emitted.
 *
 * Emits:
 *
 *  - `drop` (msg) when a message is dropped
 *  - `flush` (msgs) when the queue is flushed
 *
 * @param {Object} options
 * @api private
 */

module.exports = function(options){
  options = options || {};

  return function(sock){

    /**
     * Message buffer.
     */

    sock.queue = [];

    /**
     * Flush `buf` on `connect`.
     */

    sock.on('connect', function(){
      var prev = sock.queue;
      var len = prev.length;
      sock.queue = [];
      debug('flush %d messages', len);

      for (var i = 0; i < len; ++i) {
        this.send.apply(this, prev[i]);
      }

      sock.emit('flush', prev);
    });

    /**
     * Pushes `msg` into `buf`.
     */

    sock.enqueue = function(msg){
      var hwm = sock.settings.hwm;
      if (sock.queue.length >= hwm) return drop(msg);
      sock.queue.push(msg);
    };

    /**
     * Drop the given `msg`.
     */

    function drop(msg) {
      debug('drop');
      sock.emit('drop', msg);
    }
  };
};
```

#### Round-robin
主要是采用 Round-robin 算法，将消息发送给客户端，什么是 Round-robin，说白了，就是轮询~当服务端要向客户端发送消息的时候，轮询从 socks 取出一个 sock ，如果这时 sock 可以发送消息，则调用 write 方法将 pack  后消息发送给客户端，否则调用 fallback 方法
```js
/**
 * Deps.
 */

var slice = require('../utils').slice;

/**
 * Round-robin plugin.
 *
 * Provides a `send` method which will
 * write the `msg` to all connected peers.
 *
 * @param {Object} options
 * @api private
 */

module.exports = function(options){
  options = options || {};
  var fallback = options.fallback || function(){};

  return function(sock){

    /**
     * Bind callback to `sock`.
     */

    fallback = fallback.bind(sock);

    /**
     * Initialize counter.
     */

    var n = 0;

    /**
     * Sends `msg` to all connected peers round-robin.
     */

    sock.send = function(){
      var socks = this.socks;
      var len = socks.length;
      var sock = socks[n++ % len];

      var msg = slice(arguments);

      if (sock && sock.writable) {
        sock.write(this.pack(msg));
      } else {
        fallback(msg);
      }
    };

  };
};
```