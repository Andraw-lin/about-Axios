## 何为取消请求

想象现在有一个需求，实现一个`Tab`组件，每一个`Tab`都会对应一个内容，而且内容都是通过请求得到的。

我们在实现的时候，常常就是在`Tab`标签上添加一个可复用点击事件，接着在点击事件里向后端请求相应内容。瞅一眼貌似并没有啥问题，但是当我们细心观察时，就会发现一个问题，当我们**频繁点击`Tab`标签时，会出现内容展示不正确**。为啥？

原因很简单，其实就是因为**当我们点击标签时都会发送一个请求**，当然**最后响应的内容并不能保证对应的就是最后一次所点击的标签**。

当然，相信童鞋们也会想到很多方案，如节流、从`UI`层面控制请求的发送等等。方案都是很灵活的，而今天所讲的一种方案就是**取消请求**，原理很简单，就是**当点击不同标签，并且在发送请求前，先将前面没有内容返回或者还没进行发送请求的行为都砍掉，然后保证当前请求队列只有一个，这样一来，就能保证所点击的标签展示的内容是一一对应的**。

那么回到`axios`身上，先来看看它的取消请求是如何使用的。

首先，`axios`暴露一个`CancelToken`属性对象来专门用于处理请求取消的。

```js
const cancelToken = axios.CancelToken;
const source = cancelToken.source();

const getTabContent = key => {
  source.cancel(); // 频繁执行请求时，先将该类请求都取消掉
  return axios.get('/user/12345', {
    cancelToken: source.token
  }).catch(function(thrown){
    if (axiso.isCancel(thrown)) {
      console.log('Request canceled', thrown.message);
    } else {
      //handle error
    }
  });
}
```

当然还可以使用构造函数形式来编写：

```js
const CancelToken = axios.CancelToken;
let cancel;

const getTabContent = key => {
  if (typeof cancel === 'function') { // 频繁执行请求时，先判断cancel是否已经函数，若是则直接执行去掉先前的请求
    cancel();
  }
  return axios.get('/user/12345', {
    cancelToken: new CancelToken(function executor(c) {
      cancel = c;
    })
  }).catch(function(thrown){
    if (axiso.isCancel(thrown)) {
      console.log('Request canceled', thrown.message);
    } else {
      //handle error
    }
  });
}
```

可以看到，通过`CancelToken`来取消请求可以有两种方式，分别是：

- 使用`CancelToken.source`工厂函数。
- 使用`CancelToken`构造函数，回调函数中参数即取消函数。

在使用上难度并不大，通过上一节请求的实现中，我们可以清楚，`axios`实现请求的发送最终只是在`XHRHTTPRequest`对象以及`HTTP`对象上所做的一层封装 。那么，取消请求的实现是否也是使用这两个对象呢？事不宜迟，让我们从源码角度来分析一下。



## 源码分析

我们先来看下`CancelToken`自身的结构：

```js
function CancelToken(executor) { // 接受一个executor回调函数
  if (typeof executor !== 'function') { // 判断参数若不是函数则直接抛出异常
    throw new TypeError('executor must be a function.');
  }

  var resolvePromise; // 关键点，用于保存resolve对象
  this.promise = new Promise(function promiseExecutor(resolve) {
    resolvePromise = resolve;
  });

  var token = this;
  executor(function cancel(message) { // 执行回调函数
    if (token.reason) { // 执行回调时，判断请求是否已经取消，若是直接返回
      // Cancellation has already been requested
      return;
    }

    token.reason = new Cancel(message); // 否则，直接从Cancel构造函数中创建取消信息对象
    resolvePromise(token.reason); // 最后把错误信息对象作为resolve对象参数执行下一步
  });
}

CancelToken.prototype.throwIfRequested = function throwIfRequested() { // 若已经创建取消信息对象，直接抛出该信息出来
  if (this.reason) {
    throw this.reason;
  }
};

CancelToken.source = function source() { // 静态方法，返回一个对象，其中包括CancelToken构造函数以及cancel取消函数
  var cancel;
  var token = new CancelToken(function executor(c) {
    cancel = c;
  });
  return {
    token: token,
    cancel: cancel
  };
};
```

通过观看完`CancelToken`，再结合上面的例子，我们知道，取消请求最终调取的就是`cancel`方法，既然取消请求有两种方式，分别是通过`CancelToken`构造函数以及直接使用`source`静态方法。

我们先看通过`CancelToken`构造函数的情况，下面的写法就是使用构造函数的常用写法：

```js
let cancel;
axios.get('/user/12345', {
  cancelToken: new CancelToken(function executor(c) {
    cancel = c;
  })
});
```

通过源码可以看到，`CancelToken`构造函数执行时，由`token = this`得出构造函数得到对象其实就是`token`对象，而且回调函数中的`c`其实就是`cancel`函数了。

再来看看**`source`静态方法**，从源码可以看到，其**内部实现就是对`CancelToken`构造函数做了一层封装**，并且返回`token`对象以及`cancel`取消方法。

因此，可以得出，最终的`cancel`取消请求方法就是如下：

```js
function cancel(message) { // 执行回调函数
  if (token.reason) { // 执行回调时，判断请求是否已经取消，若是直接返回
    // Cancellation has already been requested
    return;
  }

  token.reason = new Cancel(message); // 否则，直接从Cancel构造函数中创建取消信息对象
  resolvePromise(token.reason); // 最后把错误信息对象作为resolve对象参数执行下一步
}
```

当我们执行`cancel`取消请求方法时，其实就是执行了暴露在外层的`resolve`对象。

那么问题来了，既然有了取消请求的方法`cancel`，那取消请求是怎么执行的呢？甭急，现在就来看看，首先看下`xhr.js`里的封装：

```js
// ...
if (config.cancelToken) {
  // Handle cancellation
  config.cancelToken.promise.then(function onCanceled(cancel) {
    if (!request) {
      return;
    }

    request.abort();
    reject(cancel);
    // Clean up request
    request = null;
  });
}
// ...
```

其余多余代码就不贴出来啦，就把主要代码拿出来，可以看到，会判断请求中是否写有`cancelToken`属性，接着就会把`CancelToken`中定义的`promise`属性执行其`then`方法，这样一来该`then`的处理就会俾放进事件队列中等待是否执行取消请求。

当我们调用`cancel`方法时，最终就直接执行了`request.abort`方法，即把上一次请求给砍掉。

同样的道理，我们再来看下在`Node`环境下是不是也是同样的封装？

```js
// ...
if (config.cancelToken) {
  // Handle cancellation
  config.cancelToken.promise.then(function onCanceled(cancel) {
    if (req.aborted) return;

    req.abort();
    reject(cancel);
  });
}
// ...
```

可以看到，最终依然还是执行了`req.abort`方法来取消上一次的请求。

因此，**`axios`实现整个取消请求，其实内部实现就是把通过`promise` 来把每一个`then`放进事件队列中等待执行，而每一个`then`方法里面都是直接执行相应的`abort`方法来取消请求**。



 















































