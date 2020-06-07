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