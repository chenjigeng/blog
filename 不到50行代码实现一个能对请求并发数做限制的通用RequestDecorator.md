### 使用场景

在开发中，我们可能会遇到一些对异步请求数做并发量限制的场景，比如说微信小程序的request并发最多为5个，又或者我们需要做一些批量处理的工作，可是我们又不想同时对服务器发出太多请求（可能会对服务器造成比较大的压力）。这个时候我们就可以对请求并发数进行限制，并且使用排队机制让请求有序的发送出去。

### 介绍

那么，接下来我们就来讲一下如何实现一个通用的能对请求并发数进行限制的RequestDecorator。我们先来介绍一下它的功能：

1. 既然涉及到并发数限制，它就肯定允许用户传入最大并发数限制参数:maxLimit
2. 既然是一个通用的RequestDecorator,那么它应该允许使用者传入其喜欢的异步api(比如ajax, fetch, axios等)。
3. 为了方便起见，也为了开发便利性，被RequestDecorator封装后的request请求结果都返回一个promise。
4. 由于使用者传入的异步api不一定是promise类型的，也可能是callback类型的，因此我们提供用户一个needChange2Promise参数，使用者若传入的是callback类型的api，它可以通过将这个参数设置为true来将callback类型转化为promise类型。

分析完功能后，接下来我们就来实现这个东西：

### 实现

具体代码如下，每一步我基本都做了注释，相信大家能看懂。

```javascript
const pify = require('pify');

class RequestDecorator {
  constructor ({
    maxLimit = 5,
    requestApi,
    needChange2Promise,
  }) {
    // 最大并发量
    this.maxLimit = maxLimit;
    // 请求队列,若当前请求并发量已经超过maxLimit,则将该请求加入到请求队列中
    this.requestQueue = [];
    // 当前并发量数目
    this.currentConcurrent = 0;
    // 使用者定义的请求api，若用户传入needChange2Promise为true,则将用户的callback类api使用pify这个库将其转化为promise类的。
    this.requestApi = needChange2Promise ? pify(requestApi) : requestApi;
  }
  // 发起请求api
  async request(...args) {
    // 若当前请求数并发量超过最大并发量限制，则将其阻断在这里。
    // startBlocking会返回一个promise，并将该promise的resolve函数放在this.requestQueue队列里。这样的话，除非这个promise被resolve,否则不会继续向下执行。
    // 当之前发出的请求结果回来/请求失败的时候，则将当前并发量-1,并且调用this.next函数执行队列中的请求
    // 当调用next函数的时候，会从this.requestQueue队列里取出队首的resolve函数并且执行。这样，对应的请求则可以继续向下执行。
    if (this.currentConcurrent >= this.maxLimit) {
      await this.startBlocking();
    }
    try {
      this.currentConcurrent++;
      const result = await this.requestApi(...args);
      return Promise.resolve(result);
    } catch (err) {
      return Promise.reject(err);
    } finally {
      console.log('当前并发数:', this.currentConcurrent);
      this.currentConcurrent--;
      this.next();
    }
  }
  // 新建一个promise,并且将该reolsve函数放入到requestQueue队列里。
  // 当调用next函数的时候，会从队列里取出一个resolve函数并执行。
  startBlocking() {
    let _resolve;
    let promise2 = new Promise((resolve, reject) => _resolve = resolve);
    this.requestQueue.push(_resolve);
    return promise2;
  }
  // 从请求队列里取出队首的resolve并执行。
  next() {
    if (this.requestQueue.length <= 0) return;
    const _resolve = this.requestQueue.shift();
    _resolve();
  }
}

module.exports = RequestDecorator;
```

样例代码如下：

```javascript
const RequestDecorator = require('../src/index.js')

// 一个callback类型的请求api
function delay(num, time, cb) {
  setTimeout(() => {
    cb(null, num);
  }, time);
}

// 通过maxLimit设置并发量限制，needChange2Promise将callback类型的请求api转化为promise类型的。
const requestInstance = new RequestDecorator({
  maxLimit: 5,
  requestApi: delay,
  needChange2Promise: true,
});


let promises = [];
for (let i = 0; i < 30; i++) {
  // 接下来你就可以像原来使用你的api那样使用它,参数和原来的是一样的
  promises.push(requestInstance.request(i, Math.random() * 3000).then(result => console.log('result', result), error => console.log(error)));
}
async function test() {
  await Promise.all(promises);
}

test();
```

这样，一个能对请求并发数做限制的通用RequestDecorator就已经实现了。当然，这里还有很多可以继续增加的功能点，比如

1. 允许使用者设置每个请求的retry次数。
2. 允许使用者对每个请求设置缓存处理。

**优点：**

1. 不修改用户原来的request api代码。对原有代码无副作用。
2. 不修改request api的调用方式。用户可以无缝的使用被RequestDecorator封装过的request。
3. 可扩展，后续可能不止支持并发量限制，还可能增加缓存、retry等额外的功能。

### 结语

以上，就是本篇的全部内容。[github仓库地址点击这里](https://github.com/chenjigeng/requestDecorator)。欢迎大家点赞或者star下。如果大家有兴趣的话，也可以一起来完善这个东西。这个项目还不成熟，可能还会有bug，欢迎大家在github上提issue帮助我完善它。如果觉得有帮助的话，麻烦点个赞哦，谢谢。