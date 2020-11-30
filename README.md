## 介绍

由于最近在`cli`中使用`axios`对请求和响应增加拦截器以及方法的包装，不知不觉间就对`axios`内部实现产生了强烈兴趣，究竟拦截器的实现思想是怎样的？而请求方法的内部封装又是如何的？

为此，才会有这一系列的文章出来跟大家一起分享一起学习。



## 目录

- [常用的工具函数](https://github.com/Andraw-lin/about-Axios/blob/main/docs/%E3%80%90axios%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E3%80%91%E5%B8%B8%E7%94%A8%E7%9A%84%E5%B7%A5%E5%85%B7%E5%87%BD%E6%95%B0.md)
- [从入口分析axios封装](https://github.com/Andraw-lin/about-Axios/blob/main/docs/%E3%80%90axios%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E3%80%91%E4%BB%8E%E5%85%A5%E5%8F%A3%E5%88%86%E6%9E%90axios%E5%B0%81%E8%A3%85.md)
- [axios请求的执行原理](https://github.com/Andraw-lin/about-Axios/blob/main/docs/%E3%80%90axios%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E3%80%91axios%E8%AF%B7%E6%B1%82%E7%9A%84%E6%89%A7%E8%A1%8C%E5%8E%9F%E7%90%86.md)
- [拦截器的构造和实现](https://github.com/Andraw-lin/about-Axios/blob/main/docs/%E3%80%90axios%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E3%80%91%E6%8B%A6%E6%88%AA%E5%99%A8%E7%9A%84%E6%9E%84%E9%80%A0%E5%92%8C%E5%AE%9E%E7%8E%B0.md)
- [请求的实现](https://github.com/Andraw-lin/about-Axios/blob/main/docs/%E3%80%90axios%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E3%80%91%E8%AF%B7%E6%B1%82%E7%9A%84%E5%AE%9E%E7%8E%B0.md)
- [取消请求](https://github.com/Andraw-lin/about-Axios/blob/main/docs/%E3%80%90axios%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E3%80%91%E5%8F%96%E6%B6%88%E8%AF%B7%E6%B1%82.md)
- [请求并发](https://github.com/Andraw-lin/about-Axios/blob/main/docs/%E3%80%90axios%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E3%80%91%E8%AF%B7%E6%B1%82%E5%B9%B6%E5%8F%91.md)
- 未完待续...

## 建议

在浏览本系列文章时，若发现有啥问题或者有更好的建议，可以随时随地提个`issue`，毕竟我还是很愿意聆听各位大神们的意见和见解的。































