# React hooks实践

## 前言

最近要对旧的项目进行重构，统一使用全新的react技术栈。同时，我们也决定尝试使用React hooks来进行开发，但是，由于React hooks崇尚的是使用(也只能使用)function component的形式来进行开发，而不是class component，因此，整个开发方式也会与之前产生比较大的差异。所以，我这里就积累了下实际项目中遇到的问题以及思考，看下能不能帮助大家少走弯路。

## 正文

接下来就直接进入正文。我会将项目中遇到的问题一一列举出来，并且给出解决方案。

### 执行初始化操作的时机

当我转到React hooks的时候，首先就遇到了这个问题：

一般来说，业务组件经常会遇到要通过发起ajax请求来获取业务数据并且执行初始化操作的场景。在使用class component编程的时候，我们就可以在class component提供的生命周期钩子函数(比如componentDidMount, constructor等)执行这个操作。可是如果转到React hooks之后，function component里是没有这个生命周期钩子函数的，那这个初始化操作怎么办呢？总不能每次遇到这种场景都使用class component来做吧？

解决方案：**使用useEffect**(想知道useEffect是什么的话，可以[点击这里](https://reactjs.org/docs/hooks-effect.html))

useEffect，顾名思义，就是执行有副作用的操作，你可以把它当成`componentDidMount`, `componentDidUpdate`, and `componentWillUnmount` 的集合。它的函数声明如下

```javascript
useEffect(effect: React.EffectCallback, inputs?: ReadonlyArray<any> | undefined)
```

那么，我们在实际使用中，我们就可以使用这个来执行初始化操作。举个例子

```jsx
import React, { useEffect } from 'react'

export function BusinessComponent() {
  const initData = async () => {
    // 发起请求并执行初始化操作
  }
  // 执行初始化操作，需要注意的是，如果你只是想在渲染的时候初始化一次数据，那么第二个参数必须传空数组。
  useEffect(() => {
    initData();
  }, []);

  return (<div></div>);
}
```

**需要注意的是，这里的useEffect的第二个参数必须传空数组，这样它就等价于只在componentDidMount的时候执行。如果不传第二个参数的话，它就等价于componentDidMount和componentDidUpdate**

### 做一些清理操作

由于我们在实际开发过程中，经常会遇到需要做一些副作用的场景，比如轮询操作(定时器、轮询请求等)、使用浏览器原生的事件监听机制而不用react的事件机制(这种情况下，组件销毁的时候，需要用户主动去取消事件监听)等。使用class Component编程的时候，我们一般都在componentWillUnmount或者componentDidUnmount的时候去做清理操作，可是使用react hooks的时候，我们如何做处理呢？

解决方案：**使用useEffect第一个参数的返回值**

如果useEffect的第一个参数返回了函数的时候，react会在每一次执行新的effects之前，执行这个函数来做一些清理操作。因此，我们就可以使用它来执行一些清理操作。

**例子**：比如我们要做一个二维码组件，我们需要根据传入的userId不断轮询地向后台发请求查询扫描二维码的状态，这种情况下，我们就需要在组件unmount的时候清理掉轮询操作。代码如下：

```jsx
import React, { useEffect } from 'react'

export function QRCode(url, userId) {
  // 根据userId查询扫描状态
  const pollingQueryingStatus = async () => {
  }
  // 取消轮询
  const stopPollingQueryStatus = async() => {
  }

  useEffect(() => {
    pollingQueryingStatus();

    return stopPollingQueryStatus;
  }, []);

  // 根据url生成二维码
  return (<div></div>)
}
```

这样的话，就等价于在componentWillUnmount的时候去执行清理操作。

但是，有时候我们可能需要执行多次清理操作。还是举上面的例子，我们需要在用户传入新的userId的时候，去执行新的查询的操作，同时我们还需要清除掉旧的轮询操作。想一下怎么做比较好。

其实对这种情况，官方也已经给出了解决方案了，useEffect的第二个参数是触发effects的关键，如果用户传入了第二个参数，那么只有在第二个参数的值发生变化(以及首次渲染)的时候，才会触发effects。因此，我们只需要将上面的代码改一下：

```jsx
import React, { useEffect } from 'react'

export function QRCode(url, userId) {
  // 根据userId查询扫描状态
  const pollingQueryingStatus = async () => {
  }

  const stopPollingQueryStatus = async() => {
  }
  // 我们只是将useEffect的第二个参数加了个userId
  useEffect(() => {
    pollingQueryingStatus();

    return stopPollingQueryStatus;
  }, [userId]);

  // 根据url生成二维码
  return (<div></div>)
}
```

我们只是在useEffect的第二个参数数组里，加入了一个userId。这样的话，userId的每一次变化都会先触发stopPollingQueryStatus，之后再执行effects，这样就可以达到我们的目的。

### useState与setState的差异

react hooks使用useState来代替class Component里的state。可是，在具体开发过程中，我也发现了一些不同点。useState介绍可以[点击这里](https://reactjs.org/docs/hooks-state.html)

在setState的时候，我们可以只修改state中的局部变量，而不需要将整个修改后的state传进去，举个例子

```jsx
import React, { PureComponent } from 'react';

export class Counter extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      count: 0,
      name: 'cjg',
      age: 18,
    }
  }

  handleClick = () => {
    const { count } = this.state;
    // 我们只需要传入修改的局部变量
    this.setState({
      count: count + 1,
    });
  }

  render() {
    return (
      <button onClick={this.handleClick}></button>
    )
  }
}
```

而使用useState后，我们修改state必须将整个修改后的state传入去，因为它会直接覆盖之前的state，而不是合并之前state对象。

```jsx
import React, { useState } from 'react';

export function Count() {
  const [data, setData] = useState({
    count: 0,
    name: 'cjg',
    age: 18,
  });
    
  const handleClick = () => {
    const { count } = data;
    // 这里必须将完整的state对象传进去
    setData({
      ...data,
      count: count + 1,
    })
  };

  return (<button onClick={handleClick}></button>)
}
```

### 减少不必要的渲染

在使用class Component进行开发的时候，我们可以使用`shouldComponentUpdate`来减少不必要的渲染，那么在使用react hooks后，我们如何实现这样的功能呢？

解决方案：**React.memo**和**useMemo**

对于这种情况，react当然也给出了官方的解决方案，就是使用React.memo和useMemo。

#### React.memo

React.momo其实并不是一个hook，它其实等价于PureComponent，但是它只会对比props。使用方式如下(用上面的例子):

```javascript
import React, { useState } from 'react';

export const Count = React.memo((props) => {
  const [data, setData] = useState({
    count: 0,
    name: 'cjg',
    age: 18,
  });
  
  const handleClick = () => {
    const { count } = data;
    setData({
      ...data,
      count: count + 1,
    })
  };

  return (<button onClick={handleClick}>count:{data.count}</button>)
});

```

#### useMemo

useMemo它的用法其实跟useEffects有点像，我们直接看官方给的例子

```jsx
function Parent({ a, b }) {
  // Only re-rendered if `a` changes:
  const child1 = useMemo(() => <Child1 a={a} />, [a]);
  // Only re-rendered if `b` changes:
  const child2 = useMemo(() => <Child2 b={b} />, [b]);
  return (
    <>
      {child1}
      {child2}
    </>
  )
}
```

从例子可以看出来，它的第二个参数和useEffect的第二个参数是一样的，只有在第二个参数数组的值发生变化时，才会触发子组件的更新。

## 总结

一开始在从class component转变到react hooks的时候，确实很不适应。可是当我习惯了这种写法后，我的心情如下：

![](http://img.111cn.net/attachment/art/166435/ae347058c4.jpeg)

当然，现在react hooks还是在alpha阶段，如果大家觉得不放心的话，可以再等等。反正我就先下手玩玩了哈哈哈。