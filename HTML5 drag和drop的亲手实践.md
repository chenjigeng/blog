### 起因
   最近在公司打杂的时候，突然分到了一个锅，就是要支持一个新的功能：用户可以通过拖曳组件来改变组件的顺序。因此，这阵子就看了一下网上的一些drag和drog的文章以及W3C的介绍，然后自己亲手实践了一下，毕竟打码，才能变得更强。
首先，先放一个我的demo，大家可以去那里随便拖动一下玩一玩:
https://chenjigeng.github.io/example/drag.html

### 知识储备

#### 与drag和drog有关的属性和事件
 - draggable属性: 如果你想让一个元素变得可以拖曳的话，那么你就必须设置它的draggable=true，如下
 ```html
 <div class='target' draggable="true"></div>
 ```
 
 这样，该元素就可以拖动了
 - ondragstart: 当元素开始被拖动时，触发该事件，目标对象是被拖动的元素
 - ondragover: 当被拖动元素在悬挂元素上移动的时候,该事件触发。目标对象是被拖动元素悬挂的那个元素。
 - ondragleave: 当被拖动元素离开悬挂元素时，触发该事件。目标对象是被拖动元素悬挂的那个元素。
 - ondrop: 当鼠标松开被拖动元素的时候，触发该事件。目标对象是被拖动元素悬挂的那个元素。
 - ondragend: 当鼠标松开被拖动元素的时候，触发该事件。目标对象是被拖动的元素。其中，ondrop事件会先于ondragend事件触发。
 - event.preventDefault: 当触发ondragover事件的时候，必须使用event.preventDefault(),否则的话，ondrop事件就不会触发
 - event.dataTransfer.effectAllowed:设置或返回被拖动元素允许发生的拖动行为。可设置的属性很多，这里我们就不细说，感兴趣的可以去查下，一般来说，我们都设置为"move".

#### 插入节点的方法

 - 将节点插入到另一个节点前面，代码如下
```js
 function insertBefore(insertNode, node) {
       node.parentNode.insertBefore(insertNode, node)
 }
```
 
 这个其实比较简单，就是找到节点的父亲，然后将要插入的节点放到节点的前面。
 - 将节点插入到另一个节点后面，代码如下图
 
```js
 function insertAfter(insertNode, node) {
	   if (node.nextElementSibling) {
	     insertBefore(insertNode, node.nextElementSibling)
	   } else {
	     node.parentNode.appendChild(insertNode)
	   }
 }
```
 这个其实也挺简单的，就是如果该节点有兄弟节点的话，那么就将插入节点放到它兄弟节点的前面，否则，则说明该节点是父节点的最后一个节点，因此直接将插入节点放到父节点的末尾。

### 实践
在这里，我们要做的就是一个支持各个图片拖曳来交换位置的玩意，不过，当图片交换位置的时候，不单单是图片交换位置，而是包含图片的容器交换位置。

1.我们先放置几张图片，并且将它们的dragable设置为true,这样它们就可以拖动了。代码如下:
```html
<body>
    <div class='target' draggable="true">
      <img src="./imgs/1.jpeg" alt="1">
    </div>
    <div class='target' draggable="true">
      <img src='./imgs/2.jpg' />
    </div>
    <div class='target' draggable="true">
      <img src="./imgs/3.jpg" alt="ss">
    </div>    
    <div class='target' draggable="true">
      <img src="./imgs/4.jpg" alt="ss">
    </div>   
  </body>
```
效果:
![这里写图片描述](http://img.blog.csdn.net/20170715002230778?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZ2l0aHViXzM5MTMzMTky/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
2.为每个div都设置一个ondragstart函数，当该函数触发的时候，进行初始化操作，比如记录当前的目标对象，拖动目标的y值，以及设置拖动的效果。
```js
// 拖动的目标对象
let target = ''
// 拖动的目标对象的y值
let targetOffsetTop = 0
// 当元素开始被拖动时，触发该事件，目标对象是被拖动的元素
function handleDragStart(ev) {
  target = findTarget(ev.target)
  targetOffsetTop = ev.target.offsetTop
  ev.dataTransfer.effectAllowed = 'move'
}
// 找到类名为target的目标对象
function findTarget(node) {
  if (!node || node == document) {
    return null
  }
  if (node.classList.contains('target')) {
    return node;
  }
  return findTarget(node.parentNode)
}
```
3.为每个div注册一个ondragover事件和ondragleave事件，在ondragover事件里，主要是调用event.preventDefault来防止ondrog不会被触发，并且为了看起来更明显，当ondragover事件触发的时候，为目标对象增加一个dotted类。当ondragleave事件触发的时候，则把dotted类从目标对象移除。
```js
// 当被拖动元素在悬挂元素上移动的时候,该事件触发。目标对象是被拖动元素悬挂的那个元素。
// 必须执行event.preventDefault()，不然的话ondrop不会触发
function handleDragOver(ev) {
  ev.preventDefault();
  ev.target.classList.add('dotted')
}
// 当被拖动元素离开悬挂元素时，触发该事件。目标对象是被拖动元素悬挂的那个元素。
function handleDragLeave(ev) {
  ev.target.classList.remove('dotted')
}
```
4.为每个div注册ondrog事件和ondragend事件，ondrog事件是重点，它主要是根据被拖动元素和被拖动元素悬挂的那个元素的坐标，来决定是要将被拖动元素插入到悬挂元素的前面还是后面。而ondragend主要是用于将target设置为null，代码如下:
```js
// 当鼠标松开被拖动元素的时候，触发该事件。目标对象是被拖动元素悬挂的那个元素。
function handleDrog(ev) {
  let resultOffsetTop = ev.target.offsetTop
  if (targetOffsetTop < resultOffsetTop) {
    insertAfter(target, findTarget(ev.target))
  }
  else {
    insertBefore(target, findTarget(ev.target))
  }
  ev.target.classList.remove('dotted')
}
// 将节点插入到另一个节点前面
function insertBefore(insertNode, node) {
  node.parentNode.insertBefore(insertNode, node)
}
// 将节点插入到另一个节点后面
function insertAfter(insertNode, node) {
  if (node.nextElementSibling) {
    insertBefore(insertNode, node.nextElementSibling)
  } else {
    node.parentNode.appendChild(insertNode)
  }
}
// 当松开鼠标的时候，触发该事件。目标对象是被拖动的对象
function handleDragEnd(ev) {
  target = null
}
```

这样子，我们就实现了一个可以通过拖曳来改变图片顺序的一个小玩意啦~完整的代码放到https://github.com/chenjigeng/something 上了~有兴趣的可以git clone下来跑一跑