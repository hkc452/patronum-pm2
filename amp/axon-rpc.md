山一层水一层，终于讲到 rpc 了。[axon-rpc](https://github.com/Unitech/pm2-axon-rpc) 是基于 axon 上进行封装的，下面我们就看看怎么使用 axon-rpc吧。

axon-rpc 分为客户端和服务端，服务端暴露方法出来，由客户端进行调用，一个简单的 rpc 链路完成。

```js
var rpc = require('axon-rpc')
  , axon = require('axon')
  , rep = axon.socket('rep');

var server = new rpc.Server(rep);
rep.bind(4000);
server.expose('add', function(a, b, fn){
  fn(null, a + b);
});

var rpc = require('axon-rpc')
  , axon = require('axon')
  , req = axon.socket('req');

var client = new rpc.Client(req);
req.connect(4000);

client.call('add', 1, 2, function(err, n){
  console.log(n);
  // => 3
})
```

#### Server

我们首先来看看服务端 Server，需要注意的是，虽然支持自定义 Socket ，但是基本都是使用 Req / Rep 组。

构造函数 methods 存放要暴露的方法，同时把 sock 上的 message 监听绑定要 onmessage。

expose 方法是把服务端要暴露的方法存放到 methods 上面去，支持对象属性。

现在我们来看看 onmessage 方法，需要注意的是，Server 是 Rep Socket，调用 reply 会回复 Req 的消息。当 msg 消息类型是 methods 时，我们返回服务器上面所有 method 的描述，包括参数。如果是调用函数，我们还需要看看对应 method 是否在 methods 存在，我们把 reply 封装在 回调中，让对应 method 去触发回调，如果回调触发时出现错误，调用 reply 告诉客户端错误，否则把调用结果告诉客户端 ` reply({ args: args })`。
```js
/**
 * Expose `Server`.
 */

module.exports = Server;

/**
 * Initialize a server with the given `sock`.
 *
 * @param {Socket} sock
 * @api public
 */

function Server(sock) {
  if (typeof sock.format === 'function') sock.format('json');
  this.sock = sock;
  this.methods = {};
  this.sock.on('message', this.onmessage.bind(this));
}

/**
 * Return method descriptions with:
 *
 *  `.name` string
 *  `.params` array
 *
 * @return {Object}
 * @api private
 */

Server.prototype.methodDescriptions = function(){
  var obj = {};
  var fn;

  for (var name in this.methods) {
    fn = this.methods[name];
    obj[name] = {
      name: name,
      params: params(fn)
    };
  }

  return obj;
};

/**
 * Response with the method descriptions.
 *
 * @param {Function} fn
 * @api private
 */

Server.prototype.respondWithMethods = function(reply){
  reply({ methods: this.methodDescriptions() });
};

/**
 * Handle `msg`.
 *
 * @param {Object} msg
 * @param {Object} fn
 * @api private
 */

Server.prototype.onmessage = function(msg, reply){
  if ('methods' == msg.type) return this.respondWithMethods(reply);

  if (!reply) {
    console.error('reply false');
    return false;
  }

  // .method
  var meth = msg.method;
  if (!meth) return reply({ error: '.method required' });

  // ensure .method is exposed
  var fn = this.methods[meth];
  if (!fn) return reply({ error: 'method "' + meth + '" does not exist' });

  // .args
  var args = msg.args;
  if (!args) return reply({ error: '.args required' });

  // invoke
  args.push(function(err){
    if (err) {
      if (err instanceof Error)
        return reply({ error: err.message, stack: err.stack });
      else
        return reply({error : err});
    }
    var args = [].slice.call(arguments, 1);
    reply({ args: args });
  });

  fn.apply(null, args);
};

/**
 * Expose many or a single method.
 *
 * @param {String|Object} name
 * @param {String|Object} fn
 * @api public
 */

Server.prototype.expose = function(name, fn){
  if (1 == arguments.length) {
    for (var key in name) {
      this.expose(key, name[key]);
    }
  } else {
    debug('expose "%s"', name);
    this.methods[name] = fn;
  }
};

/**
 * Parse params.
 *
 * @param {Function} fn
 * @return {Array}
 * @api private
 */

function params(fn) {
  // remove space to make it work on node 10.x.x too
  var ret = fn.toString().replace(/\s/g, '').match(/^function *(\w*)\((.*?)\)/)[2];
  if (ret) return ret.split(/ *, */);
  return [];
}
```
#### Client
OK，现在来看看 Client。客户端主要有两个方法，call 用于远程调用服务端方法并且得到结果，methods 得到服务端所有暴露方法的描述，大部分客户端是 Req Socket。

看看 call 方法，调用 sock 发送消息，类型是 call，把方法名和参数都传递过去，回调触发时，如果 msg 存在错误，把错误传给 fn，否则把结果传给 fn，fn就是调用 call 方法时传进来的回调。
```js
/**
 * Expose `Client`.
 */

module.exports = Client;

/**
 * Initialize an rpc client with `sock`.
 *
 * @param {Socket} sock
 * @api public
 */

function Client(sock) {
  if (typeof sock.format === 'function') sock.format('json');
  this.sock = sock;
}

/**
 * Invoke method `name` with args and invoke the
 * tailing callback function.
 *
 * @param {String} name
 * @param {Mixed} ...
 * @param {Function} fn
 * @api public
 */

Client.prototype.call = function(name){
  var args = [].slice.call(arguments, 1, -1);
  var fn = arguments[arguments.length - 1];

  this.sock.send({
    type: 'call',
    method: name,
    args: args
  }, function(msg){
    if ('error' in msg) {
      var err = new Error(msg.error);
      err.stack = msg.stack || err.stack;
      fn(err);
    } else {
      msg.args.unshift(null);
      fn.apply(null, msg.args);
    }
  });
};

/**
 * Fetch the methods exposed and invoke `fn(err, methods)`.
 *
 * @param {Function} fn
 * @api public
 */

Client.prototype.methods = function(fn){
  this.sock.send({
    type: 'methods'
  }, function(msg){
    fn(null, msg.methods);
  });
};
```

#### 总结 
至此 ，RPC 讲解完毕~ 长舒一口气