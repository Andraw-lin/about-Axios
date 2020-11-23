在前面讲解`axios`请求的执行原理时，不知道你们是否注意到一个地方：

```js
var chain = [dispatchRequest, undefined];
```

里面的`dispatchRequest`数组项究竟是啥？又是做些什么用的呢？

如果你有看过前面的分享话，那你肯定知道它是啥，没错，它就是我们在使用`axios`发送的请求。

既然是使用`axios`发送的请求，那么`dispatchRequest`里肯定封装了请求的实现，接下来我们就来从源码角度来看看它是如何实现一个请求的。



## 源码分析

`dispatchRequest`所处的位置为：`/lib/core/dispatchRequest.js`。

```js
module.exports = function dispatchRequest(config) {
  throwIfCancellationRequested(config); // 针对取消请求，抛出取消请求信息
  config.headers = config.headers || {};
  config.data = transformData( // 转换请求数据参数的格式
    config.data,
    config.headers,
    config.transformRequest
  );
  config.headers = utils.merge( // 合并请求头部
    config.headers.common || {},
    config.headers[config.method] || {},
    config.headers
  );
  utils.forEach( // 暂时还不清楚作者为何要将headers中请求方式删除
    ['delete', 'get', 'head', 'post', 'put', 'patch', 'common'],
    function cleanHeaderConfig(method) {
      delete config.headers[method];
    }
  );

  var adapter = config.adapter || defaults.adapter; // 获取adapter，其中adapter就是真正用于发送请求的封装方法

  return adapter(config).then(function onAdapterResolution(response) { // 发送请求
    throwIfCancellationRequested(config); // 若取消请求则直接抛出取消请求信息

    // Transform response data
    response.data = transformData( // 将响应数据和响应头部的合适进行转换
      response.data,
      response.headers,
      config.transformResponse
    );

    return response; // 最后返回转换好的响应数据
  }, function onAdapterRejection(reason) { // 响应失败时执行
    if (!isCancel(reason)) { // 在响应是没有取消情况下，则返回响应失败的默认信息
      throwIfCancellationRequested(config);

      // Transform response data
      if (reason && reason.response) {
        reason.response.data = transformData(
          reason.response.data,
          reason.response.headers,
          config.transformResponse
        );
      }
    }

    return Promise.reject(reason);
  });
}
```

在`dispatchRequest`方法中，主要做了以下事情：

- 针对取消请求，抛出取消请求的错误信息。（对于取消请求的内容，现在暂时不说太多，会在下一章节继续探讨）
- 对请求数据以及头部信息通过`transformData`进行格式转换。
- 使用`adapter`发送请求，并将响应数据以及头部信息使用`transformData`进行格式转换。

可以看到，在请求前和响应后，都会对数据和头部信息进行`transformData`进行格式转换，现在我们先来`transformData`究竟是如何转换格式的。

在文件`/lib/core/transformData.js`下：

```js
module.exports = function transformData(data, headers, fns) {
  /*eslint no-param-reassign:0*/
  utils.forEach(fns, function transform(fn) {
    data = fn(data, headers);
  });

  return data;
};
```

好明显，`transformData`做的事情并不多 ，仅仅只是遍历`fns`数组，然后将每遍历到的方法，都将当前的数据和头部信息作为参数进行执行。

而在这里，`fns`数组恰恰就是`config.transformRequest`和`config.transformResponse`。我们继续向下探索。下面为方便查看，就将代码抽离出来。

```js
function normalizeHeaderName(headers, normalizedName) { // 遍历头部信息，将normalizedName去重处理，并将新值进行填充
  utils.forEach(headers, function processHeader(value, name) {
    if (name !== normalizedName && name.toUpperCase() === normalizedName.toUpperCase()) { // 通过所有字母大写来判断normalizedName是否重复
      headers[normalizedName] = value;
      delete headers[name]; // 去掉旧的重复normalizedName
    }
  });
};

var defaults = {
  // ...
  transformRequest: [function transformRequest(data, headers) { // 转换请求数据以及头部信息
    normalizeHeaderName(headers, 'Accept'); // 设置Accept头部
    normalizeHeaderName(headers, 'Content-Type'); // 设置Content-Type头部
    if (utils.isFormData(data) ||
      utils.isArrayBuffer(data) ||
      utils.isBuffer(data) ||
      utils.isStream(data) ||
      utils.isFile(data) ||
      utils.isBlob(data)
    ) { // 针对Node环境下，无需将数据进行转换，直接返回该数据
      return data;
    }
    if (utils.isArrayBufferView(data)) {
      return data.buffer;
    }
    if (utils.isURLSearchParams(data)) { // 针对URL上拼接的数据，统一转化成application/x-www-form-urlencoded格式
      setContentTypeIfUnset(headers, 'application/x-www-form-urlencoded;charset=utf-8');
      return data.toString();
    }
    if (utils.isObject(data)) { // 以对象形式传输的数据，统一转化成application/json格式
      setContentTypeIfUnset(headers, 'application/json;charset=utf-8');
      return JSON.stringify(data);
    }
    return data;
  }],
  transformResponse: [function transformResponse(data) { // 转换响应数据以及头部信息
    /*eslint no-param-reassign:0*/
    if (typeof data === 'string') { // 只有后端以JSON字符串返回时，统一解析成对象形式，否则直接返回该数据
      try {
        data = JSON.parse(data);
      } catch (e) { /* Ignore */ }
    }
    return data;
  }],
  // ...
}
```

`transformRequest`主要做了以下内容。

- 增加默认请求头：`Accept`和`Content-Type`。
- 对于Node环境下的`Buffer`、`Stream`等类型，直接不作格式转换。
- 类似`get`请求将请求数据拼接在`URL`中，转换为`application/x-www-form-urlencoded`格式进行传递。
- 而类似`post`请求将数据作为对象形式的，则转换为`JSON`格式（即`application/json`）。

相反，`transformResponse`方法则没有那么细微的处理，仅仅对响应内容是以字符串返回时，则转换为`JSON Object`形式。

接下来，我们再来看看重头戏---`adapter`方法，一个真正封装了请求的方法。

```js
var defaults = {
  // ...
  adapter: getDefaultAdapter(),
  // ...
}

function getDefaultAdapter() { // 默认封装好的请求方法
  var adapter;
  if (typeof XMLHttpRequest !== 'undefined') { // 浏览器环境下，采用XHRHttpRequest对象发送请求
    // For browsers use XHR adapter
    adapter = require('./adapters/xhr');
  } else if (typeof process !== 'undefined' && Object.prototype.toString.call(process) === '[object process]') { // Node环境下，则采用http第三方库来发送请求
    // For node use HTTP adapter
    adapter = require('./adapters/http');
  }
  return adapter;
}
```

可以看到，**`axios`所封装的请求方法其实还是使用原生的方法，只是在原生方法的基础上做了一层语法糖**，方便开发者更好滴处理请求。另外，**针对浏览器环境会采用`XHRHttpRequest`对象处理请求，而对于`Node`环境则会使用第三方库`http`服务来处理请求**。

对于`xhr.js`文件和`http.js`文件的内部封装细节，就暂不拿出来探讨，有兴趣的盆友可以自行去看看别人封装哈🤔，当然后面有时间我还会继续单独拿出来跟大家一起分享一下的。















































