### 前言

对于前端资讯比较敏感的同学，可能这两天已经听说了 GoogleChromeLabs/quicklink 这个项目：它由 Google 公司著名开发者 Addy Osmani 发起，实现了：在空闲时间预获取页面可视区域内的链接，加快后续加载速度。

### 工作原理

Quicklink 通过以下方式加快后续页面的加载速度：

- 检测视区中的链接（使用 [Intersection Observer](https://developer.mozilla.org/en-US/docs/Web/API/Intersection_Observer_API) ）
- 等待浏览器空闲（使用 [requestIdleCallback](https://developer.mozilla.org/en-US/docs/Web/API/Window/requestIdleCallback) ）
- 检查用户是否处于慢速连接（使用 navigator.connection.effectiveType）或启用了省流模式（使用 navigator.connection.saveData）
- 预获取视区内的 URL（使用<link rel=prefetch>或 XHR）。 可根据请求优先级进行控制（若支持 fetch() 可进行切换）。

### 触发条件

如果用户的`有效连接类型`和`数据保护程序首选项表明它有用`的时候，
如果存在urls，则预取一系列URL，或者查看`document`的视口内链接。 如果进入窗口，就开始预加载

### API

quicklink 接受带有以下参数的 option 对象（可选）：

- el：指定需要预获取的 DOM 元素视区
- urls：预获取的静态 URL 数组（若此配置非空，则不会检测视区中 document 或 DOM 元素的链接）
- timeout：为 requestIdleCallback 设置的超时整数。 浏览器必须在此之前进行预获取（以毫秒为单位）， 默认取 2 秒。
- timeoutFn：指定超时处理函数。 默认为 requestIdleCallback。 也可以替换为 networkIdleCallback 等自定义函数（https://github.com/pastelsky/network-idle-callback，详见 demo）
- priority：布尔值，指定 fetch 的优先级。 默认为 false。 若配置为 true 将会尝试使用 fetch() API（而非 rel = prefetch）
- origins：允许预取的URL主机名字符串的数组。默认为相同的域源，可防止任何跨源请求。
- ignores：在origin检查后运行的自定义过滤器,默认没有

### 源码解读

1. 合并参数，并设置常量
```js
  options = Object.assign({
    timeout: 2e3,
    priority: false,
    timeoutFn: requestIdleCallback,
    el: document,
  }, options);

  observer.priority = options.priority;

  const allowed = options.origins || [location.hostname];
  const ignores = options.ignores || [];
```
2. 设置requestIdleCallback的callback和浏览器调用callback的最后期限

   这里回调函数提供了两种策略：
   - 如果参数中有urls，则只将urls所有的链接进行预加载，不会对dom下的其他链接进行预加载
   - 如果参数中没有urls，根据options的el来遍历其下的所有a标签，通过Intersection Observer来监控

   如果符合options.origins规则,且不符合options.ignores规则，将其放入预加载的列表`toPrefetch`中，
   可以通过不传options.origins来匹配所有
  
```js
const toPrefetch = new Set();
options.timeoutFn(() => {
    if (options.urls) {
      options.urls.forEach(prefetcher);
    } else {
      Array.from(options.el.querySelectorAll('a'), link => {
        // 把每一个a标签放入观察对象中，observer后面解释
        observer.observe(link);
        if (!allowed.length || allowed.includes(link.hostname)) {
          isIgnored(link, ignores) || toPrefetch.add(link.href);
        }
      });
    }
  }, {timeout: options.timeout})

```

3. 预加载

   那么预加载会做些什么呢？

   首先它会从`toPrefetch`删除这个即将请求的url

   然后通过`preFetched`判断是否已经加载过了，来减少不必要的请求。
  
   然后它会判断当前是否为2g或者省流量模式，如果是，则不做任何操作。

   接着会判断请求类型，默认是rel = prefetch，为true的时候，将会用fetch去请求数据，并对fetch做兼容。
  
   最后，更新`preFetched`这个对象

```js
// index.mjs
function prefetcher(url) {
  toPrefetch.delete(url);
  prefetch(new URL(url, location.href).toString(), observer.priority);
}

// prefetch.mjs
const preFetched = {};

function prefetch(url, isPriority, conn) {
  if (preFetched[url]) {
    return;
  }
  if (conn = navigator.connection) {
    if ((conn.effectiveType || '').includes('2g') || conn.saveData) return;
  }
  return (isPriority ? highPriFetchStrategy : supportedPrefetchStrategy)(url).then(() => {
    preFetched[url] = true;
  });
};
```
4. Intersection Observer

   通过新建一个观察者，来观察放入观察的a标签，当a标签进入窗口的时候，则开始预加载这个链接

```js

const observer = new IntersectionObserver(entries => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      const url = entry.target.href;
      if (toPrefetch.has(url)) prefetcher(url);
    }
  });
});
```
### 扩展阅读

- [你应该知道的requestIdleCallback](https://segmentfault.com/a/1190000014457824)
- [quicklink 为你的页面实现秒开](https://juejin.im/post/5c127d396fb9a049bc4c88bd)
- [使用 Intersection Observer 实现图片延迟加载](https://juejin.im/entry/59dafd506fb9a00a6a747079)
- [使用交叉点观察器延迟加载图像以提高性能](https://juejin.im/post/5abe4c0ef265da239c7b7a2b)
- [你应该知道的requestIdleCallback](https://juejin.im/post/5ad71f39f265da239f07e862)

### 参考

- [quicklink 为你的页面实现秒开](https://juejin.im/post/5c127d396fb9a049bc4c88bd)
