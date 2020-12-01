## 背景

咋一看，这文章的标题是什么鬼？甭急，慢慢往下看。🤔

对于文章标题所提出的问题，我可以肯定并不是所有的小伙伴都遇到过，但是也肯定有部分小伙伴遇到过。先来看一下以下栗子：

```js
import axios from 'axios';

const myAxios = axios.create({
  url: '...',
  // ...省略其他配置
});
myAxios.all([getUser, getName])
  .then(myAxios.spread((user, name) => {
  	console.log(user, name);
	});
```

看上述的栗子，能否正确地输出请求返回的`user`以及`name`呢？

答案是否定，而且还会报错！！！而且报的错就是这篇文章标题提及到的问题，在`myAxios`中找不到相应的`all`方法和`spread`方法。

先别急着想为什么，我们再来看看下面的栗子。

```js
import axios from 'axios';

// ...省略其他代码
axios.all([getUser, getName])
  .then(myAxios.spread((user, name) => {
  	console.log(user, name);
	});
```

再来思考下是否能正确展示相应的数据吗？答案就是肯定的。

通过上述这两个栗子，相信有不少的小伙伴已经看出了区别 ，第一个使用了`axios.create`实例，而第二个则使用了`axios`本身。

问题来了，既然`axios.create`实例和`axios`本身都能用于发起请求，那么这两者之间为什么又会有区别呢？

我们接着就来从源码的角度来进行分析。



## 源码分析

我们再一次回到`./lib/axios.js`文件中。

```js
function createInstance(defaultConfig) {
  var context = new Axios(defaultConfig);
  var instance = bind(Axios.prototype.request, context);

  // Copy axios.prototype to instance
  utils.extend(instance, Axios.prototype, context);

  // Copy context to instance
  utils.extend(instance, context);

  return instance;
}

// Create the default instance to be exported
var axios = createInstance(defaults);

// Expose Axios class to allow class inheritance
axios.Axios = Axios;

// Factory for creating new instances
axios.create = function create(instanceConfig) {
  return createInstance(mergeConfig(axios.defaults, instanceConfig));
};

// Expose Cancel & CancelToken
axios.Cancel = require('./cancel/Cancel');
axios.CancelToken = require('./cancel/CancelToken');
axios.isCancel = require('./cancel/isCancel');

// Expose all/spread
axios.all = function all(promises) {
  return Promise.all(promises);
};
axios.spread = require('./helpers/spread');

// Expose isAxiosError
axios.isAxiosError = require('./helpers/isAxiosError');

module.exports = axios;

// Allow use of default import syntax in TypeScript
module.exports.default = axios;
```

如果你们看到前面所分享的文章，那肯定不陌生。

首先，最终所暴露的`axios`是通过`createInstance`方法创建的，而`createInstance`方法内部有两个要点，分别是

- 通过`bind`方法，将`Axios.prototype.request`请求方法作用域绑定在`context`下，并且返回一个新的函数。
- 通过`extend`方法，将参数中第二个对象的所有属性以及方法都继承到上一步所得到的新函数中，例如将`get` 、`post`等方法继承到 新函数中。

接着，我们可以看到，**`Cancel`、`CancelToken`、`isCancel`、`all`、`spread`方法都是定义在上面得到新函数的静态方法上，而非原型对象上**，这一点很关键。

再看看`create`方法，实质上是将`createInstance`方法再包装了一层，并返回类似第一步中所得到新函数。

通过对比会发现，**`axios.create`实例和`axios`本身会有一些细微上的区别，那就是静态属性以及静态方法**。

说白了，`axios.create`实例返回的新函数自身是不会带有任何的静态属性或者静态方法的，相反`axios`本身则拥有静态属性以及静态方法（如`Cancel`、`CancelToken`、`isCancel`、`all`、`spread`方法）。而这也是这篇文章的标题——为何某些情况下无法使用`all`和`cancel`等方法的原因。



## 解决方法

既然已经知道某些情况下用不了`all`或者`cancel`等方法是由于`axios.create`实例不带有静态方法 `all`、`cancel`等所导致的，那么有没有什么补救的方法？

在论坛里面，提到**最简单的解决方案就是，直接重新引入`axios`包**。但是，这样处理真的能解决问题吗？

好明显，如果仅仅是想通过重新引入`axios`包来来进行处理，肯定是不能根本地解决这个问题。为什么？

一方面，我们没法通过`axios.create`来设置一些公用的配置项，如超市时间、公共请求头部等，另一方面，单单是重新引入`axios`包，会导致代码变得冗余并且会很麻烦。

既然如此，有没有一种更好的方案？

也许有些小伙伴会觉得修改源码！但是细心一看，当我们刚改`axios`版本时，岂不是又白费了？🤔

其实可以**有一种优雅的暴力方案能够很好滴解决，那就是直接通过应用层修改`axios.create`的原型指向**。

```js
const axios = require('axios');

const myAxios = axios.create({
  baseURL: 'https://www.google.com.hk'
})

/* eslint-disable no-proto */
myAxios.__proto__ = axios
/* eslint-enable */

module.exports = axios
```

可以看到，`axios.create`创建的实例`myAxios`直接将其`__proto__`修改指向了`axios`对象。

这样一来，就可以将`axios`对象上的静态方法以及静态属性都能与`myAxios`进行共享。

另外，你会发现，代码中还**多出了两行注释：`eslint-disable no-proto`和`eslint-enable`，有什么用呢？**

其实是**由于`Eslint`不允许直接修改对象的原型指向，因此需要加入注释来允许修改**。



































