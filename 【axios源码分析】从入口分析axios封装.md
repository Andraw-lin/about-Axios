## 从 axios 的简单语法糖谈起

如果你有在项目中使用过`axios`，那么对于以下的语法肯定不会陌生。

```js
// 使用axios发送get请求
axios.get('/user?ID=12345')
  .then(function (response) {
    console.log(response);
  })
  .catch(function (error) {
    console.log(error);
  });
// 使用axios发送post请求
axios.post('/user', {
    firstName: 'Fred',
    lastName: 'Flintstone'
  })
  .then(function (response) {
    console.log(response);
  })
  .catch(function (error) {
    console.log(error);
  });

axios({
  method: 'post',
  url: '/user/12345',
  data: {
    firstName: 'Fred',
    lastName: 'Flintstone'
  }
});
```

代码很好理解，无非就是利用`axios`来发送`get`请求和`post`请求。其中对于第三种发送`http`的方法，是否有一种似曾相识的感觉？

没错！就是模仿了`ajax`的写法。

```js
$ajax({
  // ...
})
```

看着上面这么便捷的语法糖，是否会引起你想对`axios`的封装过程进行探究呢？

（说句实话，我就是这样被吸引过去的😅）下面就让我们来一步一步地从入口**简单滴**看看源码是如何来对`axios`封装的。



## 源码分析

先找到入口文件`/lib/axios.js`，下面就将代码进行缩减一下进行分析。

```js
function createInstance(defaultConfig) {
  var context = new Axios(defaultConfig); // 根据传入的配置参数，创建Axios实例
  var instance = bind(Axios.prototype.request, context); // 采用bind方法，将Axios原型上的request方法直接绑定在当前的context实例作用域上，并返回一个新的函数

  // Copy axios.prototype to instance
  utils.extend(instance, Axios.prototype, context); // 将Axios原型上的所有属性和方法都赋值在instance方法上，其中方法的作用域是绑定在context上

  // Copy context to instance
  utils.extend(instance, context); // 最后再将context上的属性和方法都赋值在instance方法上

  return instance; // 最后返回该instance方法
}

var axios = createInstance(defaults); // instance方法就是最终我们在语法糖上使用的axios对象

axios.Axios = Axios;

axios.create = function create(instanceConfig) { // 封装了创建实例的方法
  return createInstance(mergeConfig(axios.defaults, instanceConfig));
};

// 封装了取消请求方法，常用于处理频繁触发请求的情况
axios.Cancel = require('./cancel/Cancel');
axios.CancelToken = require('./cancel/CancelToken');
axios.isCancel = require('./cancel/isCancel');

// 封装并发请求处理，其中axios.all就是使用了Promise.all，而axios.spread则是在axios.all响应返回时使用
axios.all = function all(promises) {
  return Promise.all(promises);
};
axios.spread = require('./helpers/spread');

// 封装是否为axios错误处理
axios.isAxiosError = require('./helpers/isAxiosError');

// 最后将axios暴露出去
module.exports = axios;

// Allow use of default import syntax in TypeScript
module.exports.default = axios;
```

看完上面的代码，相信你也应该知道，其实我们在日常开发中引入的`axios`，其实就是一个方法，而且在这个方法上还封装了如`create`、`Cancel`、`all`、`spread`等静态方法，因此我们可以直接使用如下：

```js
import axios from 'axios';

axios.all([getUserAccount(), getUserPermissions()])
	.then(axios.spread((account, permission) => {
  	console.log(account, permission);
	}))
```

但是我看了又看，都没发现`axios`上有暴露出`get`、`post`等请求方法啊。是的，其实对于日常的请求方法，都存在`Axios`上，所以才会使用`utils.extend`将`Axios`原型上的方法以及属性都继承到`axios`中来。

接下来我们就来看看`Axios`是如何封装`get`、`post`等请求方法的。

```js
// Provide aliases for supported request methods
utils.forEach(['delete', 'get', 'head', 'options'], function forEachMethodNoData(method) {
  /*eslint func-names:0*/
  Axios.prototype[method] = function(url, config) { // 将上述数组中的请求方法都封装到Axios.prototype上
    return this.request(mergeConfig(config || {}, {
      method: method,
      url: url,
      data: (config || {}).data
    }));
  };
});

utils.forEach(['post', 'put', 'patch'], function forEachMethodWithData(method) {
  /*eslint func-names:0*/
  Axios.prototype[method] = function(url, data, config) { // 将上述数组中的请求方法都封装到Axios.prototype上
    return this.request(mergeConfig(config || {}, {
      method: method,
      url: url,
      data: data
    }));
  };
});
```

可以看到，请求方法`get`、`post`等请求方法都是封装到了`Axios.prototype`原型对象上。

好了，目前大概了解`axios.get`以及`axios.post`的封装，那么还剩一个`axios`方法呢？

你会发现，不管是`axios.get`和`axios.post`，还是直接使用`axios()`方法，其实最终都是使用了`Axios.prototype.request`方法。

那么`Axios.prototype.request`方法又是如何的呢？我们将在下一章节中继续探讨。









































