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