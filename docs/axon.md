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

`Socket.prototype.use` 方法主要是用于插件调用，把当前实例出传进去，同时返回 this 表示支持链式调用。对于 axon 来说，主要支持两个插件，一个 enqueue，用于当发送数据的，要接收端没有在线，这是把消息缓存起来，另外一个 round-robin，round-robin 算法都知道是啥回事，在这里是服务器采用 round-robin 像接收端发送信息~
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