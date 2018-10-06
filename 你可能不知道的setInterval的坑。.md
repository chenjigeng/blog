## 你可能不知道的setInterval的坑

之前印象中一直记得setInterval有一些坑，但是一直不是很清楚那些坑是什么。今天去摸索了下之后，决定来做个记录以免自己忘记，也希望让更多人了解到这个坑。

### 坑的地方

1. setInterval会无视代码的错误。就算遇到了错误，它还是会一直循环下去，不会停止。这就导致了可能你代码里存在着一些问题（比如你的代码可能有个一定概率下会发生的错误，而你使用setinterval来循环调用它，由于setinterval不会因为报错停止，所以这个问题可能被隐藏），可是却很难发现。

   ```javascript
   let count = 1;
   setInterval(function () {
       count++;
       console.log(count);
       if (count % 3 === 0) throw new Error('setInterval报错');
   }, 1000)
   ```

1. setInterval会无视任何情况下定时执行。而在有些场景下，我们是不希望如此的。

   比如说，我们要实现一个功能，每隔一段时间要向服务器发送请求来查看是否有新数据。此时，若当时用户的网络状态很糟糕，客户端收到请求响应的时间大于interval循环的时间。而setInterval会无视任何情况下定时执行，这就会导致了用户的客户端里充斥着ajax请求。
   此时正确的做法应该是改用setTimeout,当用户发出去的请求得到响应或者超时后，再使用setTimeout递归发送下一个请求。这样就不会有setInterval的坑了。

2. setInterval不能确保每次调用都能执行。我们可以先看一个代码

   ```javascript
   const startDate = new Date();
   let endData;
   // 第一个调用会被略过
   setInterval(() => {
     console.log('start');
     console.log(startDate.getTime());
     console.log(endDate.getTime());
     console.log('end');
   }, 1000);
   while (startDate.getTime() + 2 * 1000 > (new Date()).getTime()) {
   }
   endDate = new Date();
   ```

   我们可以看到，第一次执行的setInterval函数输出的startDate和endDate差距在2s以上。而我们的setInterval写的是每间隔1s执行一次。因此，我们可以看出，第一次的setInterval函数调用被略过了。

   这说明了：如果说你的代码执行时间会比较久的话，就会导致setInterval中的一部分函数调用被略过。因此你的程序如果依赖于setInterval的精确执行的话，那么你就要小心这一点了。

   当然，其实setTimeout也有这个问题。浏览器的定时器都不是精确执行的。就算你调用setTimeout(fn, 0)，它也不能确保马上执行。

#### 解决方案

其实解决方案也很简单，就是使用setTimeout，然后再setTimeout里递归调用。

比如说第一个和第二个坑就可以这样写：

```javascript
function fn () {
  setTimeout(() => {
    // 程序主逻辑代码
    // 循环递归调用
    fn();
  }, 1000);
}
fn();
```

可是使用setTimeout后，我们又可能会遇到一个问题，就是计时器的下次触发时间是在当前的触发时间上开始计算的。这对于第二个坑这种情况是合理的，可是有时候我们又希望它能“匀速”地被触发。也就是说，希望计时器的触发时间尽可能在计时器注册时间+周期*delay附近。这个时候，我们就可以用预期下次发生的时间减去当前的时间来得到一个精确的delayTime。

我写了一个简单的函数来实现这一点：一开始调用该函数的时候，会记录当前的计时器注册时间，以及一个用来统计计算器调用次数的变量。之后在每次调用newFn的时候，都会使用预期下次发生的时间减去当前的时间来得到一个精确的delayTime。这样至少可以保证在一些情况下，计时器可以稍微精确的执行。

```javascript
function accurateTimers (fn, expectDelayTime) {
  let init = false;
  let registDate = new Date(); // 计时器注册时间
  let count = 0;  // 计时器调用次数
  function newFn() {
    let delayTime;
    count++;
    if (!init) {
      init = true;
      delayTime = expectDelayTime;
    } else {
      delayTime = expectDelayTime * count + registDate.getTime() - new Date().getTime();
    }
    console.log(delayTime);
    setTimeout(() => {
      fn();
      newFn();
    }, delayTime);
  }
  newFn();
}
accurateTimers(function () {
  let startDate = new Date();
  // 延迟500ms
  while (startDate.getTime() + 500 > (new Date()).getTime()) {
  }
}, 1000);
```

### 结论

以上，就是本次文章的内容。这篇文章只是做一个简单的记录，希望能帮大家了解到setInterval的坑的地方，在实际编程中可以少走点弯路。如果觉得有用的话，欢迎点个赞或者关注哦。谢谢。