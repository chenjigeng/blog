​let和const是es6新出的两种变量声明的方式，接下来我来分别针对这两个，聊一聊。

## let

​let它的出现，我认为主要是解决了块级作用域的需求。因为js以前本身是没有什么块级作用域的概念的(顶多就算上一个函数作用域),因此这也导致了很多变量污染的问题，很多时候由于你没有处理好作用域的影响，导致了奇怪的问题。因此我们一般都采取函数作用域的方式来防止变量的污染。不过既然有了let的出现，我们就可以很方便的解决这个问题.

#### 块级作用域

```javascript
for (var i = 0; i < 5; i++) {
	console.log(i)
}
console.log(i) // 5
for (let j = 0; j < 5; j++) {
	console.log(j)
}
console.log(j) // error: j is not defined
```

如上所示，如果我们在循环内部使用var声明一个变量的话，当循环结束后，该变量并没有被回收，而当我们使用let的时候，当离开这个块的时候，该变量就会回收。

#### 暂时性死区

```js
var i = 5;
(function hh() {
  console.log(i) // undefined
  var i = 10
})()

let j = 55;
(function hhh() {
  console.log(j) // ReferenceError: j is not defined
  let j = 77
})()
```

看以上代码，由于var它具有变量提升的功能，所以该声明语句会移到最上面执行，也就是等价于以下代码:

```js
var i = 5;
(function hh() {
  var i
  console.log(i) // undefined
  i = 10
})()

let j = 55;
(function hhh() {
  console.log(j) // ReferenceError: j is not defined
  let j = 77
})()
```

但是，如果将var换成let的话却会报错，如果说，let是没有变量提升的话，那么应该是直接输出55，而不应该报错啊。

其实，这个特性叫做临时性死区，也可以把它当成变量提升的一种特殊情况.也就是说，当你在一个块里面，利用let声明一个变量的时候，在块的开始部分到该变量的声明语句之间，我们称之为临时性死区，你不可以在这个区域内使用该变量，直到遇到其let语句为止。比如:

```js
var i = 5;   // j的临时性死区
(function hh() { 
  var i
  console.log(i) // undefined
  i = 10
})() // j的临时性死区
// j的临时性死区
let j = 55; // 接下来可以愉快的使用let了
console.log(j)
console.log(j+10)
(function hhh() {
  console.log(j) // 新的j的临时性死区
  let j = 77 //又有一个声明语句，从这个函数的开始部分到这里，都是新的j的临时性死区
})()
```

之所以说它是变量提升的一种特殊情况，是因为无论你在块的哪一个地方利用let声明了一个变量，都会产生一个从块的开始部分到该变量声明语句的临时性死区.

#### 比较安全可靠:对var或者是直接声明全局变量来说，变量都可以未声明或者在声明语句之前就使用，而使用了let之后，该变量必须在其声明语句后，才能使用，否则就会报错。这就在一定程度上避免了变量滥用的情况。

## const

const，顾名思义，就是声明一个常量，但是，真的是这样吗？

#### 对基本类型而言

对于基本的类型而言的话，比如number,string,boolean等来说，确实它就是声明一个不会变的常量，只要你修改了它，就会报错

```
const a = 1
a = 2 // Uncaught TypeError: Assignment to constant variable.
const b = '1231'
b = 'xcv' // Uncaught TypeError: Assignment to constant variable.
const c = true
c = false // Uncaught TypeError: Assignment to constant variable.
```

#### 对引用类型而言

不过，对于引用类型而言的话，它指的并不会对象的内容不变，而是对象的地址不变。也就是说，你可以修改对象的内部成员，但是你不可以修改该变量的地址。

```javascript
  const obj = {
    name: 'cjg'
  }
  obj.school = 'sysu'
  console.log(obj) // Object {name: "cjg", school: "sysu"}
  obj = {} // VM183:6 Uncaught TypeError: Assignment to constant variabl
```

其实，就我个人理解，const无论是作用于基本类型还是引用类型，它都是为了保证变量的地址不发生改变(因为你对基本类型而言，你给它赋一个新值,其实也就意味着修改了该变量的地址)