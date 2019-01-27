async await

浏览器遇到await 语句后，会先去执行外部的同步代码，之后再来处理await后面的函数的返回值。

注意：async函数里面是从右往左执行的，因此会先执行右边的函数，之后再遇到await做处理。

```javascript
async function async1() {
    console.log("async1 start");
    await async2();
    console.log("async1 end");
}

async function async2() {
    console.log("async2");
}

console.log("script start");

setTimeout(function() {
    console.log("setTimeout");
}, 0);

async1();

new Promise(function(resolve) {
    console.log("promise1");
    resolve();
}).then(function() {
    console.log("promise2");
});

console.log("script end");
```

先执行了async2函数，得到结果，遇到了await,则先去执行外部的同步代码，因此new Promise.then先加入micro task。之后，在处理async2得到的结果，await async2()，等同于Promise.then，因此，再将async2加入micro task.