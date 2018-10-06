**本文主要是在我读《高性能Javascript》之后，想要记录下一些有用的优化方案，并且就我本身的一些经验，来大家一起分享下**，

### Javascript的加载与执行
大家都知道，浏览器在解析DOM树的时候，当解析到script标签的时候，会阻塞其他的所有任务，直到该js文件下载、解析执行完成后，才会继续往下执行。因此，这个时候浏览器就会被阻塞在这里，如果将script标签放在head里的话，那么在该js文件加载执行前，用户只能看到空白的页面，这样的用户体验肯定是特别烂。对此，常用的方法有以下：

* 将所有的script标签都放到body最底部，这样可以保证js文件是最后加载并执行的，可以先将页面展现给用户。但是，你首先得清楚，页面的首屏渲染是否依赖于你的部分js文件，如果是的话，则需要将这一部分js文件放到head上。
* 使用defer,比如下面这种写法。使用defer这种写法时，虽然浏览器解析到该标签的时候，也会下载对应的js文件，不过它并不会马上执行，而是会等到DOM解析完后（DomContentLoader之前)才会执行这些js文件。因此，就不会阻塞到浏览器。

```javascript
<script src="test.js" type="text/javascript" defer></script>
```

* 动态加载js文件,通过这种方式，可以在页面加载完成后，再去加载所需要的代码，也可以通过这种方式实现js文件懒加载/按需加载，比如现在比较常见的，就是webpack结合vue-router/react-router实现按需加载，只有访问到具体路由的时候，才加载相应的代码。具体的方法如下：

1.动态的插入script标签来加载脚本,比如通过以下代码

```javascript
  function loadScript(url, callback) {
    const script = document.createElement('script')；
    script.type = 'text/javascript';
    // 处理IE
    if (script.readyState) {
      script.onreadystatechange = function () {
        if (script.readyState === 'loaded' || script.readyState === 'complete') {
          script.onreadystatechange = null;
          callback();
        }
      }
    } else {
      // 处理其他浏览器的情况
      script.onload = function () {
        callback();
      }
    }
    script.src = url;
    document.body.append(script);
  }

  // 动态加载js
  loadScript('file.js', function () {
    console.log('加载完成');
  })
```

2.通过xhr方式加载js文件，不过通过这种方式的话，就可能会面临着跨域的问题。例子如下：

```javascript
  const xhr = new XMLHttpRequest();
  xhr.open('get', 'file.js');
  xhr.onreadystatechange = function () {
    if (xhr.readyState === 4) {
      if (xhr.status >= 200 && xhr.status < 300 || xhr.status === 304>) {
        const script = document.createElement('script');
        script.type = 'text/javascript';
        script.text = xhr.responseText;
        document.body.append(script);
      }
    }
  }
```

3.将多个js文件合并为同一个，并且进行压缩。 原因：目前浏览器大多已经支持并行下载js文件了，但是并发下载还是有一定的数量限制了（基于浏览器，一部分浏览器只能下载4个），并且，每一个js文件都需要建立一次额外的http连接，加载4个25KB的文件比起加载一个100KB的文件消耗的时间要大。因此，我们最好就是将多个js文件合并为同一个，并且进行代码压缩。

### javascript作用域

当一个函数执行的时候，会生成一个执行上下文，这个执行上下文定义了函数执行时的环境。当函数执行完毕后，这个执行上下文就会被销毁。因此，多次调用同一个函数会导致创建多个执行上下文。每隔执行上下文都有自己的作用域链。相信大家应该早就知道了作用域这个东西，对于一个函数而言，其第一个作用域就是它函数内部的变量。在函数执行过程中，每遇到一个变量，都会搜索函数的作用域链找到第一个匹配的变量，首先查找函数内部的变量，之后再沿着作用域链逐层寻找。因此，**若我们要访问最外层的变量（全局变量），则相比直接访问内部的变量而言，会带来比较大的性能损耗**。因此，我们可以**将经常使用的全局变量引用储存在一个局部变量里**。

```javascript
const a = 5;
function outter () {
  const a = 2;
  function inner () {
    const b = 2;
    console.log(b); // 2
    console.log(a); // 2
  }
  inner();
}
```

### 对象的读取

javascript中，主要分为字面量、局部变量、数组元素和对象这四种。访问字面量和局部变量的速度最快，而访问数组元素和对象成员相对较慢。而访问对象成员的时候，就和作用域链一样，是在原型链(prototype)上进行查找。因此，若查找的成员在原型链位置太深，则访问速度越慢。因此，**我们应该尽可能的减少对象成员的查找次数和嵌套深度**。比如以下代码

```javascript
  // 进行两次对象成员查找
  function hasEitherClass(element, className1, className2) {
    return element.className === className1 || element.className === className2;
  }
  // 优化，如果该变量不会改变，则可以使用局部变量保存查找的内容
  function hasEitherClass(element, className1, className2) {
    const currentClassName = element.className;
    return currentClassName === className1 || currentClassName === className2;
  }
```

### DOM操作优化

* 最小化DOM的操作次数，尽可能的用javascript来处理，并且尽可能的使用局部变量储存DOM节点。比如以下的代码：

```javascript
  // 优化前，在每次循环的时候，都要获取id为t的节点，并且设置它的innerHTML
  function innerHTMLLoop () {
    for (let count = 0; count < 15000; count++) {
      document.getElementById('t').innerHTML += 'a';
    }
  }
  // 优化后，
  function innerHTMLLoop () {
    const tNode = document.getElemenById('t');
    const insertHtml = '';
    for (let count = 0; count < 15000; count++) {
      insertHtml += 'a';
    }
    tNode.innerHtml += insertHtml;
  }
```

* 尽可能的减少重排和重绘,重排和重汇可能会代价非常昂贵，因此，为了减少重排重汇的发生次数，我们可以做以下的优化

1.当我们要对Dom的样式进行修改的时候，我们应该尽可能的合并所有的修改并且一次处理，减少重排和重汇的次数。

```javascript
  // 优化前
  const el = document.getElementById('test');
  el.style.borderLeft = '1px';
  el.style.borderRight = '2px';
  el.style.padding = '5px';

  // 优化后,一次性修改样式，这样可以将三次重排减少到一次重排
  const el = document.getElementById('test');
  el.style.cssText += '; border-left: 1px ;border-right: 2px; padding: 5px;'
```

2.当我们要批量修改DOM节点的时候，我们可以将DOM节点隐藏掉，然后进行一系列的修改操作，之后再将其设置为可见，这样就可以最多只进行两次重排。具体的方法如下：

```javascript
  // 未优化前
  const ele = document.getElementById('test');
  // 一系列dom修改操作

  // 优化方案一，将要修改的节点设置为不显示，之后对它进行修改，修改完成后再显示该节点，从而只需要两次重排
  const ele = document.getElementById('test');
  ele.style.display = 'none';
  // 一系列dom修改操作
  ele.style.display = 'block';

  // 优化方案二，首先创建一个文档片段(documentFragment),然后对该片段进行修改，之后将文档片段插入到文档中,只有最后将文档片段插入文档的时候会引起重排，因此只会触发一次重排。。
  const fragment = document.createDocumentFragment();
  const ele = document.getElementById('test');
  // 一系列dom修改操作
  ele.appendChild(fragment);
```

3.**使用事件委托**：事件委托就是将目标节点的事件移到父节点来处理，由于浏览器冒泡的特点，当目标节点触发了该事件的时候，父节点也会触发该事件。因此，由父节点来负责监听和处理该事件。那么，它的优点在哪里呢？假设你有一个列表，里面每一个列表项都需要绑定相同的事件，而这个列表可能会频繁的插入和删除。如果按照平常的方法，你只能给每一个列表项都绑定一个事件处理器，并且，每当插入新的列表项的时候，你也需要为新的列表项注册新的事件处理器。这样的话，如果列表项很大的话，就会导致有特别多的事件处理器，造成极大的性能问题。而通过事件委托，我们只需要在列表项的父节点监听这个事件，由它来统一处理就可以了。这样，对于新增的列表项也不需要做额外的处理。而且事件委托的用法其实也很简单：

```javascript
function handleClick(target) {
  // 点击列表项的处理事件
}
function delegate (e) {
  // 判断目标对象是否为列表项
  if (e.target.nodeName === 'LI') {
    handleClick(e.target);
  }
}
const parent = document.getElementById('parent');
parent.addEventListener('click', delegate);
```