从很久之前就已经接触过了angularjs了，当时就已经了解到，angularjs是通过脏检查来实现数据监测以及页面更新渲染。之后，再接触了vue.js，当时也一度很好奇vue.js是如何监测数据更新并且重新渲染页面。今天，就我们就来一步步解析vue.js响应式的原理，并且来实现一个简单的demo。

首先，先让我们来了解一些基础知识。

## 基础知识

### Object.defineProperty

es5新增了Object.defineProperty这个api，它可以允许我们为对象的属性来设定getter和setter,从而我们可以劫持用户对对象属性的取值和赋值。比如以下代码:

```javascript
const obj = {
};

let val = 'cjg';
Object.defineProperty(obj, 'name', {
  get() {
    console.log('劫持了你的取值操作啦');
    return val;
  },
  set(newVal) {
    console.log('劫持了你的赋值操作啦');
    val = newVal;
  }
});

console.log(obj.name);
obj.name = 'cwc';
console.log(obj.name);
```

我们通过Object.defineProperty劫持了obj[name]的取值和赋值操作，因此我们就可以在这里做一些手脚啦，比如说，我们可以在obj[name]被赋值的时候触发更新页面操作。

### 发布订阅模式

发布订阅模式是设计模式中比较常见的一种，其中有两个角色：发布者和订阅者。多个订阅者可以向同一发布者订阅一个事件，当事件发生的时候，发布者通知所有订阅该事件的订阅者。我们来看一个例子了解下。

```javascript
class Dep {
  constructor() {
    this.subs = [];
  }
  // 增加订阅者
  addSub(sub) {
    if (this.subs.indexOf(sub) < 0) {
      this.subs.push(sub);
    }
  }
  // 通知订阅者
  notify() {
    this.subs.forEach((sub) => {
      sub.update();
    })
  }
}

const dep = new Dep();

const sub = {
  update() {
    console.log('sub1 update')
  }
}

const sub1 = {
  update() {
    console.log('sub2 update');
  }
}

dep.addSub(sub);
dep.addSub(sub1);
dep.notify(); // 通知订阅者事件发生，触发他们的更新函数
```

## 动手实践

我们了解了Object.defineProperty和发布订阅者模式后，我们不难可以想到，vue.js是基于以上两者来实现数据监听的。

1. vue.js首先通过Object.defineProperty来对要监听的数据进行getter和setter劫持，当数据的属性被赋值/取值的时候，vue.js就可以察觉到并做相应的处理。
2. 通过订阅发布模式，我们可以为对象的每个属性都创建一个发布者，当有其他订阅者依赖于这个属性的时候，则将订阅者加入到发布者的队列中。利用Object.defineProperty的数据劫持，在属性的setter调用的时候，该属性的发布者通知所有订阅者更新内容。

接下来，我们来动手实现(详情可以看注释)：

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
    Dep.target = this;
    const keys = this.keys.split('.');
    let value = this.vm;
    keys.forEach(_key => {
      value = value[_key];
    });
    this.value = value;
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

let data = {
  name: 'cjg',
  obj: {
    name: 'zht',
  },
};

new Observer(data);
// 监听data对象的name属性，当data.name发现变化的时候，触发cb函数
new Watcher(data, 'name', (oldValue, newValue) => {
  console.log(oldValue, newValue);
})

data.name = 'zht';

// 监听data对象的obj.name属性，当data.obj.name发现变化的时候，触发cb函数
new Watcher(data, 'obj.name', (oldValue, newValue) => {
  console.log(oldValue, newValue);
})

data.obj.name = 'cwc';
data.obj.name = 'dmh';
```

### 结语

这样，一个简单的响应式数据监听就完成了。当然，这个也只是一个简单的demo，来说明vue.js响应式的原理，真实的vue.js源码会更加复杂，因为加了很多其他逻辑。

接下来我可能会将其与html联系起来，实现v-model、computed和{{}}语法。[代码地址](https://github.com/chenjigeng/vue-data-binding) 有兴趣的欢迎来一起研究探讨下。点击[这里](https://www.cnblogs.com/chenjg/p/9548473.html)查看第二节的内容。如果觉得有收获的话也请点个赞，嘿嘿。