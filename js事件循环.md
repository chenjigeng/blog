**之前有看过一些事件循环的博客，不过一阵子没看就发现自己忘光了，所以决定来自己写一个博客总结下！**
#### **首先，我们来解释下事件循环是个什么东西:**
就我们所知，浏览器的js是单线程的，也就是说，在同一时刻，最多也只有一个代码段在执行，可是浏览器又能很好的处理异步请求，那么到底是为什么呢？我们先来看一张图(这张图来自于http://www.zcfy.cc/article/node-js-at-scale-understanding-the-node-js-event-loop-risingstack-1652.html) 
![js事件循环](http://img.blog.csdn.net/20170613001549300?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZ2l0aHViXzM5MTMzMTky/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
从上图我们可以看出，js主线程它是有一个执行栈的，所有的js代码都会在执行栈里运行。在执行代码过程中，如果遇到一些异步代码(比如setTimeout,ajax,promise.then以及用户点击等操作),那么浏览器就会将这些代码放到一个线程(在这里我们叫做幕后线程)中去等待，不阻塞主线程的执行，主线程继续执行栈中剩余的代码，当幕后线程（background thread）里的代码准备好了(比如setTimeout时间到了，ajax请求得到响应),该线程就会将它的回调函数放到**任务队列**中等待执行。而当主线程执行完栈中的所有代码后，它就会检查**任务队列**是否有任务要执行，如果有任务要执行的话，那么就将该任务放到执行栈中执行。如果当前**任务队列**为空的话，它就会一直循环等待任务到来。因此，这叫做事件循环。

#### **那么，问题来了。如果任务队列中，有很多个任务的话，那么要先执行哪一个任务呢？**
其实(正如上图所示)，js是有两个任务队列的，一个叫做Macrotask Queue(Task Queue),一个叫做Microtask Queue

 - 前者主要是进行一些比较大型的工作，常见的有setTimeout,setInterval,用户交互操作，UI渲染等
 - 后者主要是进行一些比较小型的工作，常见的有Promise,process.nextTick(nodejs)

那么，两者有什么具体的区别呢?或者说，如果两种任务同时出现的话，应该选择哪一个呢?
其实事件循环做的事情如下:
1. 检查Macrotask 队列是否为空,若不为空，则进行下一步，若为空，则跳到3
2. 从Macrotask队列中取队首(在队列时间最长)的任务进去执行栈中执行(仅仅一个)，执行完后进入下一步
3. 检查Microtask队列是否为空，若不为空，则进入下一步，否则，跳到1（开始新的事件循环）
4. 从Microtask队列中取队首(在队列时间最长)的任务进去事件队列执行,执行完后，跳到3

其中，在执行代码过程中新增的microtask任务会在当前事件循环周期内执行，而新增的macrotask任务只能等到下一个事件循环才能执行了(一个事件循环只执行一个macrotask)
首先，我们先来看一段代码

```
console.log(1)
setTimeout(function() {
  //settimeout1
  console.log(2)
}, 0);
const intervalId = setInterval(function() {
  //setinterval1
  console.log(3)
}, 0)
setTimeout(function() {
  //settimeout2
  console.log(10)
  new Promise(function(resolve) {
    //promise1
    console.log(11)
    resolve()
  })
  .then(function() {
    console.log(12)
  })
  .then(function() {
    console.log(13)
    clearInterval(intervalId)
  })
}, 0);

//promise2
Promise.resolve()
  .then(function() {
    console.log(7)
  })
  .then(function() {
    console.log(8)
  })
console.log(9)
```

你觉得结果应该是什么呢?
我在node环境和chrome控制台输出的结果如下:

```
1
9
7
8
2
3
10
11
12
13
```

在上面的例子中
**第一次事件循环:**
1. console.log(1)被执行，输出1
2. settimeout1执行，加入macrotask队列
3. setinterval1执行，加入macrotask队列
4. settimeout2执行，加入macrotask队列
5. promise2执行，它的两个then函数加入microtask队列
6. console.log(9)执行，输出9
7. 根据事件循环的定义，接下来会执行新增的microtask任务，按照进入队列的顺序，执行console.log(7)和console.log(8),输出7和8
microtask队列为空，回到第一步，进入下一个事件循环，此时macrotask队列为: settimeout1,setinterval1,settimeout2

**第二次事件循环:**
1. 从macrotask队列里取位于队首的任务(settimeout1)并执行，输出2
microtask队列为空，回到第一步，进入下一个事件循环，此时macrotask队列为: setinterval1,settimeout2

**第三次事件循环:**
1. 从macrotask队列里取位于队首的任务(setinterval1)并执行，输出3,然后又将新生成的setinterval1加入macrotask队列
microtask队列为空，回到第一步，进入下一个事件循环，此时macrotask队列为: settimeout2,setinterval1

**第四次事件循环:**
1. 从macrotask队列里取位于队首的任务(settimeout2)并执行,输出10，并且执行new Promise内的函数(new Promise内的函数是同步操作，并不是异步操作),输出11，并且将它的两个then函数加入microtask队列
2. 从microtask队列中，取队首的任务执行，直到为空为止。因此，两个新增的microtask任务按顺序执行，输出12和13，并且将setinterval1清空
此时，microtask队列和macrotask队列都为空，浏览器会一直检查队列是否为空，等待新的任务加入队列。

在这里，大家可以会想，在第一次循环中，为什么不是macrotask先执行？因为按照流程的话，不应该是先检查macrotask队列是否为空，再检查microtask队列吗？
原因：因为一开始js主线程中跑的任务就是macrotask任务，而根据事件循环的流程，一次事件循环只会执行一个macrotask任务，因此，执行完主线程的代码后，它就去从microtask队列里取队首任务来执行。

####**注意：**
由于在执行microtask任务的时候，只有当microtask队列为空的时候，它才会进入下一个事件循环，因此，如果它源源不断地产生新的microtask任务，就会导致主线程一直在执行microtask任务，而没有办法执行macrotask任务，这样我们就无法进行UI渲染/IO操作/ajax请求了，因此，我们应该避免这种情况发生。在nodejs里的process.nexttick里，就可以设置最大的调用次数，以此来防止阻塞主线程。

以此，我们来引入一个新的问题，定时器的问题。定时器是否是真实可靠的呢？比如我执行一个命令:setTimeout(task, 100),他是否就能准确的在100毫秒后执行呢？其实根据以上的讨论，我们就可以得知，这是不可能的。
原因我想大家应该也都知道了，因为你执行setTimeout(task,100)后，其实只是确保这个任务，会在100毫秒后进入macrotask队列，但并不意味着他能立刻运行，可能当前主线程正在进行一个耗时的操作，也可能目前microtask队列有很多个任务，所以这也可能是大家一直诟病setTimeout的原因吧哈哈哈哈

以上，只是我个人对事件循环的一些看法， 以及借鉴了其他优秀文章

参考:
http://www.zcfy.cc/article/node-js-at-scale-understanding-the-node-js-event-loop-risingstack-1652.html
https://github.com/ccforward/cc/issues/47