上一节讲的 amp 属于底层的协议接口，这次的 [amp-message](https://github.com/tj/node-amp-message) 是在 amp 上面封装了一层去编码、解码和操作我们的信息。

下面就让我们来看看 amp-message。首先看看 Message 的构造函数，由于支持传入 message.toBuffer 做参数，而 message.toBuffer 返回的就是 encode 完的数据，所以如果我们识别是 buffer，则调用 decode 去解码~ 否则赋值给 this.args

this.args 就是我们的数据列表，所以我们可以调用数组的 `['push','pop'，'shift','unshift']` 这四个方法去操作 args。

接下来我们去看看 encode 和 decode 方法。

encode 方法在调用 amp.encode 编码我们的消息之前，首先会对我们的消息进行一层的包装 pack。我们去看看 pack 方法，对于 buffer 类型不做包装，对于 string 类型，前面添加 `s:`，然后转化成 buffer，而对于 json 类型，前面添加 `j:`, 同时 JSON.stringify 处理数据，最后才转化成 buffer。为什么对于 string 和 json 类型前面两个要添加标识呢？这是为了 decode 的时候，通过校验前两位，从而知道我们消息的类型，进行原本数据的还原。

需要注意的是，我们 Message 的构造函数 要么是数据，要么是 message.toBuffer，不能是直接传入一个普通的 buffer，这样会导致解码失败~
```js

/**
 * Module dependencies.
 */

var fmt = require('util').format;
var amp = require('amp');

/**
 * Proxy methods.
 */

var methods = [
  'push',
  'pop',
  'shift',
  'unshift'
];

/**
 * Expose `Message`.
 */

module.exports = Message;

/**
 * Initialize an AMP message with the
 * given `args` or message buffer.
 *
 * @param {Array|Buffer} args or blob
 * @api public
 */

function Message(args) {
  if (Buffer.isBuffer(args)) args = decode(args);
  this.args = args || [];
}

// proxy methods

methods.forEach(function(method){
  Message.prototype[method] = function(){
    return this.args[method].apply(this.args, arguments);
  };
});

/**
 * Inspect the message.
 *
 * @return {String}
 * @api public
 */

Message.prototype.inspect = function(){
  return fmt('<Message args=%d size=%d>',
    this.args.length,
    this.toBuffer().length);
};

/**
 * Return an encoded AMP message.
 *
 * @return {Buffer}
 * @api public
 */

Message.prototype.toBuffer = function(){
  return encode(this.args);
};

/**
 * Decode `msg` and unpack all args.
 *
 * @param {Buffer} msg
 * @return {Array}
 * @api private
 */

function decode(msg) {
  var args = amp.decode(msg);

  for (var i = 0; i < args.length; i++) {
    args[i] = unpack(args[i]);
  }

  return args;
}

/**
 * Encode and pack all `args`.
 *
 * @param {Array} args
 * @return {Buffer}
 * @api private
 */

function encode(args) {
  var tmp = new Array(args.length);

  for (var i = 0; i < args.length; i++) {
    tmp[i] = pack(args[i]);
  }

  return amp.encode(tmp);
}

/**
 * Pack `arg`.
 *
 * @param {Mixed} arg
 * @return {Buffer}
 * @api private
 */

function pack(arg) {
  // blob
  if (Buffer.isBuffer(arg)) return arg;

  // string
  if ('string' == typeof arg) return Buffer.from('s:' + arg);

  // undefined
  if (arg === undefined) arg = null;

  // json
  return Buffer.from('j:' + JSON.stringify(arg));
}

/**
 * Unpack `arg`.
 *
 * @param {Buffer} arg
 * @return {Mixed}
 * @api private
 */

function unpack(arg) {
  // json
  if (isJSON(arg)) return JSON.parse(arg.slice(2));

  // string
  if (isString(arg)) return arg.slice(2).toString();

  // blob
  return arg;
}

/**
 * String argument.
 */

function isString(arg) {
  return 115 == arg[0] && 58 == arg[1];
}

/**
 * JSON argument.
 */

function isJSON(arg) {
  return 106 == arg[0] && 58 == arg[1];
}

```