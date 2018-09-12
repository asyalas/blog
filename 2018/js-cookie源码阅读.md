# 介绍

js-cookie是社区比较认可的一个简单的，轻量级的处理cookies的js API
本文只做源码阅读，知其然知其所以然，中文文档请转到 https://blog.csdn.net/qq_20802379/article/details/81436634

# 源码阅读

## 整体结构、变量、方法介绍
- IIFE
``` js
(function(){}())
```
通过IIFE来包裹业务逻辑，目的简单：1、避免全局污染。2、保护隐私

- 注入
``` js
  var registeredInModuleLoader
  //方法一 AMD注入
  if (typeof define === 'function' && define.amd) {
    define(factory)
    registeredInModuleLoader = true
  }
  //方法二 es6模块注入
  if (typeof exports === 'object') {
    module.exports = factory()
    registeredInModuleLoader = true
  }
  //方法三 原生的script注入
  if (!registeredInModuleLoader) {
    var OldCookies = window.Cookies
    var api = window.Cookies = factory()
    api.noConflict = function () {
      window.Cookies = OldCookies
      return api
    }
  }
```
这里提供了三种js-cookie注入的方法：AMD注入，es6模块注入以及原生的script注入
其中提供一个noConflict方法，是为了在某些环境下解决命名冲突
执行后，会返回一个api，里面是js-cookie对象，window.Cookies不变

- extend 方法
``` js

  function extend () {
    var i = 0
    var result = {}
    for (; i < arguments.length; i++) {
      var attributes = arguments[ i ]
      for (var key in attributes) {
        result[key] = attributes[key]
      }
    }
    return result
  }

```
多个对象合并，类似es6的Object.assign,相同key值后面覆盖前面

- decode方法

``` js
function decode (s) {
    return s.replace(/(%[0-9A-Z]{2})+/g, decodeURIComponent)
}
```
将%后面的两位数进行解码

- js-cookie源码
  - 1.定义js-cookie set方法
  
  ``` js
    api.set = set
    function set (key, value, attributes) {
      //判断当前环境
      if (typeof document === 'undefined') {
        return
      }
      //合并attributes参数，defaults参数可以通过js-cookie.defaults全局统一设置
      attributes = extend({
        path: '/'
      }, api.defaults, attributes)
      //设置过期时间
      //如果参数expires为number类型，过期时间 = 当前时间 + number天
      if (typeof attributes.expires === 'number') {
        attributes.expires = new Date(new Date() * 1 + attributes.expires * 864e+5)
      }

      // 翻译： 我们用expires是因为max-age在ie下不支持
      // 把过期日期转换为（根据 UTC）字符串：
      attributes.expires = attributes.expires ? attributes.expires.toUTCString() : ''

      //检查是否是对象或者数组，如果是，则字符串后赋值给value
      try {
        var result = JSON.stringify(value)
        if (/^[\{\[]/.test(result)) {
          value = result
        }
      } catch (e) {}
      //对值进行url编码，编码后的特殊字符进行解码
      value = converter.write
        ? converter.write(value, key)
        : encodeURIComponent(String(value))
          .replace(/%(23|24|26|2B|3A|3C|3E|3D|2F|3F|40|5B|5D|5E|60|7B|7D|7C)/g, decodeURIComponent)
      //对key进行url编码，编码后的特殊字符进行解码,对（）进行编码
      key = encodeURIComponent(String(key))
        .replace(/%(23|24|26|2B|5E|60|7C)/g, decodeURIComponent)
        .replace(/[\(\)]/g, escape)
      //遍历attributes，把有值且key不为true的属性合成一个字符串
      //类似 '; path=/; expires=Thu, 04 Oct 2018 05:56:19 GMT'
      var stringifiedAttributes = ''
      for (var attributeName in attributes) {
        if (!attributes[attributeName]) {
          continue
        }
        stringifiedAttributes += '; ' + attributeName
        if (attributes[attributeName] === true) {
          continue
        }

        // Considers RFC 6265 section 5.2:
        // 名称-值对（name-value-pair）字符串由直到第一个%x3B(“;”)但不包括(“;”)的字符组成，未分析的属性由剩余的 
        // set-cookie-string组成。
        stringifiedAttributes += '=' + attributes[attributeName].split(';')[0]
      }
      //拼接name-value-pair存入cookie中
      return (document.cookie = key + '=' + value + stringifiedAttributes)
    }
  ```
  - 2.定义js-cookie get方法
  ``` js
    //以字符串形式返回
    api.get = function (key) {
      return get(key, false /* read as raw */)
    }
    //以json格式返回
    api.getJSON = function (key) {
      return get(key, true /* read as json */)
    }
		function get (key, json) {
      //判断当前环境
      if (typeof document === 'undefined') {
        return
      }

      var jar = {}
      // [翻译]首先定义一个空数组
      // [翻译]以防万一cookie根本没有，导致for循环报错
      var cookies = document.cookie ? document.cookie.split('; ') : []
      //获取cookie的所有的值
      var i = 0
      for (; i < cookies.length; i++) {
        var parts = cookies[i].split('=')
        var cookie = parts.slice(1).join('=')

        if (!json && cookie.charAt(0) === '"') {
          cookie = cookie.slice(1, -1)
        }

        try {
          //对key和value进行解码
          var name = decode(parts[0])
          cookie = (converter.read || converter)(cookie, name) ||
						decode(cookie)
          //如果json为true，则value以json形式返回
          if (json) {
            try {
              cookie = JSON.parse(cookie)
            } catch (e) {}
          }

          jar[name] = cookie
          //key存在且匹配到停止循环,否则则是全量
          if (key === name) {
            break
          }
        } catch (e) {}
      }
      //key存在且返回key的value,否则则是全量
      return key ? jar[key] : jar
    }
  ```
  - 3.定义js-cookie remove方法
  ``` js
    api.remove = function (key, attributes) {
        //值制为空，且立即失效
        api(key, '', extend(attributes, {
          expires: -1
        }));
    };
  ```
  - 4.扩展方法
  ``` js
    //全局设置cookie的属性
    api.defaults = {}
    //自定义read和write函数，可以按需定制是否编码及中间件
    api.withConverter = init
  ```
  
