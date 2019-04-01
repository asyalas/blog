### 目录
- ajax
  - 简介
  - 简单的封装
  - XHR1级／2级
- fetch
  - 简介
  - fetch相对于ajax的优点
  - fetch相对于ajax的缺点
- axios
  - 简介
  - 优点
- Superagent
  - 简介
  - 优点
  - 缺点
- 其他新兴的请求api

### ajax

#### 简介

AJAX即“Asynchronous Javascript And XML”，即`异步JS与XML`，是指一种创建交互式网页应用的网页开发技术。
注：ajax的异步属于macrotask

#### 简单的封装

```js
function getXHR({method,url,isAsync=true,body,onSuccess,onError}){
  var xhr = null;
  if(window.XMLHttpRequest) {
    xhr = new XMLHttpRequest();
  } else if (window.ActiveXObject) {
    try {
      xhr = new ActiveXObject("Msxml2.XMLHTTP");
    } catch (e) {
      try {
        xhr = new ActiveXObject("Microsoft.XMLHTTP");
      } catch (e) { 
        alert("您的浏览器暂不支持Ajax!");
      }
    }
  }
  xhr.open(method,url,isAsync);  
  xhr.setRequestHeader("Content-type","application/x-www-form-urlencoded");  
  xhr.send(body);  //post请求参数放在send里面，即请求体,此时body的结构为"fname=Henry&lname=Ford"
  xhr.onreadystatechange=function()  { 
    if (xhr.readyState==4 &&xhr.status==200)  { 
      onSuccess(xhr.responseText);  
    }else{
      onError(xhr)
    }
  }
}
```

#### ajax的几个常用的属性

- onprogress
  onprogress事件回调方法在 readyState==3 状态时开始触发, 默认传入 ProgressEvent 对象, 可通过 e.loaded/e.total 来计算加载资源的进度, 该方法用于获取资源的下载进度
- ontimeout
  ontimeout方法在ajax请求超时时触发, 通过它可以在ajax请求超时时做一些后续处理.
- withCredentials
  withCredentials是一个布尔值, 默认为false, 表示跨域请求中不发送cookies等信息.
- abort
  abort方法用于取消ajax请求, 取消后, readyState 状态将被设置为 0 (UNSENT). 
- upload
  upload属性默认返回一个 XMLHttpRequestUpload 对象, 用于上传资源
- onerror
  onerror方法用于在ajax请求出错后执行.

#### XHR1级／2级

##### XHR1级

XHR1 即 XMLHttpRequest Level 1. XHR1时, xhr对象具有如下缺点:

- 仅支持文本数据传输, 无法传输二进制数据.
- 传输数据时, 没有进度信息提示, 只能提示是否完成.
- 受浏览器 同源策略 限制, 只能请求同域资源.
- 没有超时机制, 不方便掌控ajax请求节奏.
  
##### XHR2级

XHR2 即 XMLHttpRequest Level 2. XHR2针对XHR1的上述缺点做了如下改进:

- 支持二进制数据, 可以上传文件, 可以使用FormData对象管理表单.
- 提供进度提示, 可通过 xhr.upload.onprogress 事件回调方法获取传输进度.
- 依然受 同源策略 限制, 这个安全机制不会变. XHR2新提供 Access-Control-Allow-Origin 等headers, 设置为 * 时表示允许任何域名请求, 从而实现跨域CORS访问(有关CORS详细介绍请耐心往下读).
- 可以设置timeout 及 ontimeout, 方便设置超时时长和超时后续处理.

### fetch

#### 简介

fetch号称是ajax的替代品，它的API是基于`Promise`设计的，旧版本的浏览器不支持Promise，需要使用polyfill es6-promise

#### fetch相对于ajax的优点

- 语法简洁，更加语义化
- 基于标准 Promise 实现，支持 async/await
- 同构方便，使用 isomorphic-fetch

#### fetch相对于ajax的缺点

- 默认不带cookie,需要设置 fetch(url, {credentials: 'include'})
- 需要底层支持，或者使用第三方兼容包
- 不能做超时处理
- 没有abort方法
- 遇到常见的错误码不会报错，需要手动去封装

### axios

#### 简介

Axios 是一个基于`Promise`对于`ajax`封装的 HTTP 库，可用在 Node.js 和浏览器上发起 HTTP 请求，支持所有现代浏览器，甚至包括 IE8+！

#### 优点

由于是基于ajax封装，所以功能比fetch要强大一点。

- 同时支持 Node.js 和浏览器
- 支持 Promise API
- 可以配置或取消请求（ajax）
- 可以设置响应超时（ajax）
- 支持防止跨站点请求伪造（XSRF）攻击
- 可以拦截未执行的请求或响应
- 支持显示上传进度（ajax）


### Superagent

#### 简介
Superagent 是一个基于 Promise 的`轻量级渐进式` AJAX API，非常适合发送 HTTP 请求以及接收服务器响应。 与 Axios 相同，它既适用于 Node，也适用于所有现代浏览器。

#### 优点

- 它有一个插件生态，通过构建插件可以实现更多功能
- 可配置
- HTTP 请求发送接口友好
- 可以为请求链式添加方法
- 适用于浏览器和 Node
- 支持显示上传和下载进度
- 支持分块传输编码
- 支持旧风格的回调
- 繁荣的插件生态，支持众多常见功能

#### 缺点

- 其 API 不符合任何标准


### 其他新兴的请求api

- request
- fly