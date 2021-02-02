---
title: "Vue- keep-alive 原理分析 （未完待续）"
date: 2021-02-02
draft: false
tags: ["Vue源码解析"]
categories: ["vue","js"]
# featured_image: https://wujunze.com/blog-images/r/pic/20201027144234.png
---

# 定义

主要用于保留组件状态或避免重新渲染。 

include - 字符串或正则表达式。只有名称匹配的组件会被缓存。 

exclude - 字符串或正则表达式。任何名称匹配的组件都不会被缓存。 

max - 数字。最多可以缓存多少组件实例。

`<keep-alive>` 包裹动态组件时，会缓存不活动的组件实例，而不是销毁它们。 

和 `<transition> `相似，`<keep-alive>` 是一个抽象组件： 

它自身不会渲染一个 DOM 元素，也不会出现在组件的父组件链中。

当组件在 `<keep-alive> `内被切换，它的 `activated` 和 `deactivated` 这两个生命周期钩子函数将会被对应执行。

在 2.2.0 及其更高版本中，`activated` 和 `deactivated` 将会在 `<keep-alive>` 树内的所有嵌套组件中触发。

# 使用方法

```bash
  <!-- 基本 -->
  <keep-alive>
    <component :is="view"></component>
  </keep-alive>

  <!-- 多个条件判断的子组件 -->
  <keep-alive>
    <comp-a v-if="a > 1"></comp-a>
    <comp-b v-else></comp-b>
  </keep-alive>

  <!-- 和 `<transition>` 一起使用 -->
  <transition>
    <keep-alive>
      <component :is="view"></component>
    </keep-alive>
  </transition> 
```
注意，`<keep-alive>` 是用在其一个直属的子组件被开关的情形。如果你在其中有 v-for 则不会工作。如果有上述的多个条件性的子元素，`<keep-alive> `要求同时只有一个子元素被渲染。

> __2.1.0 新增__ `include` **and** `exclude`  

`include` 和 `exclude` prop 允许组件有条件地缓存。二者都可以用逗号分隔字符串、正则表达式或一个数组来表示：

```bash
<!-- 逗号分隔字符串 -->
<keep-alive include="a,b">
  <component :is="view"></component>
</keep-alive>

<!-- 正则表达式 (使用 `v-bind`) -->
<keep-alive :include="/a|b/">
  <component :is="view"></component>
</keep-alive>

<!-- 数组 (使用 `v-bind`) -->
<keep-alive :include="['a', 'b']">
  <component :is="view"></component>
</keep-alive>
```
匹配首先检查组件自身的 `name `选项，如果 `name `选项不可用，则匹配它的局部注册名称 (父组件 `components `选项的键值)。匿名组件不能被匹配。

> ***max***  _2.5.0 新增_

最多可以缓存多少组件实例。一旦这个数字达到了，在新实例被创建之前，  

已缓存组件中最久没有被访问的实例会被销毁掉。

```bash
<keep-alive :max="10">
  <component :is="view"></component>
</keep-alive>
```

`<keep-alive> ` 不会在函数式组件中正常工作，因为它们没有缓存实例。

# 使用场景
当你不希望让组件 每次都刷新的时候 那么你就该用这个方法了

# 解决方案
### 基本使用
在 `<router-view></router-view> `组件中
```bash
<div id="app" class='wrapper'>
    <keep-alive>
        <!-- 需要缓存的视图组件 --> 
        <router-view v-if="$route.meta.keepAlive"></router-view>
     </keep-alive>
      <!-- 不需要缓存的视图组件 -->
     <router-view v-if="!$route.meta.keepAlive"></router-view>
</div>
```
keepAlive 是定义在了 路由表中的 meta 中的 标识
```bash
{
  path: 'list',
  name: 'itemList', // 商品管理
  component (resolve) {
    require(['@/pages/item/list'], resolve)
 },
 meta: {
  keepAlive: true,
  title: '商品管理'
 }
}
```

# 踩坑
待完善...

# 原理简析

源码位置： **src/core/components/keep-alive.js**
先看 对外暴露的对象
```bash
export default {
  name: 'keep-alive',
  abstract: true, // 判断当前组件虚拟dom是否渲染成真实dom的关键
  props: {
      include: patternTypes, // 缓存白名单
      exclude: patternTypes, // 缓存黑名单
      max: [String, Number] // 缓存的组件
  },
  created() {
     this.cache = Object.create(null) // 缓存虚拟dom
     this.keys = [] // 缓存的虚拟dom的键集合
  },
  destroyed() {
    for (const key in this.cache) {
       // 删除所有的缓存
       pruneCacheEntry(this.cache, key, this.keys)
    }
  },
 mounted() {
   // 实时监听黑白名单的变动
   this.$watch('include', val => {
       pruneCache(this, name => matched(val, name))
   })
   this.$watch('exclude', val => {
       pruneCache(this, name => !matches(val, name))
   })
 },

 render() {
    // 先省略...
 }
}

```
可以看出，与我们定义组件的过程一样，先是设置组件名为keep-alive，其次定义了一个abstract属性，值为true。这个属性在vue的官方教程并未提及，却至关重要，后面的渲染过程会用到。props属性定义了keep-alive组件支持的全部参数。

keep-alive在它生命周期内定义了三个钩子函数：

created

初始化两个对象分别缓存VNode(虚拟DOM)和VNode对应的键集合

destroyed

删除this.cache中缓存的VNode实例。我们留意到，这不是简单地将this.cache置为null，而是遍历调用pruneCacheEntry函数删除。

```bash
function pruneCacheEntry (
  cache: VNodeCache,
  key: string,
  keys: Array<string>,
  current?: VNode
) {
 const cached = cache[key]
 if (cached && (!current || cached.tag !== current.tag)) {
    cached.componentInstance.$destroyed() // 执行组件的destroy钩子函数
 }
 cache[key] = null
 remove(keys, key)
}

```
删除缓存的VNode还要对应组件实例的destory钩子函数

mounted

在mounted这个钩子中对include和exclude参数进行监听，然后实时地更新（删除）this.cache对象数据。

pruneCache函数的核心也是去调用pruneCacheEntry

```bash
function pruneCache (keepAliveInstance: any, filter: Function) {
  const { cache, keys, _vnode } = keepAliveInstance
  for (const key in cache) {
    const cachedNode: ?VNode = cache[key]
    if (cachedNode) {
      const name: ?string = getComponentName(cachedNode.componentOptions)
      if (name && !filter(name)) {
        pruneCacheEntry(cache, key, keys, _vnode)
      }
    }
  }
}
```

**render**

```bash
render () {
  const slot = this.$slots.defalut
  const vnode: VNode = getFirstComponentChild(slot) // 找到第一个子组件对象
  const componentOptions : ?VNodeComponentOptions = vnode && vnode.componentOptions
  if (componentOptions) { // 存在组件参数
    // check pattern
    const name: ?string = getComponentName(componentOptions) // 组件名
    const { include, exclude } = this
    if (// 条件匹配
      // not included
      （include && (!name || !matches(include, name))）||
      // excluded
        (exclude && name && matches(exclude, name))
    ) {
        return vnode
    }
    
    const { cache, keys } = this
    // 定义组件的缓存key
    const key: ?string = vnode.key === null ? componentOptions.Ctor.cid + (componentOptions.tag ? `::${componentOptions.tag}` : '') : vnode.key
     if (cache[key]) { // 已经缓存过该组件
        vnode.componentInstance = cache[key].componentInstance
        remove(keys, key)
        keys.push(key) // 调整key排序
     } else {
        cache[key] = vnode //缓存组件对象
        keys.push(key)
        if (this.max && keys.length > parseInt(this.max)) {
          //超过缓存数限制，将第一个删除
          pruneCacheEntry(cahce, keys[0], keys, this._vnode)
        }
     }
     
      vnode.data.keepAlive = true //渲染和执行被包裹组件的钩子函数需要用到
 
 }
 return vnode || (slot && slot[0])
}
```

> 第一步：获取keep-alive包裹着的第一个子组件对象及其组件名；

> 第二步：根据设定的黑白名单（如果有）进行条件匹配，决定是否缓存。不匹配，直接返回组件实例（VNode），否则执行第三步；

> 第三步：根据组件ID和tag生成缓存Key，并在缓存对象中查找是否已缓存过该组件实例。如果存在，直接取出缓存值并更新该key在this.keys中的位置（更新key的位置是实现LRU置换策略的关键），否则执行第四步；

> 第四步：在this.cache对象中存储该组件实例并保存key值，之后检查缓存的实例数量是否超过max设置值，超过则根据LRU置换策略删除最近最久未使用的实例（即是下标为0的那个key）;
  
> 第五步：最后并且很重要，将该组件实例的keepAlive属性值设置为true。


## 渲染
。。。。






_写的不对，还请**批评**。_