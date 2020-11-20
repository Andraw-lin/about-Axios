## 回顾拦截器的使用

只要在项目中使用过`axios`，那么基本就脱离不了拦截器这一功能。接下来我们先来回顾一下拦截器都有哪些用法，对后面源码分析时可有一个更好的理解。

1. 向请求或响应中添加拦截器。

   ```js
   // 添加请求拦截器
   axios.interceptors.request.use(function(config) {
     // 在请求发送前做一些处理，其中config为请求配置项
     return config;
   }, function(error) {
     // 当请求发生错误时进行处理
     return Promise.reject(error);
   });
   
   // 添加响应拦截器
   axios.interceptors.respose.use(function(res) {
     // 在响应返回前做一些处理，其中res为响应内容
     return res;
   }, function(error) {
     // 当响应返回错误时进行处理
     return Promise.reject(error);
   });
   ```

2. 移除拦截器。

   ```js
   const interceptors = axios.interceptors.request.use(function(config) {
     // 在请求发送前做一些处理，其中config为请求配置项
     return config;
   }, function(error) {
     // 当请求发生错误时进行处理
     return Promise.reject(error);
   });
   // 移除请求拦截器
   axios.interceptors.request.eject(interceptors);
   ```

以上两个功能可以说是拦截器中最核心的功能，主要用于在请求发送前对于数据进行统一处理，又或者在响应返回前对数据进行额外处理。

其实在上一章节讲到`axios`的执行过程时，就有提及过拦截器的处理，**`axios`内部在处理请求时，会采用队列的形式，然后在队列前面加入请求拦截器，而在队列后面加入响应拦截器**。

那么问题来了，既然在处理请求时会在队列中加入请求拦截器和响应拦截器，那么**拦截器对于请求和响应是一起收集呢？还是分开收集呢？**

要想知道答案，就让我们在源码中一起探究。



##源码分析

其实在`Axios`构造函数中，就已经存了两份拦截器，分别是请求拦截器和响应拦截器。

```js
function Axios(instanceConfig) {
  this.defaults = instanceConfig; // 默认传入配置项
  this.interceptors = { // 拦截器
    request: new InterceptorManager(), // 请求拦截器
    response: new InterceptorManager() // 响应拦截器
  };
}
```

不管是请求拦截器还是响应拦截器，都是作为`InterceptorManager`实例，那么在`InterceptorManager`中就肯定包含了拦截器的构造以及实现。

接下来我们就来看看`InterceptorManager`的内部实现。

```js
function InterceptorManager() { // 拦截器类
  this.handlers = []; // 用于存储请求和响应各自处理函数的栈
}

InterceptorManager.prototype.use = function use(fulfilled, rejected) { // 拦截器的原型对象上实现use添加拦截器方法
  this.handlers.push({ // 每调用一次use方法，都会向栈中添加一个对象，该对象包含两个处理函数，分别为成功处理函数以及失败处理函数
    fulfilled: fulfilled,
    rejected: rejected
  });
  return this.handlers.length - 1; // 最终返回栈的长度，与原生的Array.prototype.push方法返回保持了一致
};

InterceptorManager.prototype.eject = function eject(id) { // 拦截器的原型对象上实现
  if (this.handlers[id]) { // 根据相应的id进行删除，会把相应的拦截器的成功处理函数以及失败处理函数都统一删除
    this.handlers[id] = null;
  }
};
```

看完代码，那你对于上述提出的问题也有了答案---拦截器对于请求和响应是一起收集呢？还是分开收集呢？

其实，`axios`实现了一个拦截器类，而**请求和响应都会单独创建各自的拦截器实例，因此请求拦截器和响应拦截器就会有各自单独的栈用来保存相应的拦截处理对象**。

另外，**拦截器原型对象上实现的`use`方法其实就是直接利用了栈的`push`思想，而`eject`方法则是直接根据相应的`id`来设置为`null`处理**。

现在，我们再来回过头看一下上一章节中在分析源码时会有以下一段代码：

```js
this.interceptors.request.forEach(function unshiftRequestInterceptors(interceptor) { // 在队列的前方加入请求拦截器
  chain.unshift(interceptor.fulfilled, interceptor.rejected);
});

this.interceptors.response.forEach(function pushResponseInterceptors(interceptor) { // 在队列的后方加入响应拦截器
  chain.push(interceptor.fulfilled, interceptor.rejected);
});
```

其中，`this.interceptors.request`和`this.interceptors.respose`就是分别代表了请求拦截器和响应拦截器，它们两个都是**通过遍历自身的栈，然后分别将遍历到的拦截处理对象`unshift`或者`push`处理**。

至此，相信您对拦截器的原理也有一定的理解。

总结一句便是，**拦截器本身采用的是栈数据结构来对拦截处理对象进行收集，而请求处理时则是通过队列形式来实现请求拦截---发送请求---响应拦截的过程**。











































