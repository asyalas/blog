### 前言
### application.js

1、构造器
```js
constructor() {
    super();

    this.proxy = false;
    this.middleware = [];
    this.subdomainOffset = 2;
    this.env = process.env.NODE_ENV || 'development';
    //生成新对象继承context，request，response
    this.context = Object.create(context);
    this.request = Object.create(request);
    this.response = Object.create(response);
  }
```
2、use方法
增加一个中间件->this.middleware.push(fn);
```js
use(fn) {
    if (typeof fn !== 'function') throw new TypeError('middleware must be a function!');
    if (isGeneratorFunction(fn)) {
      deprecate('Support for generators will be removed in v3. ' +
                'See the documentation for examples of how to convert old middleware ' +
                'https://github.com/koajs/koa/blob/master/docs/migration.md');
      fn = convert(fn);
    }
    debug('use %s', fn._name || fn.name || '-');
    this.middleware.push(fn);
    return this;
  }
```
3、listen方法
启一个http服务，把中间件放入callback李 main，再监听它
并对外提供参数，作用于server.listen
```js
 /**
   * Shorthand for:
   *
   *    http.createServer(app.callback()).listen(...)
   *
   * @param {Mixed} ...
   * @return {Server}
   * @api public
   */

  listen(...args) {
    debug('listen');
    const server = http.createServer(this.callback());
    return server.listen(...args);
  }

```

4、callback方法

```js

  /**
   * Return a request handler callback
   * for node's native http server.
   *
   * @return {Function}
   * @api public
   */

  callback() {
    //通过koa-compose,将所有的中间件做成洋葱模型
    //通过设置index下标，来防止在中间件中调用多次
  
    const fn = compose(this.middleware);

    if (!this.listenerCount('error')) this.on('error', this.onerror);

    const handleRequest = (req, res) => {
      // 基于req，res生成一个新的上下文
      const ctx = this.createContext(req, res);
      return this.handleRequest(ctx, fn);
    };

    return handleRequest;
  }

```

5、createContext方法
基于req，res生成一个新的上下文
```js

  /**
   * Initialize a new context.
   *
   * @api private
   */

  createContext(req, res) {
    const context = Object.create(this.context);
    const request = context.request = Object.create(this.request);
    const response = context.response = Object.create(this.response);
    context.app = request.app = response.app = this;
    context.req = request.req = response.req = req;
    context.res = request.res = response.res = res;
    request.ctx = response.ctx = context;
    request.response = response;
    response.request = request;
    context.originalUrl = request.originalUrl = req.url;
    context.state = {};
    return context;
  }
```

6、设置请求的事件
```js
传入ctx上下文到中间件中，执行中间件，中间件执行完毕后，对respond进行处理，然后返回
/**
   * Handle request in callback.
   *
   * @api private
   */

  handleRequest(ctx, fnMiddleware) {
    const res = ctx.res;
    res.statusCode = 404;
    const onerror = err => ctx.onerror(err);
    const handleResponse = () => respond(ctx);
    onFinished(res, onerror);
    //此时，中间件执行的顺序就是push的顺序，与redux的洋葱模型不一样
    //所有的返回操作都必须在洋葱模型执行完以后再进行，而express可以自由控制在哪执行
    return fnMiddleware(ctx).then(handleResponse).catch(onerror);
  }

```

7、对中间件处理完后的res进行处理，最后返回请求结果

```js
/**
 * Response helper
 * 该回调用来根据中间件执行过程中所做的事情来决定返回给客户端什么数据。
 */

function respond(ctx) {
  // allow bypassing koa
  if (false === ctx.respond) return;

  const res = ctx.res;
  //如果上下文的writable为false，则不做任何处理直接返回
  if (!ctx.writable) return;

  let body = ctx.body;
  const code = ctx.status;

  // ignore body
  if (statuses.empty[code]) {
    // strip headers
    ctx.body = null;
    return res.end();
  }

  if ('HEAD' == ctx.method) {
    if (!res.headersSent && isJSON(body)) {
      ctx.length = Buffer.byteLength(JSON.stringify(body));
    }
    return res.end();
  }

  // status body
  if (null == body) {
    if (ctx.req.httpVersionMajor >= 2) {
      body = String(code);
    } else {
      body = ctx.message || String(code);
    }
    if (!res.headersSent) {
      ctx.type = 'text';
      ctx.length = Buffer.byteLength(body);
    }
    return res.end(body);
  }

  // responses
  if (Buffer.isBuffer(body)) return res.end(body);
  if ('string' == typeof body) return res.end(body);
  if (body instanceof Stream) return body.pipe(res);

  // body: json
  body = JSON.stringify(body);
  if (!res.headersSent) {
    ctx.length = Buffer.byteLength(body);
  }
  res.end(body);
}


```

8、request、response的get、set方法
```js
//request的get方法
get(field) {
    const req = this.req;
    switch (field = field.toLowerCase()) {
      case 'referer':
      case 'referrer':
        return req.headers.referrer || req.headers.referer || '';
      default:
        return req.headers[field] || '';
    }
  },
//response的get方法
get(field) {
    return this.header[field.toLowerCase()] || '';
  },

  /**
   * Set header `field` to `val`, or pass
   * an object of header fields.
   *
   * Examples:
   *
   *    this.set('Foo', ['bar', 'baz']);
   *    this.set('Accept', 'application/json');
   *    this.set({ Accept: 'text/plain', 'X-API-Key': 'tobi' });
   *
   * @param {String|Object|Array} field
   * @param {String} val
   * @api public
   */
//response的set方法
  set(field, val) {
    if (this.headerSent) return;

    if (2 == arguments.length) {
      if (Array.isArray(val)) val = val.map(v => typeof v === 'string' ? v : String(v));
      else if (typeof val !== 'string') val = String(val);
      this.res.setHeader(field, val);
    } else {
      for (const key in field) {
        this.set(key, field[key]);
      }
    }
  },
```

9、res的redirect方法
需要注意的是，调用完redirect之后并没有直接触发response.end()，它仅仅是添加了一个statusCode及Location而已。
```js
redirect(url, alt) {
    // location
    if ('back' == url) url = this.ctx.get('Referrer') || alt || '/';
    this.set('Location', url);

    // status
    if (!statuses.redirect[this.status]) this.status = 302;

    // html
    if (this.ctx.accepts('html')) {
      url = escape(url);
      this.type = 'text/html; charset=utf-8';
      this.body = `Redirecting to <a href="${url}">${url}</a>.`;
      return;
    }

    // text
    this.type = 'text/plain; charset=utf-8';
    this.body = `Redirecting to ${url}.`;
  },
```
10、koa-compose的源码

- next用来进入下一个中间件
- next在当前中间件执行完成后会触发回调通知上一个中间件，而完成的前提是内部的中间件已经执行完成(resolved)
```js
function (context, next) {
  // 通过设置index，来防止next在中间件中调用多次
  let index = -1
  return dispatch(0)
  function dispatch (i) {
    if (i <= index) return Promise.reject(new Error('next() called multiple times'))
    index = i
    let fn = middleware[i]
    if (i === middleware.length) fn = next
    if (!fn) return Promise.resolve()
    try {
      return Promise.resolve(fn(context, dispatch.bind(null, i + 1)))
    } catch (err) {
      return Promise.reject(err)
    }
  }
}
```