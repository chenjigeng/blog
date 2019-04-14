# React 中无用但可以装逼的知识

最近看了[Dan Abramov](https://mobile.twitter.com/dan_abramov)的一些[博客](<https://overreacted.io/>)，学到了一些React的一些有趣的知识。决定结合自己的理解总结下。这些内容可能对你实际开发并没有什么帮助，不过这可以让你了解到更多React底层实现的内容以及为什么要怎样实现。可以让你跟别人有更多的谈资，当然，也可以在某些场合装一下逼。那么接下来直接进入正文。

## React如何区分类组件和函数组件

我们可以考虑从几种方式：

### 统一使用new方法来生成实例

问题：

- 对于函数组件而言，这样会让它们生成一个多余的`this`作为对象实例。

- 对于箭头函数而言，会报错。因为箭头函数并没有`this`,它的`this`是取自于定义这个箭头函数所在环境的`this`

  ```javascript
  const fun = () => console.log(2);
  new fun(); // Uncaught TypeError: fun is not a constructor
  ```

- 使用`new`会妨碍函数组件返回原始类型(string、number等)。

  我们都知道，使用new操作符后，只有当函数返回非`null` 和非`undefined`的对象的时候，返回值才会生效。否则new操作符的返回值都会是对象。关于`new`操作符详细的内容可以点击[这里](<https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/new>)

  ```javascript
  function Greeting() {
    return 'Hello';
  }
  
  // 并不会返回字符串
  new Gretting(); // Gretting {}
  ```

综上所述，这个方法不可行。

### 通过instanceof来判断

不知道你有没有察觉，我们写`React`的类组件的时候，我们都需要通过`extends React.Component`的方式来写。那么，我们是否可以通过以下方式来判断呢？

```javascript
class A extends React.Component {
}

A.prototype instanceOf React.Component; // true
```

通过这种方式，我们确实可以区分类组件和函数组件，可是也存在一些问题：

- 箭头函数没有`prototyoe`

  这个问题其实好解决，如下

  ```javascript
  function getType(Component) {
    if (Component.prototyoe && Component.prototype instance React.Component) {
      return 'class';
    }
    
    return 'function';
  }
  ```

- 对于一些项目(虽然很少)可能存在着多个React副本，并且我们目前要检查的组件它继承的React.Component是来自于另一个React副本的，这就会出现问题。这个问题的话就没办法解决了。

### 通过为React.Component增加一个特别的标记

写过`React`的类组件的人都知道，我们每一个类组件都是要继承于`React.Component`的。因此，如果我们在`React.Component`增加一个标记`isReactComponent`，这样通过继承的方式，我们就可以根据这个标记来判断是不是类组件了。

```javascript
// React 内部
class Component {}
Component.prototype.isReactComponent = {};

// 检查
class Greeting extends Component {};
console.log(Greeting.prototype.isReactComponent);
```

**事实上，[React](<https://github.com/facebook/react/blob/769b1f270e1251d9dbdce0fcbd9e92e502d059b8/packages/react-reconciler/src/ReactFiber.js#L297-L300>)目前就是通过这种方式来进行检查的。**如果你没有`extends React.Component`，React不会在原型上找到`isReactComponent`，因此不会把组件当做类组件来处理。



## React Elements为什么要有一个$typeof属性

假如我们的jsx长这个样子：

```jsx
<Button type="primary">点击</Button>
```

实际上，在经过babel后，它会变成下面这段代码：

```javascript
React.createElement(
  /* type */ 'Button',
  /* props */ { type: 'primary' },
  /* children */ '点击'
)
```

之后，这个函数执行结果会返回一个对象，这个对象我们称为`React Element`。它是一个用来描述我们将要渲染的页面结构的一个不可变对象。想了解更多与`React Component`,`Elements`和`Inastances`的可以[点击这里](<https://reactjs.org/blog/2015/12/18/react-components-elements-and-instances.html>)。

```javascript
// React Element
{
  type: 'Button',
  props: {
    type: 'primary',
    children: '点击',
  },
  key: null,
  ref: null,
  $$typeof: Symbol.for('react.element'), // 为什么有这个东西
}
```

对于React开发者来说，上面这些属性大部分都是比较常见的。可是为什么混进了一个奇怪的`$$typeof`？？它是干嘛的呢？它的值为什么是一个`Symbol`呢？

这个属性的引入，其实要从一个安全漏洞说起。

假如我们要显示一个变量，如果你使用纯js来写的话，可能是这样：

```javascript
const messageEl = document.getElementById('message');
messageEl.innerHTML = `<div>${message}</div>`;
```

这一段代码，对于熟悉或者了解过XSS攻击的人来说，一看就知道会有问题，**存在着XSS攻击**。如果`message`是用户可以控制的变量(比如说是用户输入的评论)的话，那么用户就可以进行攻击了。比如用户可以构造下面的代码来进行攻击:

```javascript
message = '<img onerror="alert(2)" src="" />';
```

如果我们明确知道，我们只想单纯的渲染文本，不想把它当成html来渲染的话，那么我们可以通过textContent来避免这个问题。

```javascript
const messageEl = document.getElementById('message');
messageEl.textContent = `<div>${message}</div>`;
```

而对于React而言的话，想要实现相同的效果，只需要:

```jsx
<div>{message}</div>
```

即使message里面含有`img`、`script`类似的标签，它们最终也不会以实际上的标签显示。React会对渲染的内容进行转译，比如说上面的攻击代码会被转译为:

```javascript
message = '<img onerror="alert(2)" src=""/>';
// 转译为
message = '&lt;img onerror="alert(2)" src=""/&gt;'
```

因此，这样就可以避免大部分场景下的XSS攻击了。

当然，React也提供了另一种方式来将用户输入的内容当成html来渲染: 

```jsx
<div dangerouslySetInnerHTML={{ __html: message }}></div>
```

前面说了这么多，那么跟`$$typeof`又有什么关系呢？别急，重点来了。

对于下面这种写法，我们一般都知道，message可以传基本类型、自定义组件和jsx片段。

```jsx
<div>{message}</div>
```

可是，其实我们还可以直接传React Element。比如，我们可以直接这样写

```jsx
class App extends React.Component {
  render() {
    const message = {
      type: "div",
      props: {
        dangerouslySetInnerHTML: {
          __html: `<h1>Arbitrary HTML</h1>
            <img onerror="alert(2)" src="" />
            <a href='http://danlec.com'>link</a>`
        }
      },
      key: null,
      ref: null,
      $$typeof: Symbol.for("react.element")
    };
    return <>{message}</>;
  }
}
```

这样在运行的时候，就会弹出一个alert框了。[查看demo](<https://codesandbox.io/s/1rjnq88rlj>)。那么，这样会有什么风险呢？

考虑一个场景，比如一个博客网站的评论信息`message`是由用户提供的，并且支持传入JSON。那么如果用户直接将上文的message发送给后台保存。之后，通过下面这种方式展示的话，用户就可以进行XSS攻击了。

```jsx
<div>{message}</div>
```

假设如果没有$$typeof属性的话，这种攻击确实可行。因为其他的属性都是可序列化的。

```javascript
const message = {
  type: "div",
  props: {
    dangerouslySetInnerHTML: {
      __html: `<h1>Arbitrary HTML</h1>
<img onerror="alert(2)" src="" />
<a href='http://danlec.com'>link</a>`
    }
  },
  key: null,
  ref: null,
};

JSON.stringify(message);
```

事实上，React 0.13当时就存在着这个漏洞。之后，React 0.14就修复了这个问题，修复方式就是通过引入$$typeof属性，并且用Symbol来作为它的值。

```javascript
// 引入 $$typeof
const message = {
  type: "div",
  props: {
    dangerouslySetInnerHTML: {
      __html: `<h1>Arbitrary HTML</h1>
<img onerror="alert(2)" src="" />
<a href='http://danlec.com'>link</a>`
    }
  },
  key: null,
  ref: null,
  $$typeof: Symbol.for("react.element")
};

JSON.stringify(message); // Symbol无法被序列化
```

这是一个有效的方法，因为JSON是不支持`Symbol`类型的。所以，即使用户提交了如上的`message`信息，到最后服务端也不会保存$$typeof属性。而在渲染的时候，React 会检测是否有`$$typeof`属性。如果没有这个属性，则拒绝处理该元素。

那么如果浏览器不支持[`Symbol`](<https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol#Browser_compatibility>)怎么办？

是的，那这种保护方案就没用了。React 依然会加上$$typeof字段，并且将其值设置为`0xeac7`。(为什么是这个数字呢，因为这个数字看起来有点像`React`)。

想查看具体的攻击流程，可以查看[这篇博客](<http://danlec.com/blog/xss-via-a-spoofed-react-element>)。

## 总结

* `React`会给`React.Component.prototype`增加一个`isReactElement`标志。这样，`React`就可以在渲染的时候判断当前渲染的组件是类组件还是函数组件。
* `React Element`是一个用于描述要渲染的页面结构的一个不可变对象。`React`函数组件和类组件执行到最后，其实都是生成一个React Elements树。之后再由实际的渲染层(react-dom、react-native)根据这个`React Elements`树渲染为实际的页面。
* `<div>{message}</div>`这种方式不仅可以传原型类型、jsx和组件，还可以直接传React Element对象。
* `$$typeof`的出现就是为了防止服务端允许储存JSON而引起的XSS攻击。可是对于不支持`Symbol`的浏览器，这个问题依然存在。

## 参考资料

[Why Do React Elements Have a $$typeof Property?](<https://overreacted.io/why-do-react-elements-have-typeof-property/>)

[How Does React Tell a Class from a Function?](<https://overreacted.io/how-does-react-tell-a-class-from-a-function/>)

[XSS via a spoofed React element](http://danlec.com/blog/xss-via-a-spoofed-react-element)