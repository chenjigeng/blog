[上一节](https://www.cnblogs.com/chenjg/p/9541291.html)我们已经分析了vue.js是通过Object.defineProperty以及发布订阅模式来进行数据劫持和监听,并且实现了一个简单的demo。今天，我们就基于上一节的代码，来实现一个MVVM类，将其与html结合在一起，并且实现v-model以及{{}}语法。

**tips:本节新增代码（去除注释）在一百行左右。使用的Observer和Watcher都是延用上一节的代码，没有修改**。

接下来，让我们一步步来，实现一个MVVM类。

### 构造函数

首先，一个MVVM的构造函数如下(和vue.js的构造函数一样)：

```javascript
class MVVM {
  constructor({ data, el }) {
    this.data = data;
    this.el = el;
    this.init();
    this.initDom();
  }
}
```

和vue.js一样，有它的data属性以及el元素。

### 初始化操作

vue.js可以通过this.xxx的方法来直接访问this.data.xxx的属性，这一点是怎么做到的呢？其实答案很简单，它是通过Object.defineProperty来做手脚，当你访问this.xxx的时候，它返回的其实是this.data.xxx。当你修改this.xxx值的时候，其实修改的是this.data.xxx的值。具体可以看如下代码：

```javascript
class MVVM {
  constructor({ data, el }) {
    this.data = data;
    this.el = el;
    this.init();
    this.initDom();
  }
  // 初始化
  init() {
    // 对this.data进行数据劫持
    new Observer(this.data);
    // 传入的el可以是selector,也可以是元素，因此我们要在这里做一层处理，保证this.$el的值是一个元素节点
    this.$el = this.isElementNode(this.el) ? this.el : document.querySelector(this.el);
    // 将this.data的属性都绑定到this上，这样用户就可以直接通过this.xxx来访问this.data.xxx的值
    for (let key in this.data) {
      this.defineReactive(key);
    }
  }
    
  defineReactive(key) {
    Object.defineProperty(this, key, {
      get() {
        return this.data[key];
      },
      set(newVal) {
        this.data[key] = newVal;
      }
    })
  }
  // 是否是属性节点
  isElementNode(node) {
    return node.nodeType === 1;
  }
}
```

在完成初始化操作后，我们需要对this.$el的节点进行编译。目前我们要实现的语法有v-model和{{}}语法,v-model这个属性只可能会出现在元素节点的attributes里，而{{}}语法则是出现在文本节点里。

### fragment

在对节点进行编译之前，我们先考虑一个现实问题：如果我们在编译过程中直接操作DOM节点的话，每一次修改DOM都会导致DOM的回流或重绘，而这一部分性能损耗是很没有必要的。因此，我们可以利用fragment,将节点转化为fragment,然后在fragment里编译完成后，再将其放回到页面上。

```javascript
class MVVM {
  constructor({ data, el }) {
    this.data = data;
    this.el = el;
    this.init();
    this.initDom();
  }
  
  initDom() {
    const fragment = this.node2Fragment();
    this.compile(fragment);
    // 将fragment返回到页面中
    document.body.appendChild(fragment);
  }
  // 将节点转为fragment,通过fragment来操作DOM，可以获得更高的效率
  // 因为如果直接操作DOM节点的话，每次修改DOM都会导致DOM的回流或重绘，而将其放在fragment里，修改fragment不会导致DOM回流和重绘
  // 当在fragment一次性修改完后，在直接放回到DOM节点中
  node2Fragment() {
    const fragment = document.createDocumentFragment();
    let firstChild;
    while(firstChild = this.$el.firstChild) {
      fragment.appendChild(firstChild);
    }
    return fragment;
  }
}
```

### 实现v-model

在将node节点转为fragment后，我们来对其中的v-model语法进行编译。

由于v-model语句只可能会出现在元素节点的attributes里，因此，我们先判断该节点是否为元素节点，若为元素节点，则判断其是否是directive(目前只有v-model)，若都满足的话，则调用CompileUtils.compileModelAttr来编译该节点。

编译含有v-model的节点主要有两步：

1. 为元素节点注册input事件，在input事件触发的时候，更新vm(this.data)上对应的属性值。
2. 对v-model依赖的属性注册一个Watcher函数，当依赖的属性发生变化，则更新元素节点的value。

```javascript
class MVVM {
  constructor({ data, el }) {
    this.data = data;
    this.el = el;
    this.init();
    this.initDom();
  }
  
  initDom() {
    const fragment = this.node2Fragment();
    this.compile(fragment);
    // 将fragment返回到页面中
    document.body.appendChild(fragment);
  }
  
  compile(node) {
    if (this.isElementNode(node)) {
      // 若是元素节点，则遍历它的属性，编译其中的指令
      const attrs = node.attributes;
      Array.prototype.forEach.call(attrs, (attr) => {
        if (this.isDirective(attr)) {
          CompileUtils.compileModelAttr(this.data, node, attr)
        }
      })
    }
    // 若节点有子节点的话，则对子节点进行编译
    if (node.childNodes && node.childNodes.length > 0) {
      Array.prototype.forEach.call(node.childNodes, (child) => {
        this.compile(child);
      })
    }
  }
  // 是否是属性节点
  isElementNode(node) {
    return node.nodeType === 1;
  }
  // 检测属性是否是指令(vue的指令是v-开头)
  isDirective(attr) {
    return attr.nodeName.indexOf('v-') >= 0;
  }
}

const CompileUtils = {
  // 编译v-model属性,为元素节点注册input事件，在input事件触发的时候，更新vm对应的值。
  // 同时也注册一个Watcher函数，当所依赖的值发生变化的时候，更新节点的值
  compileModelAttr(vm, node, attr) {
    const { value: keys, nodeName } = attr;
    node.value = this.getModelValue(vm, keys);
    // 将v-model属性值从元素节点上去掉
    node.removeAttribute(nodeName);
    node.addEventListener('input', (e) => {
      this.setModelValue(vm, keys, e.target.value);
    });
      
    new Watcher(vm, keys, (oldVal, newVal) => {
      node.value = newVal;
    });
  },
  /* 解析keys，比如，用户可以传入
  *  <input v-model="obj.name" />
  *  这个时候，我们在取值的时候，需要将"obj.name"解析为data[obj][name]的形式来获取目标值
  */
  parse(vm, keys) {
    keys = keys.split('.');
    let value = vm;
    keys.forEach(_key => {
      value = value[_key];
    });
    return value;
  },
  // 根据vm和keys，返回v-model对应属性的值
  getModelValue(vm, keys) {
    return this.parse(vm, keys);
  },
  // 修改v-model对应属性的值
  setModelValue(vm, keys, val) {
    keys = keys.split('.');
    let value = vm;
    for(let i = 0; i < keys.length - 1; i++) {
      value = value[keys[i]];
    }
    value[keys[keys.length - 1]] = val;
  },
}
```

### 实现{{}}语法

{{}}语法只可能会出现在文本节点中，因此，我们只需要对文本节点做处理。如果文本节点中出现{{key}}这种语句的话，我们则对该节点进行编译。在这里，我们可以通过下面这个正则表达式来对文本节点进行处理，判断其是否含有{{}}语法。

```javascript
const textReg = /\{\{\s*\w+\s*\}\}/gi; // 检测{{name}}语法
console.log(textReg.test('sss'));
console.log(textReg.test('aaa{{  name  }}'));
console.log(textReg.test('aaa{{  name  }} {{ text }}'));
```

若含有{{}}语法，我们则可以对其处理，由于一个文本节点可能出现多个{{}}语法，因此编译含有{{}}语法的文本节点主要有以下两步：

1. 找出该文本节点中所有依赖的属性，并且保留原始文本信息，根据原始文本信息还有属性值，生成最终的文本信息。比如说，原始文本信息是"test {{test}} {{name}}",那么该文本信息依赖的属性有this.data.test和this.data.name,那么我们可以根据原本信息和属性值，生成最终的文本。
2. 为该文本节点所有依赖的属性注册Watcher函数，当依赖的属性发生变化的时候，则更新文本节点的内容。

```javascript
class MVVM {
  constructor({ data, el }) {
    this.data = data;
    this.el = el;
    this.init();
    this.initDom();
  }
  
  initDom() {
    const fragment = this.node2Fragment();
    this.compile(fragment);
    // 将fragment返回到页面中
    document.body.appendChild(fragment);
  }
  
  compile(node) {
    const textReg = /\{\{\s*\w+\s*\}\}/gi; // 检测{{name}}语法
    if (this.isTextNode(node)) {
      // 若是文本节点，则判断是否有{{}}语法，如果有的话，则编译{{}}语法
      let textContent = node.textContent;
      if (textReg.test(textContent)) {
        // 对于 "test{{test}} {{name}}"这种文本，可能在一个文本节点会出现多个匹配符，因此得对他们统一进行处理
        // 使用 textReg来对文本节点进行匹配，可以得到["{{test}}", "{{name}}"]两个匹配值
        const matchs = textContent.match(textReg);
        CompileUtils.compileTextNode(this.data, node, matchs);
      }
    }
    // 若节点有子节点的话，则对子节点进行编译
    if (node.childNodes && node.childNodes.length > 0) {
      Array.prototype.forEach.call(node.childNodes, (child) => {
        this.compile(child);
      })
    }
  }
  // 是否是文本节点
  isTextNode(node) {
    return node.nodeType === 3;
  }
}

const CompileUtils = {
  reg: /\{\{\s*(\w+)\s*\}\}/, // 匹配 {{ key }}中的key
  // 编译文本节点，并注册Watcher函数，当文本节点依赖的属性发生变化的时候，更新文本节点
  compileTextNode(vm, node, matchs) {
    // 原始文本信息
    const rawTextContent = node.textContent;
    matchs.forEach((match) => {
      const keys = match.match(this.reg)[1];
      console.log(rawTextContent);
      new Watcher(vm, keys, () => this.updateTextNode(vm, node, matchs, rawTextContent));
    });
    this.updateTextNode(vm, node, matchs, rawTextContent);
  },
  // 更新文本节点信息
  updateTextNode(vm, node, matchs, rawTextContent) {
    let newTextContent = rawTextContent;
    matchs.forEach((match) => {
      const keys = match.match(this.reg)[1];
      const val = this.getModelValue(vm, keys);
      newTextContent = newTextContent.replace(match, val);
    })
    node.textContent = newTextContent;
  }
}
```

### 结语

这样，一个具有v-model和{{}}功能的MVVM类就已经完成了。[代码地址点击这里。](https://github.com/chenjigeng/vue-data-binding)有兴趣的小伙伴可以上去看下（也可以star or fork下哈哈哈）。

这里也有一个[简单的样例](https://chenjigeng.github.io/example/vue-data-binding/index.html)(忽略样式)。

接下来的话，可能会继续实现computed属性,v-bind方法，以及支持在{{}}里面放表达式。如果觉得这个文章对你有帮助的话，麻烦点个赞，嘻嘻。

最后，贴上所有的代码:

```javascript
class Observer {
  constructor(data) {
    // 如果不是对象，则返回
    if (!data || typeof data !== 'object') {
      return;
    }
    this.data = data;
    this.walk();
  }

  // 对传入的数据进行数据劫持
  walk() {
    for (let key in this.data) {
      this.defineReactive(this.data, key, this.data[key]);
    }
  }
  // 创建当前属性的一个发布实例，使用Object.defineProperty来对当前属性进行数据劫持。
  defineReactive(obj, key, val) {
    // 创建当前属性的发布者
    const dep = new Dep();
    /*
    * 递归对子属性的值进行数据劫持，比如说对以下数据
    * let data = {
    *   name: 'cjg',
    *   obj: {
    *     name: 'zht',
    *     age: 22,
    *     obj: {
    *       name: 'cjg',
    *       age: 22,
    *     }
    *   },
    * };
    * 我们先对data最外层的name和obj进行数据劫持，之后再对obj对象的子属性obj.name,obj.age, obj.obj进行数据劫持，层层递归下去，直到所有的数据都完成了数据劫持工作。
    */
    new Observer(val);
    Object.defineProperty(obj, key, {
      get() {
        // 若当前有对该属性的依赖项，则将其加入到发布者的订阅者队列里
        if (Dep.target) {
          dep.addSub(Dep.target);
        }
        return val;
      },
      set(newVal) {
        if (val === newVal) {
          return;
        }
        val = newVal;
        new Observer(newVal);
        dep.notify();
      }
    })
  }
}

// 发布者,将依赖该属性的watcher都加入subs数组，当该属性改变的时候，则调用所有依赖该属性的watcher的更新函数，触发更新。
class Dep {
  constructor() {
    this.subs = [];
  }

  addSub(sub) {
    if (this.subs.indexOf(sub) < 0) {
      this.subs.push(sub);
    }
  }

  notify() {
    this.subs.forEach((sub) => {
      sub.update();
    })
  }
}

Dep.target = null;

// 观察者
class Watcher {
  /**
   *Creates an instance of Watcher.
   * @param {*} vm
   * @param {*} keys
   * @param {*} updateCb
   * @memberof Watcher
   */
  constructor(vm, keys, updateCb) {
    this.vm = vm;
    this.keys = keys;
    this.updateCb = updateCb;
    this.value = null;
    this.get();
  }

  // 根据vm和keys获取到最新的观察值
  get() {
    // 将Dep的依赖项设置为当前的watcher,并且根据传入的keys遍历获取到最新值。
    // 在这个过程中，由于会调用observer对象属性的getter方法，因此在遍历过程中这些对象属性的发布者就将watcher添加到订阅者队列里。
    // 因此，当这一过程中的某一对象属性发生变化的时候，则会触发watcher的update方法
    Dep.target = this;
    this.value = CompileUtils.parse(this.vm, this.keys);
    Dep.target = null;
    return this.value;
  }

  update() {
    const oldValue = this.value;
    const newValue = this.get();
    if (oldValue !== newValue) {
      this.updateCb(oldValue, newValue);
    }
  }
}

class MVVM {
  constructor({ data, el }) {
    this.data = data;
    this.el = el;
    this.init();
    this.initDom();
  }

  // 初始化
  init() {
    // 对this.data进行数据劫持
    new Observer(this.data);
    // 传入的el可以是selector,也可以是元素，因此我们要在这里做一层处理，保证this.$el的值是一个元素节点
    this.$el = this.isElementNode(this.el) ? this.el : document.querySelector(this.el);
    // 将this.data的属性都绑定到this上，这样用户就可以直接通过this.xxx来访问this.data.xxx的值
    for (let key in this.data) {
      this.defineReactive(key);
    }
  }

  initDom() {
    const fragment = this.node2Fragment();
    this.compile(fragment);
    document.body.appendChild(fragment);
  }
  // 将节点转为fragment,通过fragment来操作DOM，可以获得更高的效率
  // 因为如果直接操作DOM节点的话，每次修改DOM都会导致DOM的回流或重绘，而将其放在fragment里，修改fragment不会导致DOM回流和重绘
  // 当在fragment一次性修改完后，在直接放回到DOM节点中
  node2Fragment() {
    const fragment = document.createDocumentFragment();
    let firstChild;
    while(firstChild = this.$el.firstChild) {
      fragment.appendChild(firstChild);
    }
    return fragment;
  }

  defineReactive(key) {
    Object.defineProperty(this, key, {
      get() {
        return this.data[key];
      },
      set(newVal) {
        this.data[key] = newVal;
      }
    })
  }

  compile(node) {
    const textReg = /\{\{\s*\w+\s*\}\}/gi; // 检测{{name}}语法
    if (this.isElementNode(node)) {
      // 若是元素节点，则遍历它的属性，编译其中的指令
      const attrs = node.attributes;
      Array.prototype.forEach.call(attrs, (attr) => {
        if (this.isDirective(attr)) {
          CompileUtils.compileModelAttr(this.data, node, attr)
        }
      })
    } else if (this.isTextNode(node)) {
      // 若是文本节点，则判断是否有{{}}语法，如果有的话，则编译{{}}语法
      let textContent = node.textContent;
      if (textReg.test(textContent)) {
        // 对于 "test{{test}} {{name}}"这种文本，可能在一个文本节点会出现多个匹配符，因此得对他们统一进行处理
        // 使用 textReg来对文本节点进行匹配，可以得到["{{test}}", "{{name}}"]两个匹配值
        const matchs = textContent.match(textReg);
        CompileUtils.compileTextNode(this.data, node, matchs);
      }
    }
    // 若节点有子节点的话，则对子节点进行编译。
    if (node.childNodes && node.childNodes.length > 0) {
      Array.prototype.forEach.call(node.childNodes, (child) => {
        this.compile(child);
      })
    }
  }
  
  // 是否是属性节点
  isElementNode(node) {
    return node.nodeType === 1;
  }
  // 是否是文本节点
  isTextNode(node) {
    return node.nodeType === 3;
  }

  isAttrs(node) {
    return node.nodeType === 2;
  }
  // 检测属性是否是指令(vue的指令是v-开头)
  isDirective(attr) {
    return attr.nodeName.indexOf('v-') >= 0;
  }

}

const CompileUtils = {
  reg: /\{\{\s*(\w+)\s*\}\}/, // 匹配 {{ key }}中的key
  // 编译文本节点，并注册Watcher函数，当文本节点依赖的属性发生变化的时候，更新文本节点
  compileTextNode(vm, node, matchs) {
    // 原始文本信息
    const rawTextContent = node.textContent;
    matchs.forEach((match) => {
      const keys = match.match(this.reg)[1];
      console.log(rawTextContent);
      new Watcher(vm, keys, () => this.updateTextNode(vm, node, matchs, rawTextContent));
    });
    this.updateTextNode(vm, node, matchs, rawTextContent);
  },
  // 更新文本节点信息
  updateTextNode(vm, node, matchs, rawTextContent) {
    let newTextContent = rawTextContent;
    matchs.forEach((match) => {
      const keys = match.match(this.reg)[1];
      const val = this.getModelValue(vm, keys);
      newTextContent = newTextContent.replace(match, val);
    })
    node.textContent = newTextContent;
  },
  // 编译v-model属性,为元素节点注册input事件，在input事件触发的时候，更新vm对应的值。
  // 同时也注册一个Watcher函数，当所依赖的值发生变化的时候，更新节点的值
  compileModelAttr(vm, node, attr) {
    const { value: keys, nodeName } = attr;
    node.value = this.getModelValue(vm, keys);
    // 将v-model属性值从元素节点上去掉
    node.removeAttribute(nodeName);
    new Watcher(vm, keys, (oldVal, newVal) => {
      node.value = newVal;
    });
    node.addEventListener('input', (e) => {
      this.setModelValue(vm, keys, e.target.value);
    });
  },
  /* 解析keys，比如，用户可以传入
  *  let data = {
  *    name: 'cjg',
  *    obj: {
  *      name: 'zht',
  *    },
  *  };
  *  new Watcher(data, 'obj.name', (oldValue, newValue) => {
  *    console.log(oldValue, newValue);
  *  })
  *  这个时候，我们需要将keys解析为data[obj][name]的形式来获取目标值
  */
  parse(vm, keys) {
    keys = keys.split('.');
    let value = vm;
    keys.forEach(_key => {
      value = value[_key];
    });
    return value;
  },
  // 根据vm和keys，返回v-model对应属性的值
  getModelValue(vm, keys) {
    return this.parse(vm, keys);
  },
  // 修改v-model对应属性的值
  setModelValue(vm, keys, val) {
    keys = keys.split('.');
    let value = vm;
    for(let i = 0; i < keys.length - 1; i++) {
      value = value[keys[i]];
    }
    value[keys[keys.length - 1]] = val;
  },
}
```