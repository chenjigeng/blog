####之前一直很好奇，SPA应用到底是怎么实现的，昨天无意间看到了有一篇介绍的文章，就想着来试一下水(以下根据我的理解所写，可能会让你看的云里雾里，如果想加深了解，最好先了解下window.location.hash是什么东西)

#####其实，SPA的原理就是，一开始将一些必要的页面都加载进来，当你在页面输入别的路由的时候，其实还是待在当前的页面，只不过是他识别出你想要去的地址，然后将那个页面的内容获取到，替代掉当前页面的内容，并且相应的改变url地址，这样给人看起来就好像到了另一个页面，实际上你还是在这个页面里，没有离开过.

#####比如，例如当前你在localhost:8080/index.html这个页面时，你想跳转到#list-view页面(使用hashChange)，或者你点击某个跳转按钮要跳转到那个页面的时候，他先获取你那个#list-view页面的内容，然后将当前页面的内容清除掉，然后再把list-view的内容呈现出来，并没有跳转到别的页面，你从头到尾都是在这个页面里，不过url地址会变化，因此看起来就像你到了另一个页面，这样给人的用户体验特别好，因为不需要等待页面加载过程.

###说了这么多，我们来根据他的原理做一个SPA的小应用吧(里面的html和css代码直接复制了我之前看的那个博客的作者的，因为懒得自己设计)

##html代码如下:

```html
    <!DOCTYPE html>
<html>
  <head>
    <title>SPA</title>
    <link rel="stylesheet" type="text/css" href="index.css" />
    <script type="text/javascript" src="jquery-3.1.1.min.js"></script>
    <script type="text/javascript" src="spa.js"></script>
  </head>
  <body>
    <div class="pageview" style="background: #3b76c0" id="main-view">
      <h3>首页</h3>
      <div title="-list-view" class="right-arrow"></div>
    </div>
    <div class="pageview" style="background: #58c03b;display: none" id="list-view">
      <h3>列表页面</h3>
      <div class="left-arrow"></div>
      <div title="-detail-view" class="right-arrow"></div>
    </div>
    <div class="pageview" style="background: #c03b25;display: none" id="detail-view">
      <h3>列表详情页面</h3>
      <div class="left-arrow"></div>
    </div>
  </body>
</html>
```

####在这里，我们先创建了三个div,第一个main-view的display不设置，其他的两个display设置为node,这样我们一开始进去就只能看到main-view这个页面

###之后，我们可以通过js代码来模拟SPA对路由跳转的处理

```javascript
var states ;
var currentState;
$(document).ready(function() {
  states = registState();
  currentState = init();
  //监听hash路由变化
  window.addEventListener("hashchange", function() {
    var nextState;
    console.log(window.location.hash);
    //判断地址是否为空，若为空，则默认到main-view页面
    if (window.location.hash == "") {
      nextState = "main-view";
    }
    else {
      //若不为空，则获取hash路由信息，得到下一个状态
      nextState = window.location.hash.substring(1);
    }
    //判断当前状态是否注册过(是有存在这个状态)0g
    var validState = checkState(states, nextState);
    //若不存在，则返回当前状态
    if (!validState) {
      console.log("you enter the false validState");
      window.location.hash = "#" + currentState;
      return;
    }
    $('#'+ currentState).css("display", "none");
    $('#'+ nextState).css("display", "block");
    currentState = nextState;
  })

})
//状态注册
function registState() {
  var states = [];
  //状态注册
  $(".pageview").map(function() {
    return states.push($(this)[0].id);
  })
  return states;
}
//初始化，对用户一开始输入的url进行处理
function init() {
  var currentState = window.location.hash.substring(1);
  if (currentState == "") {
    currentState = "main-view";
  }
  if (currentState != "main-view") {
    $('#main-view').css("display", "none");
    $('#'+ currentState).css("display", "block");
  }
  return currentState;
}
//判断状态是否存在
function checkState(states, nextState) {
  var tof = false;
  states.forEach(function(element) {
      if (element == nextState) {
        tof = true;
      }
    })
  return tof;
}
```

###这里，我们首先将每一个div当做一个状态，当用户输入的地址匹配了某个状态之后，就呈现那个状态所代表的页面（每个div的状态名我们设置为他们的id名字）

####代码我觉得还算比较清晰，首先，我们就先注册这三个div的状态(registState)，然后根据用户输入的url地址来初始化页面，返回匹配的那个状态的页面(init)。

####并且，注册一个hashchange事件，这个事件是当用户输入的hash地址变化后触发，我们在里面获取用户的输入地址，然后返回匹配的那个状态的页面，若没有匹配的状态，则返回上一个匹配的状态。以下的截图

###值得一提的是，我里面替换页面的做法是：将当前状态的页面的display设置为none,然后将下一个状态的页面的display设置为block,这样就完成了页面的替换以及路由的变换，而且不会导致路由的变化

###初始页面：
![](http://images2015.cnblogs.com/blog/993343/201610/993343-20161026210146515-709123930.png)

###修改路由地址，修改为file:///C:/Users/chenjg/Desktop/Interest/SPA/index.html#list-view,可以看到页面发送了相应的变化
![](http://images2015.cnblogs.com/blog/993343/201610/993343-20161026210235968-1382545102.png)

###输入错的地址，没有匹配到合适的状态，则恢复到上一个状态：file:///C:/Users/chenjg/Desktop/Interest/SPA/index.html#list-vi
![](http://images2015.cnblogs.com/blog/993343/201610/993343-20161026210618625-426783569.png)

####接下来打算继续试下路由的嵌套，以及动态加载html文件作为路由的模块。