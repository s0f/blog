---
title: "Vue源码阅读一、代码的执行流程"
description: "主要是从Vue创建实例开始，看它代码的执行过程，跟着这个过程来阅读Vue的源码，学习Vue的一些底层原理，通过把整个流程跑通后，再去仔细阅读Vue在一些细节上是怎么实现的"
date: 2020-03-16T23:19:01+08:00
draft: true
categories: ["Vue", "源码阅读"]
tags: ["Vue源码阅读", "Vue"]
keywords: ["Vue源码阅读", "Vue"]
share: true

---

主要是从Vue创建实例开始，看它代码的执行过程，跟着这个过程来阅读Vue的源码，学习Vue的一些底层原理，通过把整个流程跑通后，再去仔细阅读Vue在一些细节上是怎么实现的。

<!--more-->
### 入口

有多个不同的打包入口文件，跟平台与是否加载模板编译库有关，作用是会注入一些跟运行平台相关的属性与方法。现在主要看的是Web平台，也就是浏览器相关的文件
- platforms/web/runtime/index.js 从core/index.js 导入Vue构造函数
    - $mount  Vue原型添加$mount方法，浏览器环境下根据el获取dom，作为挂载点，调用mountComponent
    - 注入 \__patch\__  方法 (core/vdom/patch.js  createPatchFunction), 用来创建vnode
    - directives 与 components 注入Vue自带指令与组件
- platforms/web/entry-runtime.js 运行时版本 直接导出 runtime/index.js 下 Vue 构造函数
- platforms/web/entry-runtime-with-compiler.js 运行时带编译器版本
    - 保存原型上$mount，覆盖原型上的 $mount，也就是runtime/index.js中定义的$mount， 如果配置中不存在render方法，将从template属性获取模板。
    如果没有template，将从el属性获取dom并将内容作为模板
    - 根据获取到的template，调用 compileToFunctions(template) 方法创建render函数，赋值给 vm.options.render
    - 调用原来的$mount方法
    - Vue.compile = compileToFunctions
    
- core/index.js
  - initGlobalAPI 扩展Vue全局API
    - set/delete/nextTick
    - component/directive/filter Vue.options[xxx] 添加存放通过全局API注册的功能
    - 注册全局组件 keep-alive
    - initUse(Vue) 提供插件安装函数 Vue.use
    - initMixin(Vue)  提供Vue.mixin
    - initExtend(Vue) 提供Vue.extend，作用是创建一个Vue的子类，这也是使用Vue.component注册组件时，用来创建组件的方法
    - initAssetRegisters(Vue) 过Vue.[component, directive, filter]定义或获取全局可用的component、directive、filter
- **core/instance/index.js** 这里是定义Vue构造函数的地方
    - function Vue(options){ this.**__init__**(options) }  通过init方法来启动Vue实例的创建
    - initMixin(Vue)
    - stateMixin(Vue)
    - eventsMixin(Vue)
    - lifecycleMixin(Vue)
    - renderMixin(Vue)
### initMixin  

- 混入 init 相关功能，初始化一些内部参数。并调用 $mount 挂载元素
- initLifecycle
  - $parent
    options 中有parent属性，如果没有parent，即代表是root元素
  - $root
    如果没有parent，即代表是root元素，属性取值为当前元素
  - children
  - $refs
  - _watcher
  - _inactive
  - _directInactive
  - _isMounted
  - _isDestroyed
  - _isBeginDestroyed 
- initEvents
- initRender
- **hook beforeCreated**
- initInjections
- initState
- initProvide
- **hook created**
- vm.$mount **调用原型上的$mount方法，在不同平台上运行时，这个方法是由不同平台的代码提供的。** 
    在web平台，会覆盖Vue原型上的$mount方法，对模板进行解析编译，返回一个render函数，然后在调用原来的$mount方法。
    这里的render为一个调用createElement创建vnode的函数。这个函数里，会访问到响应式数据，触发get，进行以来收集
  - mountComponent
  - **hook beforeMount**
  - updateComponent
    vm._update(vm._render())
  - new Watcher()
  - **hook mounted**

#### initEvents

 为实例创建一个events数组，存放events回调，如果options中有 parentListeners，则调用 updateComponentListeners。作为root元素，
 这个属性为undefined

- _events
- _hasHookEvent


#### initRender

- 解析slots
- 绑定 _c 与 $createElement两个函数，_c 在经过编译的模板中，用来创建VNode的方法
- 响应式$attrs (得到父元素的attrs)
- 响应式$listeners (得到父元素的options._parentListeners)


### 触发beforeCreated 回调
面试问题：你知道beforeCreate的触发时机吗，它是在初始化哪些内容后被触发？


#### initInjections 注入

#### initState      初始化响应数据

- _watchers 定义观察者数组，用来收集观察者
- initProps 如果有定义props，初始化props
- initMethods
- initData 这里就是初始化响应式数据的地方了，重点
  - proxy(vm, "_data", key)  创建_data, 将 data 上的属性, 响应化后挂到_data中，代码中可通过this.xxx访问到_data上的数据
    _data 就是响应化后的data
  - observe(data, asRoot:true) 定义观察者，进行以来收集
    - 避免循环观察
    - Observer
- initComputed
- initWatch

### stateMixin

- $data/$props
- $watch/$set/$delete 原型上添加set/delete方法，修复defineProperty无法监听添加删除属性的缺陷

### eventsMixin

- $on/$once/$emit/$off 原型添加以上方法，实现vue实例的观察者/监听者模式
- $on
  - $on('hook:xx') 订阅hook事件，设置实例 vm._hasHookEvent 为true，在 hook事件调用时，会根据这个标记调用相应hook回调

### lifecycleMixin

- _update() 第一次挂载和以后的数据更新都会调用此方法去更新视图，有一个isMount属性来区分
  - __patch__() 区分第一次渲染与更新渲染，
    - createElm()
        - createChildren() 编译children数组，对每个children逐个调用createElm()
        - createComponent() 如果children为component
            - componentVNodeHooks.init()
                - child.$mount() 
        - vnode.elm = nodeOps.createTextNode(vnode.text)/createElement/createComment;
        - insert(parentElm, vnode.elm, refElm);
  - __vue__ 会在 $el 上绑定当前对应的vue实例
- $destroy
  - 删除dom、watcher、vnode、listener
  - hook beforeDestroy
  - hook destroy
- foreUpdate

### renderMixin 往原型上添加render相关的函数

- installRenderHelpers  给原型上绑定一些快捷方法
  - _l renderList
  - _n toNumber
  - _s toString
- $nextTick
- _render 调用options.render() 创建vnode
    - vm.$options.render.call(vm._renderProxy, vm.$createElement)  这里render函数，就是模板编译后生成的render函数，通过这段
    函数创建Vnode

### Observer(value) 传入 data 对象

- dep Dep实例
- __ob__ 已经观察
- walk(obj: Object) 对数据类型为对象的进行响应式处理
- observeArray(items: Array<any>) 对数组进行响应式处理
- defineReactive(obj, key, value) 重点，定义依赖收集
  - dep Dep实例 每个key都会有个dep实例保存在defineReactive的闭包中
  - observe

### Watcher
- newDeps 依赖收集阶段，用来收集本组件模板中被使用到的属性的依赖（dep对象）
- get() 调用 expOrFn， 触发依赖收集
  - pushTarget(this)
  - this.getter.call(vm, vm) getter 为updateComponent
      - updateComponent
      - vm._update(vm._render()) // _render在 render.js renderMixin 定义
        调用模板编译返回的render函数，创建该组件的vnode并返回给 _update()
        调用diff算法对比新旧vnode，进行dom操作
      - \__patch\__() 挂载
        - createElm
        - createChildren
  - popTarget()
- pushTarget  popTarget
  将当前的Watcher实例设置为Dep静态属性target，用在属性get中，进行依赖收集，这里是关键。popTarget 作用是
  收集依赖后清除target
- update()
    - queueWatcher() // 异步更新视图
- cleanupDeps

### Dep
- target 指向当前正在收集依赖的Watcher
- subs() 收集Watcher
- depend() 通过Watcher收集dep实例
    - Dep.target.addDep(this)
- notify() 通知Watcher数据更新了
    - subs[i].update()
- pushTarget()/popTarget() 设置target

### Compiler 编译模板
  - mountComponent 创建一个函数，做为当前组件的Watcher的一个expOrFn
    updateComponent = () => { vm._update(vm._render()) }
  - new Watcher(vm, updateComponent)  

- createCompilerCreator 返回一个 createCompiler 函数
  - createCompiler
  - baseCompile
  - parse html 转换 AST
- createElement
  - with
  - simpleNormalizeChildren(children)
- parse 解析vue的各种指令、表达式 \vue\src\compiler\parser\index.js
  - processAttrs
  - processIf
  - processFor
- 模板执行

```javascript
`<div id="box">
	{{ name }}
	{{ age }}

	<p>用户名： {{ name }}</p>
	<child v-if="showChild" :name="name"></child>
</div>`

Vue.component('child', {
  template: `<h1>Child,show parent name: {{name}}</h1>`,
  props: ['name'],
})

new Vue({
  el: '#box', 
  data() {
      return {
          name: 'init Name',
          age: 12,
          showChild: true,
      }
  },
})
```

```javascript
/*
执行顺序
1. name
2. age
3. createTextVNode
4. name
5. createTextVNode
6. _createElement('p')
7. createTextVNode(' ')
8. cihldCancel
9. name
10. _createElement('child')
11. _createElement('div') */

_createElement('div', {attrs: {"id": "box"}}, [
    createTextVNode("\n\t" + toString(name) + "\n\t" + toString(age) + "\n\n\t"),
    _createElement('p', [createTextVNode("用户名： " + toString(name))]),
    createTextVNode(" "),
    (cihldCancel) ? _createElement('child', {attrs: {"name": name}}) : _e()
], 1) 
```



### Directives

- v-html
- v-model
- v-text
- v-for
- v-if


### Virtual Dom

- diff
- patch
  - patchChildren
    - createKeyToOldIdx 返回一个对象，key为 子节点的 key属性，value为子节点的位置index
- 树diff的时间复杂度 O(n^3)，不可用，优化到 O(n)
  - 比较同层tag
  - tag不同，直接删除重建，不再深度比较
  - tag与key相同，不再深度比较
- _v createTextVNode
- createComponent

### 问题

- Vue.component(id, options) 的过程是怎样的, 他的创建时机？
  - VueComponent 注册一个component，会调用Vue.extend(options) 返回一个VueComponent构造函数，并缓存起来，
    创建实例时，调用Vue.ini方法创建Vue实例，VueComponent继承自Vue，
    同时也有与Vue相同的一些全局API 'extend, mixin, use, component, filter, directive, options, cid'
      - VueComponent.options
  - 最后通过createComponent方式创建
    - componentVNodeHooks
    - createComponentInstanceForVnode
    - new vnode.componentOptions.Cto(options) 这里是VueComponent 构造函数
      - Vue._init(options)
- vnode如何表示一个component？ vnode与一般html标签一致，就看_c 就是 createElement 方法如何处理 component了
  - _c('child',{attrs:{"name":name}}) <child :name="name"></child>
  - createElement 方法如何处理 component? 最后返回一个vnode vm._update(vnode)
    - vnode = new VNode // 如果是html标签
    - vnode = createComponent() // 一个component


