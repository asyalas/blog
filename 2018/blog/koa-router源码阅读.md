### 前言
### 源码阅读
|  File   |     Description      |      
|-------------|:-------------|
|   layer    |  信息存储：路径、METHOD、路径对应的正则匹配、路径中的参数、路径对应的中间件 | 
|    router  |  主要逻辑：对外暴露注册路由的函数、提供处理路由的中间件，检查请求的URL并调用对应的layer中的路由处理|
### 参数
|  Param   |     Type      |      Default|Description|
|-------------|:-------------|:-------------|:-------------|
|   sensitive    | Boolean| false| 是否严格匹配大小写| 
|   strict    | Boolean| false| 如果设置为false则匹配路径后边的/是可选的| 
|   methods    | Array[String]| ['HEAD','OPTIONS','GET','PUT','PATCH','POST','DELETE']| 设置路由可以支持的METHOD|
|   routerPath    | String| null| | 
|   prefix    | String| null|添加一个Router注册时的前缀 | 
### 实例化
```js
function Router(opts) {
  //单例模式
  if (!(this instanceof Router)) {
    return new Router(opts)
  }

  this.opts = opts || {}
  this.methods = this.opts.methods || [
    'HEAD',
    'OPTIONS',
    'GET',
    'PUT',
    'PATCH',
    'POST',
    'DELETE'
  ]

  this.params = {}
  this.stack = []
}

```

### allowedMethods方法

```js

Router.prototype.allowedMethods = function (options) {
  options = options || {};
  var implemented = this.methods;

  return function allowedMethods(ctx, next) {
    //直接调用next()，所有逻辑在then，逻辑后置，必须放在最后
    return next().then(function() {
      var allowed = {};

      if (!ctx.status || ctx.status === 404) {
        ctx.matched.forEach(function (route) {
          route.methods.forEach(function (method) {
            allowed[method] = method;
          });
        });

        var allowedArr = Object.keys(allowed);

        if (!~implemented.indexOf(ctx.method)) {
          if (options.throw) {
            var notImplementedThrowable;
            if (typeof options.notImplemented === 'function') {
              notImplementedThrowable = options.notImplemented(); // set whatever the user returns from their function
            } else {
              notImplementedThrowable = new HttpError.NotImplemented();
            }
            throw notImplementedThrowable;
          } else {
            ctx.status = 501;
            ctx.set('Allow', allowedArr.join(', '));
          }
        } else if (allowedArr.length) {
          if (ctx.method === 'OPTIONS') {
            ctx.status = 200;
            ctx.body = '';
            ctx.set('Allow', allowedArr.join(', '));
          } else if (!allowed[ctx.method]) {
            if (options.throw) {
              var notAllowedThrowable;
              if (typeof options.methodNotAllowed === 'function') {
                notAllowedThrowable = options.methodNotAllowed(); // set whatever the user returns from their function
              } else {
                notAllowedThrowable = new HttpError.MethodNotAllowed();
              }
              throw notAllowedThrowable;
            } else {
              ctx.status = 405;
              ctx.set('Allow', allowedArr.join(', '));
            }
          }
        }
      }
    });
  };
};
```
### routes方法

```js
**
 * Returns router middleware which dispatches a route matching the request.
 * 返回一个匹配当前路由的router中间件
 * @returns {Function}
 */

Router.prototype.routes = Router.prototype.middleware = function () {
  var router = this;

  var dispatch = function dispatch(ctx, next) {
    debug('%s %s', ctx.method, ctx.path);

    var path = router.opts.routerPath || ctx.routerPath || ctx.path;
    var matched = router.match(path, ctx.method);
    var layerChain, layer, i;

    if (ctx.matched) {
      ctx.matched.push.apply(ctx.matched, matched.path);
    } else {
      ctx.matched = matched.path;
    }

    ctx.router = router;

    if (!matched.route) return next();

    var matchedLayers = matched.pathAndMethod
    var mostSpecificLayer = matchedLayers[matchedLayers.length - 1]
    ctx._matchedRoute = mostSpecificLayer.path;
    if (mostSpecificLayer.name) {
      ctx._matchedRouteName = mostSpecificLayer.name;
    }

    layerChain = matchedLayers.reduce(function(memo, layer) {
      memo.push(function(ctx, next) {
        ctx.captures = layer.captures(path, ctx.captures);
        ctx.params = layer.params(path, ctx.captures, ctx.params);
        ctx.routerName = layer.name;
        return next();
      });
      return memo.concat(layer.stack);
    }, []);

    return compose(layerChain)(ctx, next);
  };

  dispatch.router = this;

  return dispatch;
};
```