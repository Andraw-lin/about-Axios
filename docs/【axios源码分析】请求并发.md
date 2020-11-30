在[从入口分析axios封装](https://github.com/Andraw-lin/about-Axios/blob/main/docs/%E3%80%90axios%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E3%80%91%E4%BB%8E%E5%85%A5%E5%8F%A3%E5%88%86%E6%9E%90axios%E5%B0%81%E8%A3%85.md)中，有提及到两个静态方法，分别是`axios.all`和`axios.spread`，如果看过那篇分享的话，相信你肯定知道，**`axios.all`和`axios.spread`这两个方法都是用来处理请求并发的**。

先岔开话题，我们先来看看它们两个是如何使用的。

```js
import axios from 'axios';

export default {
  // ...
  mounted() {
    axios.all([this.getUser, this.getName])
    	.then(axios.spread((user, name) => {
      	console.log(user, name); // 分别输出用户以及名字
    	}))
  },
  methods: {
    getUser() { // 获取用户
      return axios({
        url: '...',
        params: {
          // ...
        }
      })
    },
    getName() { // 获取名字
      return axios({
        url: '...',
        params: {
          // ...
        }
      })
    }
  }
  // ...
}
```

从使用上，可以看到`axios.all`和`axios.spread`是结合使用的，并且最终在所有请求都返回后，再按输入顺序逐个输出（即`getUser`对应`user`，而`getName`对应`name`）。

因此，`axios.all`只是收集请求，而`axios.spread`则是将内容按顺序输出。既然如此，那在源码实现上又是如何的呢？事不宜迟，我们就从源码角度来看看。



## 源码分析

咋一看源码，你会发现`axios.all`和`axios.spread`这两个方法的实现是很简单的，有多简单，你看看下面就清楚了。

```js
axios.all = function all(promises) {
  return Promise.all(promises);
};
axios.spread = function spread(callback) {
  return function wrap(arr) {
    return callback.apply(null, arr);
  };
};
```

好明显，`axios.all`方法仅仅只是对`Promise.all`的一个封装。`Promise.all`可谓是我们在日常开发中经常会使用到的方法，也许有些童鞋会对此有些陌生，甭急，当看完下面代码，肯定会茅塞顿开！😄

```js
var fn = time => {
 	return new Promise((res, rej) => {
    setTimeout(() => {
      res(time)
    }, time * 1000);
  });
}
Promise.all([fn(3), fn(1), fn(2)]).then(res => {
  console.log(res);
})
```

你会觉得最终输出什么？答案就是`[3, 1, 2]`，以数组形式输入，并且最终也按顺序以数组形式输出。

另外，有一个好明显的地方就是，**`Promise.all`的数组参数中，必须都是`Promise`实例**。同样的道理，**`axios.all`中数组参数中的每一项元素也必须是`Promise`实例**。

好啦，我们接下来再来看看`axios.spread`方法。

由于上面提及过，`axios.spread`方法会结合`axios.all`方法一起使用，那么为什么要结合使用呢？

从源码可以看到，`axios.spread`实则是一个**闭包**，并且最外层参数是一个回调函数，在执行函数的参数中则是一个数组，并且把该数组通过`apply`传入到回调函数中作为其参数使用。

由于`axios.all`是在处理`then`时，返回结果是以数组返回，而**`axios.spread` 则是利用`apply`并且结合闭包，将数组形式转换成逐个参数形式，并且不改变原写法的情况下**。

















































