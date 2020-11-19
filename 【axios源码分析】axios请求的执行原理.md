从上一个章节可以知道，日常开发中`axios`所使用各种请求方法，其实最终都是在执行`Axios.prototype.request`方法。

在这里就不摆葫芦了，其实从`Axios.prototype.request`方法中，我们可以看出**整个`axios`在发送请求的过程中是如何对请求进行处理的**，下面我们就来一起探究一下。



## 源码分析

下面直接贴出相关源码。

```js
Axios.prototype.request = function request(config) {
  /*eslint no-param-reassign:0*/
  // Allow for axios('example/url'[, config]) a la fetch API
  if (typeof config === 'string') { // 判断配置是否为字符串，若是字符串那么该字符串肯定是url，而配置项则是作为第二个参数进行传递（即arguments[1]）
    config = arguments[1] || {};
    config.url = arguments[0];
  } else { // 若不是字符串，那么就是直接作为配置项进行处理
    config = config || {};
  }

  config = mergeConfig(this.defaults, config); // 将默认配置项与传递进来的配置项进行合并

  // Set config.method
  if (config.method) { // 若配置项中传递有请求方式，则作小写处理
    config.method = config.method.toLowerCase();
  } else if (this.defaults.method) { // 若默认配置项中有请求方式，则继续作小写处理
    config.method = this.defaults.method.toLowerCase();
  } else { // 否则最后默认选择get请求方式
    config.method = 'get';
  }

  // Hook up interceptors middleware
  // 采用队列数据结构来存储请求
  var chain = [dispatchRequest, undefined];
  var promise = Promise.resolve(config);

  this.interceptors.request.forEach(function unshiftRequestInterceptors(interceptor) { // 在队列的前方加入请求拦截器
    chain.unshift(interceptor.fulfilled, interceptor.rejected);
  });

  this.interceptors.response.forEach(function pushResponseInterceptors(interceptor) { // 在队列的后方加入响应拦截器
    chain.push(interceptor.fulfilled, interceptor.rejected);
  });

  while (chain.length) { // 最后执行队列中内容，每次拿两项出来，直到队列为空为止
    promise = promise.then(chain.shift(), chain.shift());
  }

  return promise;
};
```

可以看到，**`axios`请求的执行原理是通过队列的形式进行**，其中`dispatchRequest`就是我们发送的请求。

在执行队列中任务前，会先把请求的拦截器放到队列的前头，而响应拦截器则会放到队列的后面，得到的队列会是下面这样子：

```js
[请求拦截器成功处理, 请求拦截器失败处理, dispatchRequest, undefined, 响应拦截器成功处理, 响应拦截器失败处理]
```

因此，在队列执行过程中，每次都会拿两项出来，**放到`Promise.then`中**，这样一来，就可以**通过`Promise`来控制队列中任务的执行**，执行顺序也就是：**请求拦截器 --- 发送请求 --- 响应拦截器**。









































