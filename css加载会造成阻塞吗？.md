之前面试今日头条的时候，今日头条面试官问我，js执行会阻塞DOM树的解析和渲染，那么css加载会阻塞DOM树的解析和渲染吗？所以，接下来我就来对css加载对DOM树的解析和渲染做一个测试。

为了完成本次测试，先来科普一下，如何利用chrome来设置下载速度

1. 打开chrome控制台(按下F12),可以看到下图，重点在我画红圈的地方
![image](https://user-gold-cdn.xitu.io/2018/8/31/1658ea2522e04bdb?w=721&h=266&f=png&s=32211)
2. 点击我画红圈的地方(No throttling),会看到下图,我们选择GPRS这个选项
![image](https://user-gold-cdn.xitu.io/2018/8/31/1658ea2522dd691c?w=869&h=426&f=png&s=59985)
3. 这样，我们对资源的下载速度上限就会被限制成20kb/s，好，那接下来就进入我们的正题

# css加载会阻塞DOM树的解析渲染吗？

用代码说话：

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <title>css阻塞</title>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <style>
      h1 {
        color: red !important
      }
    </style>
    <script>
      function h () {
        console.log(document.querySelectorAll('h1'))
      }
      setTimeout(h, 0)
    </script>
    <link href="https://cdn.bootcss.com/bootstrap/4.0.0-alpha.6/css/bootstrap.css" rel="stylesheet">
  </head>
  <body>
    <h1>这是红色的</h1>
  </body>
</html>
```

假设： css加载会阻塞DOM树解析和渲染

假设结果:  在bootstrap.css还没加载完之前，下面的内容不会被解析渲染，那么我们一开始看到的应该是白屏，h1不会显示出来。并且此时console.log的结果应该是一个空数组。

实际结果:如下图
![image](https://user-gold-cdn.xitu.io/2018/8/31/1658ea252321cb0f?w=1920&h=985&f=gif&s=407802)

#### css会阻塞DOM树解析？

由上图我们可以看到，当css还没加载完成的时候，h1并没有显示，但是此时控制台输出如下
![image](https://user-gold-cdn.xitu.io/2018/8/31/1658ea2522f2439c?w=635&h=72&f=png&s=2308)

可以得知，此时DOM树至少已经解析完成到了h1那里，而此时css还没加载完成，也就说明，css并不会阻塞DOM树的解析。
#### css加载会阻塞DOM树渲染？

由上图，我们也可以看到，当css还没加载出来的时候，页面显示白屏，直到css加载完成之后，红色字体才显示出来，也就是说，下面的内容虽然解析了，但是并没有被渲染出来。所以，css加载会阻塞DOM树渲染。


#### 个人对这种机制的评价

 其实我觉得，这可能也是浏览器的一种优化机制。因为你加载css的时候，可能会修改下面DOM节点的样式，如果css加载不阻塞DOM树渲染的话，那么当css加载完之后，DOM树可能又得重新重绘或者回流了，这就造成了一些没有必要的损耗。所以我干脆就先把DOM树的结构先解析完，把可以做的工作做完，然后等你css加载完之后，在根据最终的样式来渲染DOM树，这种做法性能方面确实会比较好一点。

# css加载会阻塞js运行吗？

​	由上面的推论，我们可以得出，css加载不会阻塞DOM树解析，但是会阻塞DOM树渲染。那么，css加载会不会阻塞js执行呢?

同样，通过代码来验证.

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <title>css阻塞</title>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <script>
      console.log('before css')
      var startDate = new Date()
    </script>
    <link href="https://cdn.bootcss.com/bootstrap/4.0.0-alpha.6/css/bootstrap.css" rel="stylesheet">
  </head>
  <body>
    <h1>这是红色的</h1>
    <script>
      var endDate = new Date()
      console.log('after css')
      console.log('经过了' + (endDate -startDate) + 'ms')
    </script>
  </body>
</html>
```

假设: css加载会阻塞后面的js运行

预期结果: 在link后面的js代码，应该要在css加载完成后才会运行

实际结果:
![image](https://user-gold-cdn.xitu.io/2018/8/31/1658ea252305ff0f?w=1920&h=985&f=gif&s=420127)

由上图我们可以看出，位于css加载语句前的那个js代码先执行了，但是位于css加载语句后面的代码迟迟没有执行，直到css加载完成后，它才执行。这也就说明了，css加载会阻塞后面的js语句的执行。详细结果看下图(css加载用了5600+ms):

![image](https://user-gold-cdn.xitu.io/2018/8/31/1658ea2522ff363d?w=705&h=152&f=png&s=3484)

## 结论

由上所述，我们可以得出以下结论:

1. css加载不会阻塞DOM树的解析
2. css加载会阻塞DOM树的渲染
3. css加载会阻塞后面js语句的执行、

因此，为了避免让用户看到长时间的白屏时间，我们应该尽可能的提高css加载速度，比如可以使用以下几种方法:

1. 使用CDN(因为CDN会根据你的网络状况，替你挑选最近的一个具有缓存内容的节点为你提供资源，因此可以减少加载时间)
2. 对css进行压缩(可以用很多打包工具，比如webpack,gulp等，也可以通过开启gzip压缩)
3. 合理的使用缓存(设置cache-control,expires,以及E-tag都是不错的，不过要注意一个问题，就是文件更新后，你要避免缓存而带来的影响。其中一个解决防范是在文件名字后面加一个版本号)
4. 减少http请求数，将多个css文件合并，或者是干脆直接写成内联样式(内联样式的一个缺点就是不能缓存)

## 更新

### 原理解析

那么为什么会出现上面的现象呢？我们从浏览器的渲染过程来解析下。

不用浏览器使用的内核不同，所以他们的渲染过程也是不一样的。目前主要有两个：

**webkit渲染过程**

![webkit渲染流程](https://user-gold-cdn.xitu.io/2018/9/3/1659db14e773f9cc?w=624&h=289&f=png&s=41057)

**Gecko渲染过程**

![Gecko渲染过程](https://user-gold-cdn.xitu.io/2018/9/3/1659db14e7df8a8f?w=624&h=290&f=png&s=102808)

从上面两个流程图我们可以看出来，浏览器渲染的流程如下：

1. HTML解析文件，生成DOM Tree，解析CSS文件生成CSSOM Tree
2. 将Dom Tree和CSSOM Tree结合，生成Render Tree(渲染树)
3. 根据Render Tree渲染绘制，将像素渲染到屏幕上。

从流程我们可以看出来

1. DOM解析和CSS解析是两个并行的进程，所以这也解释了为什么CSS加载不会阻塞DOM的解析。
2. 然而，由于Render Tree是依赖于DOM Tree和CSSOM Tree的，所以他必须等待到CSSOM Tree构建完成，也就是CSS资源加载完成(或者CSS资源加载失败)后，才能开始渲染。因此，CSS加载是会阻塞Dom的渲染的。
3. 由于js可能会操作之前的Dom节点和css样式，因此浏览器会维持html中css和js的顺序。因此，样式表会在后面的js执行前先加载执行完毕。所以css会阻塞后面js的执行。 

## 补充

### DOMContentLoaded

对于浏览器来说，页面加载主要有两个事件，一个是DOMContentLoaded，另一个是onLoad。而onLoad没什么好说的，就是等待页面的所有资源都加载完成才会触发，这些资源包括css、js、图片视频等。

而DOMContentLoaded，顾名思义，就是当页面的内容解析完成后，则触发该事件。那么，正如我们上面讨论过的，css会阻塞Dom渲染和js执行，而js会阻塞Dom解析。那么我们可以做出这样的假设

1. 当页面只存在css，或者js都在css前面，那么DomContentLoaded不需要等到css加载完毕。
2. 当页面里同时存在css和js，并且js在css后面的时候，DomContentLoaded必须等到css和js都加载完毕才触发。

我们先对第一种情况做测试：

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <title>css阻塞</title>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <script>
      document.addEventListener('DOMContentLoaded', function() {
        console.log('DOMContentLoaded');
      })
    </script>
    <link href="https://cdn.bootcss.com/bootstrap/4.0.0-alpha.6/css/bootstrap.css" rel="stylesheet">
  </head>
  <body>
  </body>
</html>
```

实验结果如下图：
![](https://user-gold-cdn.xitu.io/2018/9/7/165b4131f29bfa2e?w=600&h=337&f=gif&s=15611845)

从动图我们可以看出来，css还未加载完，就已经触发了DOMContentLoaded事件了。因为css后面没有任何js代码。

接下来我们对第二种情况做测试，很简单，就在css后面加一行代码就行了

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <title>css阻塞</title>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <script>
      document.addEventListener('DOMContentLoaded', function() {
        console.log('DOMContentLoaded');
      })
    </script>
    <link href="https://cdn.bootcss.com/bootstrap/4.0.0-alpha.6/css/bootstrap.css" rel="stylesheet">

    <script>
      console.log('到我了没');
    </script>
  </head>
  <body>
  </body>
</html>
```

实验结果如下图：

![](https://user-gold-cdn.xitu.io/2018/9/7/165b41371c61520c?w=599&h=344&f=gif&s=8893674)
我们可以看到，只有在css加载完成后，才会触发DOMContentLoaded事件。因此，我们可以得出结论：

1. 如果页面中同时存在css和js，并且存在js在css后面，则DOMContentLoaded事件会在css加载完后才执行。
2. 其他情况下，DOMContentLoaded都不会等待css加载，并且DOMContentLoaded事件也不会等待图片、视频等其他资源加载。 


以上，就是所有内容。觉得还不错的点个推荐呗hhhh,欢迎交流