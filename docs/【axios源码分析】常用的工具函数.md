工具函数在日常开发的过程中起着重要的地位，常常在一些逻辑处理之间能够进行复用，当然`axios`也不例外。现在就来看看`axios`中所封装的工具函数，除了学习以外，还能对后续探究其他功能源码的时候能一目了然（毕竟后续功能源码中都会使用到该工具函数～）。



## 源码分析

目录是在`/lib/utils.js`中，先来看看都封装了哪些工具函数：

```js
module.exports = {
  isArray: isArray,
  isArrayBuffer: isArrayBuffer,
  isBuffer: isBuffer,
  isFormData: isFormData,
  isArrayBufferView: isArrayBufferView,
  isString: isString,
  isNumber: isNumber,
  isObject: isObject,
  isPlainObject: isPlainObject,
  isUndefined: isUndefined,
  isDate: isDate,
  isFile: isFile,
  isBlob: isBlob,
  isFunction: isFunction,
  isStream: isStream,
  isURLSearchParams: isURLSearchParams,
  isStandardBrowserEnv: isStandardBrowserEnv,
  forEach: forEach,
  merge: merge,
  extend: extend,
  trim: trim,
  stripBOM: stripBOM
};
```

可以看到，以`is`开头的方法都是用于判断是否为某个类型，下面看看几个以`is`开头的判断方法：

```js
// 通过Object.prototype.toString方法来判断复杂数据类型
var toString = Object.prototype.toString;

function isArray(val) {
  return toString.call(val) === '[object Array]';
}

function isUndefined(val) {
  return typeof val === 'undefined';
}

function isNumber(val) {
  return typeof val === 'number';
}

function isFunction(val) {
  return toString.call(val) === '[object Function]';
}

function isDate(val) {
  return toString.call(val) === '[object Date]';
}
```

好明显，对于`js`中的原始数据类型，使用的都是`typeof`方法进行判断，而对于复杂数据类型，如`Array`、`Function`、`Date`等都会采用`Object.prototype.toString`方法。

接下来，我们再来看看其余的几个方法：`forEach`、`merge`、`extend`、`trim`。

```js
function forEach(obj, fn) {
  // Don't bother if no value provided
  if (obj === null || typeof obj === 'undefined') { // 先判断遍历的对象是否为null或undefined
    return;
  }

  // Force an array if not already something iterable
  if (typeof obj !== 'object') { // 若遍历的对象是原始数据类型时，那么就把它包装在一个数组中
    /*eslint no-param-reassign:0*/
    obj = [obj];
  }

  if (isArray(obj)) { // 若遍历的对象是一个数组，那么就通过for循环将遍历到的每一项执行fn方法
    // Iterate over array values
    for (var i = 0, l = obj.length; i < l; i++) {
      fn.call(null, obj[i], i, obj);
    }
  } else { // 若遍历的对象是一个对象（除了数组外），那么就通过for...in循环将该遍历对象中的每一个属性（必须是自身属性，不包括原型属性），逐一调用fn方法
    // Iterate over object keys
    for (var key in obj) {
      if (Object.prototype.hasOwnProperty.call(obj, key)) { // 判断遍历到的属性是否为自身属性，目的是剔除原型属性 
        fn.call(null, obj[key], key, obj);
      }
    }
  }
}
```

可以看到，`forEach`的封装并不是我们对于数组遍历时所使用的`forEach`函数那么简单。它主要有以下几方面内容：

- 遍历的对象不允许是`null`和`undefined`；
- 当遍历的对象是原始数据类型时，那么会使用数组进行包装，即`[原始数据类型]`；
- 若遍历对象是数组，则通过`for`循环将遍历的每一项值都调用`fn`方法；
- 若遍历对象是对象（除了数组），那么通过`for--in`循环并结合`hasOwnProperty`方法将每一项**自身属性**调用`fn`方法；

也许有些朋友对于`for--in`循环结合`hasOwnProperty`产生疑惑，其实`for--in`循环会将对象自身属性以及对象的原型对象上属性都会遍历出来，那么再结合`hasOwnProperty`就能将对象的原型对象上属性进行剔除，只保留对象的自身属性。

再来看下一个方法：`merge`。

```js
function merge(/* obj1, obj2, obj3, ... */) {
  var result = {}; // 最终合并的对象
  function assignValue(val, key) {
    if (isPlainObject(result[key]) && isPlainObject(val)) { // 若传入的属性已经在result中存在，并且还是纯对象时，那么最终得到的值就是两个属性的合并
      result[key] = merge(result[key], val);
    } else if (isPlainObject(val)) { // 若传入的属性不在result中，并且还是纯对象时，那么最终得到的值就是一个空对象和该属性值的合并
      result[key] = merge({}, val);
    } else if (isArray(val)) { // 若传入的属性不在result中，并且还是数组时，那么直接将该数组拷贝到最终属性值中（利用纯函数slice来深拷贝）
      result[key] = val.slice();
    } else { // 对于其他类型，则是直接进行覆盖处理
      result[key] = val;
    }
  }

  for (var i = 0, l = arguments.length; i < l; i++) { // 遍历传入对象参数
    forEach(arguments[i], assignValue); // 将遍历到每一个对象，再将其中每一项属性值都执行assignValue方法
  }
  return result;
}
```

可以看到，`merge`函数可是既兼顾了**深拷贝**的知识，也兼顾了**属性合并**的知识，可谓一举两得。

再来看看`extend`函数：

```js
function bind(fn, thisArg) { // 封装的bind方法
  return function wrap() { // 使用闭包形式实现作用域的绑定
    var args = new Array(arguments.length);
    for (var i = 0; i < args.length; i++) {
      args[i] = arguments[i];
    }
    return fn.apply(thisArg, args);
  };
};

function extend(a, b, thisArg) {
  forEach(b, function assignValue(val, key) { // 遍历参数对象b，并将对象中每一个选项都执行assignValue方法
    if (thisArg && typeof val === 'function') { // 若传入thisArg第三个对象，并且遍历到的值是函数时，那么会将该函数的作用域绑定在thisArg对象上，并且返回一个新的函数
      a[key] = bind(val, thisArg); // bind方法就是用于绑定作用域
    } else {
      a[key] = val; // 对于其他类型都是采用覆盖的形式进行赋值
    }
  });
  return a; // 最后返回a
}
```

`extend`方法从理解上可以说是继承，说白了就是将`b`对象中的方法和属性都继承到`a`对象中，但有一点需要注意的是，对于传入第三个对象`thisArg`并且遍历到`b`对象中的方法时，都会把方法中的作用域绑定到`thisArg`中。

最后只剩一个`trim`方法啦，好简单，就相当于字符串中的`trim`是一样的。

```js
function trim(str) {
  return str.replace(/^\s*/, '').replace(/\s*$/, '');
}
```

类同地，`js`中字符串原声方法`trim`的实现其实也是采用正则的方式实现。

































