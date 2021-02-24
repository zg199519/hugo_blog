---
title: "Vue- 双向数据绑定 原理分析 （未完待续）"
date: 2021-02-02
draft: false
tags: ["Vue源码解析"]
categories: ["vue","js"]
# featured_image: https://wujunze.com/blog-images/r/pic/20201027144234.png
---

# 变化侦测篇

## 综述
### 前言

众所周知，Vue最大的特点之一就是数据驱动视图！
`ui = render(state)`

ui,state 都是用户的 可操作的，唯一不变的是render Vue 就扮演了
render 这个角色 当 Vue 发现 state 变化之后 经过一系列 处理 最终将变化
反应到ui上  

vue 怎么知道 state 变化了呢？

### 什么是变化侦测

变化侦测就是追踪状态，或者说是追踪数据的变化，一旦发生了变化，就要去更新视图。
Angular 中是通过脏值检查流程来实现变化侦测的。
React 中是通过对比虚拟DOM 来实现变化侦测的。
Vue 中也有自己一套变化侦测机制。

### 总结

数据驱动视图： 数据的变化引起视图变化 通过侦测数据变化来驱动 视图的变化。

## Object 变化的侦测

### 前言

js 为我们提供了 `Object.defineProperty` 通过该方法 我们就可以轻松的知道数据在什么时候发生变化了。

### 使Object 数据变得可观测

数据什么时候读了 取了 我们称之为 set  get 操作 的时候 我们可以知道 ， 或者说是 会通知我们， 就称为 数据 是可 观测的！
借助于`Object.defineProperty` 这个方法我们就可以使得数据变为可观测。


下面举例子

```bash
let obj = {}
let value = 2000
Object.defineProperty(obj, 'hello' , {
    enumerable: true,
    configurable: true,
    get () {
        console.log('属性被获取了')
        return value
    },
    set (newVal) {
        console.log('属性被修改了')
    }
})
```

结果如下：
```bash
    obj.hello
    obj.hello = 3000
    VM964:7 属性被获取了
    VM964:11 属性被修改了
    3000
```

这只是 实现个别数据 可 观测 并不完美
接下来我们尝试 使用递归的方法 让 Obj 的所有属性 都变为可观测的。

```bash
// 源码位置：src/core/observer/index.js

// Observer类会通过递归的方式把一个对象的所有属性都转化成可观测对象

export class Observer {
    constructor (value) {
        this.value = value
        // 给value 新增一个属性 __obj__ ， 值为该value的 Observer实例
        // 相当于 给 value 打上了标记， 表示他已经被转换成了响应式了， 为了避免重复操作
        def(value, '__obj__', this)
        if(Array.isArray(value)){
            // 为数组的 操作逻辑 ...
        } else {
            this.walk(value)
        }
    }

    walk (obj) {
        const keys = Object.keys(obj)
        for (let i = 0; i < keys.length; i++) {
            defineReactive(obj, keys[i])
        }
    }

    /**
    * 使一个对象转化成可观测对象
    * @param { Object } obj 对象
    * @param { String } key 对象的key
    * @param { Any } val 对象的某个key的值
    */
    function defineReactive (obj,key,val) {
        // 如果只传了obj和key，那么val = obj[key]
        if (arguments.length === 2) {
            val = obj[key]
        }
        if(typeof val === 'object'){
            // 如果 属性值还是 object 那么重复调用 当前类
            new Observer(val)
        }
        Object.defineProperty(obj, key, {
            enumerable: true,
            configurable: true,
            get(){
            console.log(`${key}属性被读取了`);
            return val;
            },
            set(newVal){
            if(val === newVal){
                return
            }
            console.log(`${key}属性被修改了`);
            val = newVal;
            }
        })
    }
}

```
在上面的代码中，我们定义了observer类，它用来将一个正常的object转换成可观测的object。

并且给value新增一个`__ob__`属性，值为该value的`Observer`实例。

这个操作相当于为value打上标记，表示它已经被转化成响应式了，避免重复操作然后判断数据的类型，
只有object类型的数据才会调用`walk`将每一个属性转换成getter/setter的形式来侦测变化。 

最后，在defineReactive中当传入的属性值还是一个object时使用`new observer（val）`来递归子属性，这样我们就可以把obj中的所有属性（包括子属性）都转换成`getter/seter`的形式来侦测变化。 


也就是说，只要我们将一个`object`传到`observer`中，那么这个`object`就会变成可观测的、响应式的`object`。

### 依赖收集
#### 什么是依赖收集？
我们上面 已经可以侦听 数据的变化了 那我们怎么 让视图更新呢 ， 同时我们不能把整个视图更新一遍 这样做没有意义。
我们可以 只更新 那些用到这个数据的地方 视图中 谁依赖这个数据 谁就是这个数据的依赖，那依赖收集就是把视图中用到这个数据
中的所有依赖收集起来，就是依赖收集。

#### 何时收集依赖？何时通知依赖更新？
既然我们把数据 变为了可观测的，那么就是说 获取数据 会触发 getter , 更新数据 会触发 setter.
那么就好办了 我们在获取数据的时候 也就是说 getter的时候 收集依赖就好了，
在setter 的时候 我们去通知 所有 针对于这个数据的依赖 让他们去更新 依赖即可！

简述： 在getter 中收集依赖， 在setter 中触发依赖更新！

#### 依赖收集到哪里去？

简单理解 依赖收集 那么 依赖 就可能 >= 1 个 是一个集合，那么js 中如果存储一个集合！
当然是数组了。
但是说 仅仅用一个数组去存储 依赖 还不够，怎么去 更新依赖呢， 收集依赖的方法呢。
所以我们可以创建一个依赖收集器， 收集针对当前数据的依赖 并为这些 依赖 创建 收集 和 更新的方法。

```bash
// 源码位置：src/core/observer/dep.js
export default class Dep {
  constructor () {
    this.subs = []
  }

  addSub (sub) {
    this.subs.push(sub)
  }
  // 删除一个依赖
  removeSub (sub) {
    remove(this.subs, sub)
  }
  // 添加一个依赖
  depend () {
    if (window.target) {
      this.addSub(window.target)
    }
  }
  // 通知所有依赖更新
  notify () {
    const subs = this.subs.slice()
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
}

/**
 * Remove an item from an array
 */
export function remove (arr, item) {
  if (arr.length) {
    const index = arr.indexOf(item)
    if (index > -1) {
      return arr.splice(index, 1)
    }
  }
}
```
有了依赖管理器后，我们就可以在getter中收集依赖，在setter中通知依赖更新了，代码如下：

```bash
function defineReactive (obj,key,val) {
  if (arguments.length === 2) {
    val = obj[key]
  }
  if(typeof val === 'object'){
    new Observer(val)
  }
  const dep = new Dep()  //实例化一个依赖管理器，生成一个依赖管理数组dep
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get(){
      dep.depend()    // 在getter中收集依赖
      return val;
    },
    set(newVal){
      if(val === newVal){
          return
      }
      val = newVal;
      dep.notify()   // 在setter中通知依赖更新
    }
  })
}

```
在上述代码中，我们在getter中调用了dep.depend()方法收集依赖，在setter中调用dep.notify()方法通知所有依赖更新。

### 依赖到底是什么？

在vue中还实现了一个叫做 `Watcher` 的类 谁用到了这个数据 谁就是依赖 我们就为哪个谁创建一个  `Watcher` 
之后数据变化的时候 我们不直接去通知依赖更新，而是通知依赖对应的 `Watch` 实例 由 Watcher 实例去通知真正的依赖更新。

```bash
export default class Watcher {
  constructor (vm,expOrFn,cb) {
    this.vm = vm;
    this.cb = cb;
    this.getter = parsePath(expOrFn)
    this.value = this.get()
  }
  get () {
    window.target = this;
    const vm = this.vm
    let value = this.getter.call(vm, vm)
    window.target = undefined;
    return value
  }
  update () {
    const oldValue = this.value
    this.value = this.get()
    this.cb.call(this.vm, this.value, oldValue)
  }
}

/**
 * Parse simple path.
 * 把一个形如'data.a.b.c'的字符串路径所表示的值，从真实的data对象中取出来
 * 例如：
 * data = {a:{b:{c:2}}}
 * parsePath('a.b.c')(data)  // 2
 */
const bailRE = /[^\w.$]/
export function parsePath (path) {
  if (bailRE.test(path)) {
    return
  }
  const segments = path.split('.')
  return function (obj) {
    for (let i = 0; i < segments.length; i++) {
      if (!obj) return
      obj = obj[segments[i]]
    }
    return obj
  }
}
```

分析下 `Watcher` 类 代码的实现逻辑
在构造函数中 先去调用了 `this.get()` 实例方法在 `get()` 方法中
把 自身 赋值给了一个全局的一个唯一 对象 `window.target` 然后通过 `let value = this.getter.call(vm, vm)` 获取一下被依赖的数据
获取依赖数据的目的是触发该数据上面的`getter` 然后 在 `getter` 方法里面会调用实例化的 dep 中的 `dep.depend()` 收集依赖
在 `dep.depend()` 方法中 取到挂载 `window.target` 的值 并将值存入依赖的数组中 在 get() 方法最后 将 `window.target` 释放。

![22](http://localhost:1313/20210202/001.png)

### 不足之处

虽然我们通过`Object.defineProperty`方法实现了对`object`数据的可观测，但是这个方法仅仅只能观测到`object`数据的取值及设置值，当我们向`object`数据里添加一对新的`key/value`或删除一对已有的`key/value`时，它是无法观测到的，导致当我们对object数据添加或删除值时，无法通知依赖，无法驱动视图进行响应式更新。

当然，Vue也注意到了这一点，为了解决这一问题，Vue增加了两个全局API:`Vue.set`和`Vue.delete`


## Array 变化的侦测

未完待续


































_写的不对，还请**批评**。_