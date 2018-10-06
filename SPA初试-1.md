本篇内容是在上一次的基础上进行改进，对状态的定义进行了修改，一个状态的定义如下:
```javascript
function state(stateName, template, templateUrl) {
  this.stateName = stateName;
  if (template) {
    this.template = template;
  }
  if (templateUrl) {
    this.templateUrl = templateUrl;
  }
}
```
即每一个页面对应着一个状态，一个状态有一个状态名，还有一个模板/模板url，这样我们就可以将不同页面的内容写到不同的html里，然后通过templateUrl将他们动态加载进来渲染页面。

先贴上js代码：
```javascript
var states = [];
var currentState;
$(document).ready(function() {
  registState();
  console.log(states);
  currentState = init();
  //监听hash路由变化
  window.addEventListener("hashchange", hashChange)
})

//哈希路由處理事件
function hashChange() {
  var nextState;
  console.log(window.location.hash);
  //判断地址是否为空，若为空，则默认到main-view页面
  if (window.location.hash == "") {
    nextState = "mainView";
  } else {
    //若不为空，则获取hash路由信息，得到下一个状态
    nextState = window.location.hash.substring(1);
  }
  //判断当前状态是否注册过(是有存在这个状态)0g
  var validState = checkState(states, nextState);
  //若不存在，则返回当前状态
  if (!validState) {
    createState(nextState, "", "./test.html");
    currentState = nextState;
    return;
  }
  $('#' + currentState).remove();
  if (!nextState.view) {
    states.forEach( function(element, index) {
      if (element.stateName == nextState) {
        createView(element);
      }
    });
  }
  currentState = nextState;
}

//状态注册
function registState() {

  var newState = new state("mainView", "", "./main-view.html");
  var newState1 = new state("listView", "", "./list-view.html");
  var newState2 = new state("detailView", "", "./detail-view.html");

  states.push(newState);
  states.push(newState1);
  states.push(newState2);
}

//初始化，对用户一开始输入的url进行处理
function init() {
  nextState = window.location.hash.substring(1);
  //若用户输入的hash值为空，则默认跳转到states[0]的页面
  if (nextState == "") {
    createView(states[0]);
    return states[0].stateName;
  }
  states.forEach( function(element, index) {
      if (element.stateName == nextState) {
        createView(element);
      }
    });
  return nextState;
}

//判断状态是否存在
function checkState(states, nextState) {
  var tof = false;
  states.forEach(function(element) {
      if (element.stateName == nextState) {
        tof = true;
      }
    })
  return tof;
}

//创建一个状态
function createState(stateName, template, templateUrl) {
  var newState = new state(stateName, template, templateUrl);
  createView(newState);
  $("#" + newState.stateName).css("display", "block");
  $('#'+ currentState).css("display", "none");
  currentState = newState;
  states.push(newState);
}

//创建状态所对应的视图，并将视图放到body里
function createView(state) {
  if (state.template) {
    template = state.template;
    state.view = $("<div id='" + state.stateName + "'></div>").html(template);
    $("body").append(state.view);
  }  else if (state.templateUrl) {
    htmlobj = $.ajax({url: state.templateUrl, async: false});
    template = htmlobj.responseText;
    state.view = $("<div id='" + state.stateName + "'></div>").html(template);
    $("body").append(state.view);
  }
}

//状态对象
function state(stateName, template, templateUrl) {
  this.stateName = stateName;
  if (template) {
    this.template = template;
  }
  if (templateUrl) {
    this.templateUrl = templateUrl;
  }
}
```

###处理逻辑
1. 一开始进入页面的时候，先利用registState()注册一些状态，然后利用init()函数来对用户一开始输入的url进行处理
2. 当用户输入的路由发送变化的时候，调用hashChange()函数:
    1. 获取用户输入的hash值,这个hash值即为状态名
    2. 检查这个状态是否已经注册，若已经注册过，则将当前的页面内容清除掉，然后为输入的状态创建视图
    3. 若状态未注册，则创建一个新的默认状态

####文件结构如下:
![](http://images2015.cnblogs.com/blog/993343/201610/993343-20161027132818781-469034407.png)

####index.html里的内容
里面没有任何东西，内容都是我们动态加载进去的

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
  </body>
</html>
```

####注意
我使用的是chrome浏览器，由于安全问题，chrome必须通过http等方式才能用$.ajax来获取到文件内容，因此我用了nodejs的http-server自己搭建了一个简单的服务器. 
其他的页面都只是单纯的html文件，没有什么特别,所以就不列举出来了


####截图
输入服务器的url
![](http://images2015.cnblogs.com/blog/993343/201610/993343-20161027133301625-783701980.png)
修改url，在后面加上#listView(之前在初始化的时候就已经注册过的状态)
![](http://images2015.cnblogs.com/blog/993343/201610/993343-20161027133432093-1871796672.png)
输入一个没有注册过的状态(注册了一个默认的状态)
![](http://images2015.cnblogs.com/blog/993343/201610/993343-20161027133459890-1730907136.png)

接下来打算做一下嵌套状态，如果有什么好的建议，麻烦告诉下我~~