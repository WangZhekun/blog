# vue2源码阅读总结

- [vue2源码阅读总结](#vue2源码阅读总结)
  - [1. 生命周期](#1-生命周期)
    - [1.1. Vue脚本执行](#11-vue脚本执行)
    - [1.2. 在`beforeCreate`之前](#12-在beforecreate之前)
    - [1.3. `beforeCreate`到`created`之间](#13-beforecreate到created之间)
    - [1.4. `created`到`beforeMount`之间](#14-created到beforemount之间)
    - [1.5. `beforeMount`到`mounted`之间](#15-beforemount到mounted之间)
    - [1.6. `beforeUpdate`之前](#16-beforeupdate之前)
    - [1.7. `beforeUpdate`到`updated`之间](#17-beforeupdate到updated之间)
    - [1.8. `beforeDestroy`之前](#18-beforedestroy之前)
    - [1.9. `beforeDestroy`到`destroyed`之间](#19-beforedestroy到destroyed之间)
    - [1.10. `destroyed`之后](#110-destroyed之后)
    - [1.11. `activated`之前](#111-activated之前)
    - [1.12. `deactivated`之前](#112-deactivated之前)
  - [2. 编译&渲染](#2-编译渲染)
  - [3. 双向绑定](#3-双向绑定)
  - [4. 时间片](#4-时间片)
  - [5. VNode的Diff算法](#5-vnode的diff算法)
  - [6. KeepAlive](#6-keepalive)
  - [7. 服务端渲染](#7-服务端渲染)
    - [7.1. 业务项目的SSRClient端的webpack入口JS脚本](#71-业务项目的ssrclient端的webpack入口js脚本)
    - [7.2. 业务项目的SSRServer端的webpack入口JS脚本](#72-业务项目的ssrserver端的webpack入口js脚本)
    - [7.3. `vue-loader`和`vue-style-loader`](#73-vue-loader和vue-style-loader)
    - [7.4. `VueSSRClientPlugin` webpack插件](#74-vuessrclientplugin-webpack插件)
    - [7.5. `VueSSRServerPlugin` webpack插件](#75-vuessrserverplugin-webpack插件)
    - [7.6. `vue-ssr-client-manifest.json`文件](#76-vue-ssr-client-manifestjson文件)
    - [7.7. `vue-ssr-server-bundle.json`文件](#77-vue-ssr-server-bundlejson文件)
    - [7.8. `vue-server-renderer`的入口文件](#78-vue-server-renderer的入口文件)
    - [7.9. `vue-server-renderer`的`createBundleRenderer` API](#79-vue-server-renderer的createbundlerenderer-api)
    - [7.10. `vue-server-renderer`的`createRenderer` API](#710-vue-server-renderer的createrenderer-api)
  - [8. 目录结构](#8-目录结构)

## 1. 生命周期

### 1.1. Vue脚本执行

### 1.2. 在`beforeCreate`之前

> `beforeCreate`钩子处于Vue的原型方法`_init`中。`_init`方法在Vue的构造函数中执行，也是唯一一个在构造函数中执行的方法。

初始化当前实例的`$options`

初始化当前实例的声明周期相关属性，如`$parent`、`$root`、`$children`、`_isMounted`等

初始化当前实例的事件监听，主要是`_events`和`_hasHookEvent`属性

初始化当前实例的渲染相关属性，如`_vnode`、`$slots`、`$createElement`等

> 总结：`breforeCreate`之前，主要共工作是处理Vue实例与实例之间的关系

### 1.3. `beforeCreate`到`created`之间

> `created`钩子位置同上。

初始化当前实例需要注入的内容，即把配置项中的`inject`配置的`key`作为当前实例的根属性，并将注入的`key`及其值做响应式处理

初始化当前实例的配置项中的`props`、`methods`、`data`、`computed`、`watch`

初始化当前实例的配置项中的`provide`，即给当前实例添加`_provided`属性，属性值为`provide`配置项的值

> 总结：`beforeCreate`到`created`之间是初始化Vue实例自身的属性（主要是响应式数据对象）

### 1.4. `created`到`beforeMount`之间

> `beforeMount`钩子在`core/instance/lifecycle.js`的`mountComponent`方法中，该方法在当前实例的`$mount`方法中被调用。

如果当前实例的配置对象（`$options`）中有`el`，即挂载点，执行`$mount`。所以，作为根Vue实例，必须是通过`el`配置挂载点的，否则后续的子组件都不会被挂载。

在web平台中，`$mount`方法中先执行了`template`的编译过程。条件：配置对象（`$options`）的`render`不存在，存在`template`属性，如果`template`不存在，则获取`el`对应的模板字符串。

根据平台，获取`el`挂载点，如浏览器，需要获取DOM。

设置`$el`，即挂载点对应的DOM节点（浏览器）。

如果当前实例的配置对象（`$options`）中没有`render`函数，则以创建空VNode的函数初始化之

**问题：VNode树是否在该阶段生成，模板编译生成AST树是否在该阶段生成？？？**

> 总结：`created`到`beforeMount`之间是在准备挂载点

### 1.5. `beforeMount`到`mounted`之间

> `mounted`钩子的位置同上

创建渲染Watcher（RenderWatcher）实例，即当前实例的`_watcher`属性的值。RenderWatcher实例的监听对象的值访问器（`getter`属性）是可以执行当前实例的`_render`和`_update`方法的函数，在RenderWatcher构造函数创建过程中执行。完成渲染过程。

对于根Vue实例（表现为`$vnode`属性为`null`，`$vnode`为当前实例在父实例中的占位节点）：置当前实例的`_isMounted`为`true`，即已挂载标志，并执行`mounted`钩子

对于非根Vue实例：在其对应的（父实例中的）占位节点创建时，即父实例渲染过程中，会执行VNode的钩子，在`init`钩子，创建该实例，并维护实例间的父子关系，执行`$mount`方法。在占位节点的`insert`钩子中，置该实例的`_isMounted`为`true`，即已挂载标志，并执行实例的`mounted`钩子。

> 总结：`beforeMount`到`mounted`之间是在编译模板，完成渲染过程

### 1.6. `beforeUpdate`之前

Vue实例中`data`数据的变化，将模板对应的Watcher实例加入到调度队列中。在`nexttick`中执行调度程序。

### 1.7. `beforeUpdate`到`updated`之间

在调度程序中，调用Watcher的`run`方法，在`run`方法中执行模板渲染，即`vm._update(vm._render(), hydrating)`。

### 1.8. `beforeDestroy`之前

父组件在更新`vnode`时，触发`vnode`的`destory`钩子，在该钩子方法中，如果该`vnode`为子组件的占位节点，那么调用该子组件实例的`$destroy` API。`beforeDestroy`到`destroyed`都在`$destroy` API中触发。

### 1.9. `beforeDestroy`到`destroyed`之间

置Vue实例的`_isBeingDestroyed`开始销毁标志。

删除Vue实例与父实例的关联关系。

删除Vue实例内Watcher与Dep的订阅关系。

清除Vue实例的根响应式数据对象的Ovserver实例的计数器。

置Vue实例的`_isDestroyed`已销毁标志。

执行Vue实例的`__patch__`方法，删除虚拟节点树。

### 1.10. `destroyed`之后

清空Vue实例的事件监听。

移除Vue实例渲染结果（DOM树）对实例的引用。

移除Vue实例的虚拟节点树对父节点的引用。

### 1.11. `activated`之前

在父组件的`vnode`的`insert`钩子中被调用，用于激活`vnode`对应的子组件实例。

置Vue实例及其子实例的失活标志`_inactive`为`false`。

### 1.12. `deactivated`之前

在父组件的`vnode`的`destroy`钩子中被调用，用于失活`vnode`对应的子组件实例。

置Vue实例及其子实例的失活标志`_inactive`为`true`。

## 2. 编译&渲染

编译过程是在Vue的原型方法`$mount`中完成的。

执行编译模块的`createCompilerCreator(function baseCompile(template, options){ ... })`函数（编译函数的工厂函数的生成器），创建`createCompiler(baseOptions)`方法。

执行编译模块的`createCompiler(baseOptions)`方法（编译函数的工厂函数，用于缓存平台的基础编译配置），传入平台专属的基础配置，生成`compileToFunctions(template, options)`方法。

执行`compileToFunctions(template, options)`方法，将模板转换成`vm.$options.render()`方法（不同平台的处理方式不同）。

`vm._render()`方法将模板编译成VNode树。该方法的核心功能是调用`vm.$options.render()`方法，生成VNode树。

`vm._update()`方法将VNode树渲染成DOM树。该方法的核心功能是调用`vm.__patch__()`方法，渲染VNode树。

创建当前Vue实例的根Watcher实例（渲染Watcher），并在Watcher实例的构造方法中执行`vm._update(vm._render(), hydrating)`。

`render()`函数的生成过程见如下简化代码：

```javascript
function createCompileToFunctionFn(compile) { // 编译模块内
  return function compileToFunctions(template, options, vm) { // 编译函数
    // 执行compile
    // 将字符串状态的render，包装成Function
    return { render/* 此处是在实际使用的render函数 */, staticRenderFns}
  }
}
function createCompilerCreator(baseCompile) { // 编译模块内
  return function createCompiler(baseOptions) {
    function compile(template, options) {
      // 合并options和baseOptions，包括一些模块和指令及其他配置项
      // 执行baseCompile，完成编译过程
      return { ast, render, staticRenderFns, errors, tips}
    }
    return {
      compile,
      compileToFunctions: createCompileToFunctionFn(compile)
    }
  }
}
const createCompiler = createCompilerCreator(function baseCompile(template, options) { 
  // 将模板解析成AST树
  // 根据AST树，生成render函数内部的函数体字符串
  return { ast, render/* 是with(this){}包裹的字符串 */, staticRenderFns }
}) // 编译模块内
const { compile, compileToFunctions } = createCompiler(baseOptons) // 平台模块内
const { render, staticRenderFns } = compileToFunctions(template, options) // 平台模块的$mount方法内
options.render = render
options.staticRenderFns = staticRenderFns
```

根据上述简化过程，编译的过程实际上包括：
1. 合并平台基础配置和实际Vue实例的挂载时的配置
2. 编译模板为AST树
3. 根据AST树，生成render函数的函数体
4. 将render函数的函数体包装成实际的Function
5. 执行render函数

`__patch__()`方法的生成过程见如下简化代码：

```javascript
function createPatchFunction(backend) { // core/vdom模块
  return function patch(oldVnode, vnode, hydrating, removeOnly) {
    // 只有新的VNode，则创建对应的DOM树
    // 新旧VNode没有变化（判断条件见下文“判断VNode没有变化的条件”），对比子节点
    // 旧VNode是DOM节点，或新旧VNode发生变化，创建新VNode对应的DOM树
    /*
      判断VNode没有变化的条件：
      1. key一致 && ( 2 || 3 )
      2. tag一致 && 都是注释节点 && data属性一致 && 是相同的Input节点
      3. 旧VNode节点是异步占位节点 && 异步工厂方法相同 && 新VNode节点的异步工厂方法没有错误
    */
  }
}
Vue.prototype.__patch__ = createPatchFunction({ nodeOps/* 平台的DOM节点操作API */, modules/* 平台的模块和核心的基础模块 */ }) // 平台模块内
```

## 3. 双向绑定

双向绑定指的是从`data`到视图，和从视图到`data`的双向数据流。典型的指令是`v-model`指令。

在编译的生成AST树的过程中，将`v-model`指令解析，去掉`v-`前缀。

在编译的生成render函数的函数体字符串过程中，会将解析后的`v-model`，转化为`value`属性的绑定和`input`事件的监听。（这一步是在平台中实现的，并且不同DOM节点的处理方式不同，子组件的占位节点处理后，在`render`函数执行，创建VNode实例时处理）

事件的监听实现了从视图到`data`的数据流。

各种属性的绑定实现了从`data`到视图的数据流。即，响应式数据系统。

在Vue组件的`beforeCreate`到`created`生命周期之间，使用core/observer模块的`observe`方法，对`data`做了响应式处理。

响应式系统的核心代码如下：

```javascript
function initData() { // core/instance模块
  // 将data中的属性，代理到Vue实例的根属性上
  observe(data, true)
}
function observe(value, asRootData) { // core/observer模块，该方法用来对value做响应式处理
  // 创建Observer实例
}
class Observer {
  constructor(value) {
    // 创建Dep实例，与当前Observer实例一一绑定
    // 给value添加`__ob__`属性，值为当前Observer实例，作为已经做了响应式处理的标志
    // 如果value是数组，则扩展其原型链（改写`__proto__`属性，对于不支持`__proto__`属性的，添加改写后的Array原型对象）。改写的作用，主要是在调用这些方法对数组进行操作时，触发当前Observer实例对应的Dep实例的`notify()`方法。
    // 并对数组的每一个元素调用`observe()`方法
    // 如果value是对象，则对其每一个属性，都调用`defineReactive(obj, key)`方法，改造属性的访问方式
  }
}
function defineReactive(obj, key, val, customSetter, shallow) {
  // 创建Dep实例，与`key`属性一一绑定
  // 调用`observe()`方法处理`key`属性的值
  // 使用`Object.defineProperty()`重新定义`key`属性的Getter、Setter访问器
  /*
    Getter:
    1、获取`key`属性的实际值
    2、如果有Watcher实例（`Dep.target`）在收集依赖，则：
    2.1、调用当前Dep实例的`depend()`方法，关联Dep实例和Watcher实例
    2.2、如果`key`属性的值有`__ob__`属性，即值是数组或对象，则调用`__ob__`属性的值（Observer实例）对应的Dep实例的`depend()`方法，关联Dep实例和Watcher实例
    2.3、如果`key`属性的值是数组，则执行2.4
    2.4、调用数组元素（对象）的`__ob__`属性的值（Observer实例）对应的Dep实例的`depend()`方法，关联Dep实例和Watcher实例。
    2.5、如果`key`属性的值是数组，数组元素也是数组，则重复2.4
    总结：Getter中收集一个属性的依赖，包括：该属性的Dep实例、该属性的值（对象，或多维数组的第一层对象）的Observer实例的Dep实例。
  */
  /*
    Setter:
    1、对比新旧值，一样则不作处理
    2、修改值
    3、调用`observe()`方法对新值做处理
    4、调用该属性对应的Dep实例的`notify()`方法
  */
}
class Dep {
  depend() {
    // 将当前Dep实例，添加到`Dep.target`（Watcher实例）的Dep列表中
  }
  notify() {
    // 调用订阅当前Dep实例的所有Watcher实例的`update()`接口，通知其依赖的数据有更新
  }
}
class Watcher {
  /**
    * vm: Vue实例
    * expOrFn: 监听的值的表达式或函数
    * cb: 监听到变化后的回调函数
    * options: Watcher实例的配置项
    * isRenderWatcher: vm的渲染Watcher标志，渲染Watcher指vm的模板渲染过程对应的唯一Watcher实例
    */
  constructor(vm, expOrFn, cb, options, isRenderWatcher) {
    // 初始化各种属性
    // 将expOrFn转化为Function，保存在`this.getter`属性上
    // 如果不是懒更新（`this.lazy`，由`options`参数传入），则执行`this.get()`获取并保存监听的值
  }
  get() {
    // 将当前Watcher实例，放在`Dep.target`上。用来收集`this.getter()`执行过程中依赖的Dep实例，即相关属性的Getter访问器
    // 执行`this.getter`
    // 如果需要深度监听，则深度遍历上一步执行结果的所有属性，即访问这些属性的Getter访问器，收集其依赖
    // 还原`Dep.target`，结束当前Watcher实例的依赖收集
    // 返回执行结果
  }
  update() {
    // 如果是懒更新，则记录有脏数据，即有依赖的数据发生变更
    // 如果不是懒更新，是同步更新，则执行`this.run`
    // 其他情况，调用`queueWatcher()`函数，将当前Watcher实例加入到调度队列
  }
  run() {
    // 执行`this.get()`，获取新的监听的值
    // 对比新旧监听的值，如果发生变化，或者新值为对象，或需要深度监听，则
    // 更新旧值
    // 执行监听的值发生变化后的回调函数
  }
  evaluate() {
    // 执行`this.get()`，获取并保存新的监听的值
    // 重置脏数据标志
  }
}
function queueWatcher(watcher) {
  // 缓存watcher，避免重复添加
  // 如果调度程序（`flushSchedulerQueue`函数）未正在执行，则将watcher加入队列
  // 如果调度程序正在执行，将watcher插入到队列中未处理索引之后，比watcher的id小的Watcher实例之后
  // 如果未注册调度程序，则调用`nextTick()`方法将调度程序注册到下一个时间片中执行
}
function flushSchedulerQueue() {
  // 将队列中的Watcher实例按照id从小到大排序
  // 遍历队列，执行每个Watcher实例的`run()`方法
  // 重置队列状态
  // 调用相关Vue实例的activated钩子。相关的Vue实例列表，是在这些组件的占位节点被创建时加入激活队列的。
  // 调用相关Vue实例的updated钩子（只有Watcher实例是渲染Wacther时，其Vue实例才会调用钩子）
}
```

使用`Watcher`类的三种情况：

```javascript
// 第一种：渲染Watcher
function mountComponent() { // core/instance模块，属于渲染过程，`beforeMount`到`mounted`之间执行
  /*
    创建Watcher实例，作为渲染Watcher。该Watcher实例相关的参数说明如下：
    1、传入执行`vm._update(vm._render(), hydrating)`渲染过程的函数，作为该Watcher实例的`getter`方法
    2、没有监听数据（监听数据为`undefined`）更新后的回调
    3、该Watcher实例，非懒更新，非同步更新（即，数据更新后，在调度程序中执行Watcher实例的`run()`方法），有`before`回调（在`flushSchedulerQueue()`函数中Watcher实例的`run()`方法执行之前执行）
    4、渲染Watcher标志为`true`
  */
}
// 第二种：计算属性
function initComputed() { // core/instance模块，属于Vue实例初始化过程，`beforeCreate`到`created`之间执行
  /*
    创建Watcher实例，并将其加入到`vm._computedWatchers`映射中，`key`为计算属性的属性名。Watcher实例相关参数说明如下：
    1、传入计算属性的Getter访问器作为Watcher实例的`getter`方法
    2、没有监听数据更新后的回调
    3、Watcher实例，懒更新，非同步更新
  */
  // 调用`defineComputed(vm, key, userDef)`方法，给Vue实例定义计算属性。`userDef`为Vue实例的`$options.computed`配置项中的属性值。
}
function defineComputed(target, key, userDef) {
  // 调用`createComputedGetter(key)`方法，获取计算属性的Getter访问器
  // 给`target`添加`key`属性，用属性的Getter和Setter访问器包装`userDef`
}
function createComputedGetter(key) {
  return function computedGetter() {
    // 获取`key`计算属性对应的Watcher实例
    // 如果Watcher实例存在脏数据，则调用Watcher实例的`evaluate()`方法重新计算监听的值
    // 如果有Watcher实例（`Dep.target`）在收集依赖，则将`key`计算属性对应的Watcher实例的所有依赖，添加到`Dep.target`中。这样的好处是：Watcher实例都依赖的是`data`（或`inject`注入的属性）的Dep实例，计算属性不需要定义对应的Dep实例。
    // 返回`key`计算属性对应的Watcher实例的监听值。这一步即是，官方文当中“计算属性缓存”的意思。
  }
}
// 第三种：`$watch` API
function stateMixin() { // core/instance模块
  Vue.prototype.$watch = function() { // 初始化Vue实例的watch配置，也调用的是该API
    /*
      创建Watcher实例。Watcher实例相关参数说明如下：
      1、用户自定义`expOrFn`和`cb`
      2、可自定义Wacher实例的配置项
      3、该Watcher实例的`user`标志为true，即表示是用户自定义Watcher
    */
  }
}
```

## 4. 时间片

核心代码如下：

```javascript
/**
  * cb: 注册到时间片上的回调函数
  * ctx: cb的执行上下文
  */
function nextTick(cb, ctx) { // core/util模块
  // 用函数包装cb的执行过程，将该函数加入到回调队列中
  // 如果下一个时间片未创建，则调用`timerFunc()`函数创建之
  // 如果未传入cb，并且执行环境支持Promise，则返回Promise对象
}
function timerFunc() {
  // 如果执行环境原生支持Promise，则将`flushCallbacks()`作为Promise对象的`then`方法的回调。
  // 如果执行环境非IE，且原生支持MutationObserver，则用`timerFunc()`方法触发一个DOM节点的变化，以此触发MutationObserver对象执行其回调方法（`flushCallbacks()`函数作为其回调方法）。
  // 如果执行环境原生支持`setImmediate`，则将`flushCallbacks()`作为`setImmediate()`的回调函数，在下一个宏任务中执行
  // 其他情况，通过`setTimeout(flushCallbacks, 0)`，在下一个宏任务中执行`flushCallbacks()`
}
function flushCallbacks() {
  // 执行回调队列
}
```

其他资料：
* [MutationObserver](https://developer.mozilla.org/zh-CN/docs/Web/API/MutationObserver)
* [setImmediate](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/setImmediate)

## 5. VNode的Diff算法

```javascript
function createPatchFunction(backend) { // core/vdom模块
  return function patch(oldVnode, vnode, hydrating, removeOnly) {
    // 只有`vnode`（新的VNode），则调用`createElm()`创建对应的DOM树
    // 新旧VNode都不是DOM节点，且调用`sameVnode()`函数判断新旧VNode没有变化，调用`patchVnode()`函数更新vnode，并对比新旧VNode的子节点
    // 旧VNode是DOM节点，或新旧VNode发生变化，创建新VNode对应的DOM树
  
  }
}
function sameVnode(a, b) {
  /*
    判断VNode没有变化的条件：
    1. key一致 && ( 2 || 3 )
    2. tag一致 && 都是注释节点 && data属性一致 && 是相同的Input节点
    3. 旧VNode节点是异步占位节点 && 异步工厂方法相同 && 新VNode节点的异步工厂方法没有错误
  */
}
function createElm(vnode, insertedVnodeQueue, parentElm, refElm, nested, ownerArray, index) {
  // 如果`vnode`是组件的占位节点，则调用`createComponent()`同步组件实例与`vnode`的相关属性值
  // 如果是普通的VNode节点，则：
  /*
    1、调用DOM操作API，创建VNode对应的DOM节点
    2、调用`createChildren()`函数，创建`vnode`的子节点对应的DOM节点，并与`vnode`的DOM节点形成DOM树
    3、调用`vnode`的`create`、`insert`钩子
    4、将`vnode`的DOM树插入到其父节点的DOM节点的子节点列表中
  */
}
function createComponent(vnode, insertedVnodeQueue, parentElm, refElm) {
  // 根据`vnode.componentInstance`是否存在，判断`vnode`是否是组件的占位节点
  // 如果`vnode`对应的组件实例，被keepalive缓存，则激活之，并将对应的DOM树插入到父DOM节点的子节点列表中
  // 如果`vnode`对应的组件实例，则执行`vnode`的`create`、`insert`钩子（如果组件（及其子孙组件）的根节点是可以渲染成DOM节点的VNode节点），并将组件实例的根DOM节点作为`vnode`的DOM节点
}
function createChildren(vnode, children, insertedVnodeQueue) {
  // 如果子节点是列表，则调用`createElm()`函数创建数组元素的DOM树
  // 如果子节点是文本，则调用DOM操作API创建文本节点
}
function patchVnode(oldVnode, vnode, insertedVnodeQueue, ownerArray, index, removeOnly) {
  // 将旧vnode的DOM树同步到新vnode
  // 执行新vnode的`prepatch`钩子
  // 执行新vnode的`update`钩子（如果组件（及其子孙组件）的根节点是可以渲染成DOM节点的VNode节点）
  // 如果新vnode是文本节点，且存在变化，则更新新vnode的DOM节点的内容
  // 如果新vnode不是文本节点，则：
  /*
    1、如果新旧vnode的子节点存在，且不完全相等，则调用`updateChildren()`更新子节点
    2、如果新vnode存在子节点，旧vnode不存在子节点，则创建新vnode的子节点的DOM树
    3、如果旧vnode存在子节点，新vnode不存在子节点，则删除旧vnode的子节点及其DOM树，并执行相关VNode的`destory`钩子
    4、如果旧vnode是文本节点，则清空新vnode的DOM节点的内容
  */
  // 执行新vnode的`postpatch`钩子
}
function updateChildren(parentElm, oldCh, newCh, insertedVnodeQueue, removeOnly) {
  // 遍历新旧子节点列表，
  // 存在以`sameVnode()`函数判断的相同新旧VNode节点，则调用`patchVnode()`函数更新新VNode，并对比新旧VNode的子节点
  // 没有相同的旧VNode节点的新VNode节点，则调用`createElm()`函数创建其DOM树
  // 有多余的旧VNode节点，则删除之，并删除其DOM树，并执行其`destory`钩子
}
```

## 6. KeepAlive

`keep-alive`为抽象组件，在实例内部有`cache`和`keys`两个属性，分别存储被缓存的组件和组件的`key`。会根据组件的`include`和`exclude`属性决定组件是否被缓存。组件的`render`函数的功能如下：

1. 取组件默认插槽的第一个组件（插槽内只能有一个子元素被渲染）对应的VNode节点（组件的占位节点）
2. 取组件名，以及`include`和`exclude`属性决定是否缓存。如果不缓存，返回上述VNode节点即可
3. 取上述VNode节点的`key`，如果没有则根据上述VNode节点对应的实例`id`和`tag`生成`key`
4. 如果`key`已经被缓存，这取缓存的组件实例替换上述VNode节点的组件实例
5. 否则，缓存上述VNode节点，根据`max`判断是否需要清理旧的缓存
6. 置上述VNode节点的`keepAlive`属性为true
7. 返回上述VNode或插槽的第一个节点

VNode节点的`keepAlive`属性的使用场景：

1. 在创建VNode节点（占位节点）对应的组件实例时
   1. 执行VNode节点的`init`钩子
   2. 存在VNode节点对应的组件实例时，激活组件（执行组件的`activate`钩子，插入DOM树）
2. VNode的`init`钩子
   1. 如果组件实例未被销毁，且已缓存，则执行VNode的`prepatch`钩子（更新子组件实例）
   2. 否则，创建组件实例并挂载
3. VNode的`insert`钩子
   1. 如果VNode节点已缓存且已挂载，则给VNode节点对应的组件实例置活跃标志，加入到活跃组件队列，该队列会在下一个时钟周期激活组件树（递归调用，置组件实例激活标志，调用`activated`钩子）
   2. 如果VNode节点已缓存且未挂载，激活组件树。
4. VNode的`destroy`钩子
   1. 如果组件实例未挂载且VNode已缓存，使组件树失活（递归调用，置组件实例失活标志，调用`deactivated`钩子）

## 7. 服务端渲染

服务端渲染涉及到的：vue源码的server模块、vue源码目标平台的server模块、vue源码中的SSR的webpack插件（ClientPlugin和ServerPlugin）、vue-loader、vue-style-loader

SSR的流程：

1. 业务项目的SSRClient端的webpack入口JS脚本 => webpack + `vue-loader` + `vue-style-loader` => webpack + `VueSSRClientPlugin` => 生成编译后的项目文件 + `vue-ssr-client-manifest.json`
2. 业务项目的SSRServer端的webpack入口JS脚本 => webpack + `vue-loader` + `vue-style-loader` => webpack + `VueSSRServerPlugin` => `vue-ssr-server-bundle.json`
3. Node Server脚本提供服务 => `vue-server-renderer`的`createBundleRenderer` API + `vue-ssr-server-bundle.json` + `vue-ssr-client-manifest.json` 完成服务端渲染
4. 浏览器加载Node Server返回的渲染结果，运行后由Client Vue接管渲染

以[Vue SSR官网](https://ssr.vuejs.org/zh/)的指南为例对上述过程做如下说明：

### 7.1. 业务项目的SSRClient端的webpack入口JS脚本

主要流程：

1. 创建Vue实例、VueRouter实例、Vuex实例
2. 根据服务端渲染出的State（从`window.__INITIAL_STATE__`访问）初始化Store
3. 路由加载完成后，添加Router的`beforeResolve`钩子处理下一个组件加载前的异步数据获取，挂载Vue实例
4. 用Vue mixin插入Router的`beforeRouteUpdate`钩子处理组件复用时的异步数据获取

### 7.2. 业务项目的SSRServer端的webpack入口JS脚本

主要流程：

1. 默认导出`(context) => { return new Promise((resolve, reject) => { ...resolve(app)  }) }`函数
   1. 传入的`context`是渲染上下文对象，在Node Server脚本调用`vue-server-renderer`的渲染对象（见下文）的`renderToString` API时传入，会在该API中经过一些处理，添加一些属性和方法
   2. 必须返回一个Promise对象
   3. Promise对象的完成状态必须传入待渲染的Vue实例（`app`）
2. Promise传入的函数内的主要流程：
   1. 创建Vue实例、VueRouter实例、Vuex实例
   2. 从`context`中获取客户端向Node Server请求的页面地址，并将改地址设置为VueRouter实例的地址
   3. 当VueRouter实例加载完成，调用其`getMatchedComponents` API获取匹配到的组件
   4. 调用匹配到的组件的获取异步数据的方法，在该方法内会将获取到的远程数据保存到Store里
   5. 将Store中的`state`插入到`context`中的`state`属性，`vue-server-renderer`会将`context`中的`state`的值插入到`window.__INITIAL_STATE__`中

### 7.3. `vue-loader`和`vue-style-loader`

`vue-loader`在处理各模块代码的过程中，会插入一段函数，该函数的作用是以`__VUE_SSR_CONTEXT__`为上下文对象，并且调用该对象的`_registeredComponents` API保存当前正在处理的模块的编号（模块地址的hash值）。详见`vue-router`的`lib/runtime/componentNormalizer.js`和`lib/index.js`文件。

`vue-style-loader`在处理CSS模块代码的过程中，会插入一段函数，该函数的作用是以`__VUE_SSR_CONTEXT__`为上下文对象，将CSS文件的内容装换成对象，存放到上下文对象的`_styles`属性中。详见`vue-style-loader`的`lib/addStylesServer.js`和`index.js`文件。

### 7.4. `VueSSRClientPlugin` webpack插件

该插件在Vue源码的`packages/vue-server-renderer/client-plugin.js`文件中实现。

插件的主要功能：

1. 注册webpack的`emit`钩子，该钩子会在输出 asset 到 output 目录之前执行
2. 该钩子的执行函数的主要作用：
   1. 将 webpack 编译的资源的名称提取分类到JSON对象
   2. 维护“模块的编号”（在`vue-loader`中通过`__VUE_SSR_CONTEXT__._registeredComponents` API注册的模块编号）与相关资源的映射关系
   3. 将SSR Client需要输出的JSON文件（默认是`vue-ssr-client-manifest.json`文件）插入到资源列表中

### 7.5. `VueSSRServerPlugin` webpack插件

该插件在Vue源码的`packages/vue-server-renderer/server-plugin.js`文件中实现。

插件的主要功能：

1. 注册webpack的`emit`钩子，该钩子会在输出 asset 到 output 目录之前执行
2. 该钩子的执行函数的主要作用：
   1. 将 webpack 编译好的资源提取到JSON对象
   2. 删除资源列表内的内容
   3. 添加SSR Server需要输出的JSON文件（默认是`vue-ssr-server-bundle.json`文件）插入到资源列表中

### 7.6. `vue-ssr-client-manifest.json`文件

文件的内容：

```javascript
{
  "publicPath": "/", // webpack的output.publicPath
  "all": [ // webpack处理的所有资源
    "css/app.*****.css",
    "css/app.*****.css.map",
    "index.html",
    "js/app.*****.js",
    "js/app.*****.js.map",
    ...
  ],
  "initial": [ // 入口js或css资源
    "css/app.*****.css",
    "js/app.*****.js",
    ...
  ],
  "async": [ // 非入口js或css资源
    "css/chunk-*****.*****.css",
    "js/chunk-*****.*****.js",
    ...
  ],
  "modules": { // 模块编号到编译后的资源在“all”中的索引（多个）的映射关系
    "********": [
      1, // “all”的索引值
      13,
      ...
    ],
    ...
  }
}
```

### 7.7. `vue-ssr-server-bundle.json`文件

文件的内容：

```javascript
{
  "entry": "js/app.*****.js", // 入口文件
  "files": { // 所有资源文件到模块编译后的内容的映射
    "js/app.*****.js": "module.exports=function(t){...}", // 这个模块就是上文的“业务项目的SSRServer端的webpack入口JS脚本”经webpack编译生成的
    "js/chunk-*****.*****.js": "exports.***=...",
    ...
  }
  "maps": { // 资源文件的SourceMap
    "js/app.*****.js": {/* 该对象是SourceMap对象 */},
    "js/chunk-*****.*****.js": {/* 该对象是SourceMap对象 */},
    ...
  }
}
```

### 7.8. `vue-server-renderer`的入口文件

以Web平台为例，`vue-server-renderer`的入口文件为`platforms/web/entry-server-render.js`。该模块导出两个函数：`createRender`和`createBundleRenderer`。

### 7.9. `vue-server-renderer`的`createBundleRenderer` API

`createBundleRenderer` API是在入口文件内，调用`server/bundle-renderer/create-bundle-renderer.js`的`createBundleRendererCreator`函数，入参为`createRender`。

`createBundleRendererCreator`函数核心代码如下：

```javascript
function createBundleRendererCreator(createRenderer) { // `createRenderer` 为`vue-server-renderer`的`createRenderer` API
  return function createBundleRenderer(bundle, rendererOptions) { // `bundle`参数一般传入`vue-ssr-server-bundle.json`；`rendererOptions`作为`createRenderer`函数的实参传入
    // 读取通过`bundle`参数传入的JSON文件
    // 调用 `createRenderer` 函数，创建渲染对象（`renderer`对象）
    // 调用 `createBundleRunner` 函数，创建一个可以执行`bundle`参数传入的JSON文件的`entry`模块的导出函数的执行器（`run`函数）
    return {
      renderToString: (context, cb) => { // context 为渲染上下文对象
        // 处理只传入一个参数（参数类型为函数）的情况，此时，传入的参数为cb
        // 处理没有传入cb的情况，该函数返回Promise实例
        run(context).then(app => { // 执行上文的执行器，该执行器实际是“业务项目的SSRServer端的webpack入口JS脚本”导出的函数
          if (app) { // app为“业务项目的SSRServer端的webpack入口JS脚本”内创建Vue实例
            renderer.renderToString(app, context, (err, res) => { /* 执行cb结束 */ }) // 完成渲染，通过cb回调函数传出渲染结果
          }
        })
      },
      renderToStream: (context) => {
        const res = new PassThrough() // 转换流，用于将文本输出
        run(context).then(app => {
          if(app) {
            const renderStream = renderer.renderToStream(app, context) // 渲染，获得渲染结果的文件流
            // 当有模板存在时，做renderStream和res之间的事件传递
            renderStream.pipe(res)
          }
        })
        return res
      }
    }
  }
}
function createBundleRunner(entry, files, basedir, runInNewContext) { // entry入口文件的文件名，files为文件名到文件内容的映射
  const evaluate = compileModule(files, basedir, runInNewContext) // 创建一个可以获得files中指定文件（模块）的导出结果的函数
  // 根据runInNewContext参数判断，如果在新的执行上下文中执行模块
  return (userContext) => new Promise(resolve => {
    userContext._registeredComponents = new Set() // 在entry模块执行时，会插入数据。在vue-loader做代码转换时，将_registeredComponents的相关访问插入到enty模块中
    const res = evaluate(entry, createSandbox(userContext)) // 在新的执行上下文对象下，执行entry模块，获取模块的导出结果。createSandbox函数用来创建新的执行上下文对象
    resolve(typeof res === 'function' ? res(userContext) : res) // 如果entry模块的导出结果为函数，则执行之
  })
  // 如果不是在新的执行上下文执行模块
  let runner
  return (userContext) => new Promise(resolve => {
    if(runner) {
      const sandbox = runInNewContext === 'once' ? createSandbox() : global // 获取执行上下文对象，即runner每次执行时都用同一个执行上下文对象
      runner = evaluate(entry, sandbox) // 执行entry模块，获取模块的导出结果（导出结果为“业务项目的SSRServer端的webpack入口JS脚本”导出的函数）
      // 当runner不是函数时，抛异常
    }
    userContext._registeredComponents = new Set()
    // 处理一些entry模块内的样式。由vue-style-loader注入的
    resolve(runner(userContext))
  })
}
function compileModule(files, basedir, runInNewContext) { // files为文件名到文件内容的映射
  return (filename, sandbox, evaluatedFiles) => { // filename为要编译的模块的文件名；sandbox为执行上下文
    const wrapper = NativeModule.wrap(files[filename]) // 将filename对应的文件内容用'(function (exports, require, module, __filename, __dirname) { ' 和 '\n});' 包裹。文件内容为：“module.exports=function(t){...}”或“exports.ids=[...]...”
    const script = new vm.Script(wrapper, { // 编译模块，但不执行
      filename,
      displayErrors: true
    })
    const compiledWrapper = runInNewContext === false ? script.runInThisContext() : script.runInNewContext(sandbox) // 执行模块，compiledWrapper是函数function(exports, require, module, __filename, __dirname){ ... }
    const m = { exports: {}} // 模块的导出结果
    compiledWrapper.call(m.exports, m.exports, file => { ... return require(file) }, m) // 执行函数function(exports, require, module, __filename, __dirname){ ... }
    return Object.prototype.hasOwnProperty.call(m.exports, 'default') ? m.exports.default : m.exports // 如果模块有默认导出，则返回之，否则返回导出对象
  }
}
```

### 7.10. `vue-server-renderer`的`createRenderer` API

`createRenderer` API是在入口文件内，调用`server/create-renderer.js`的`createRenderer`函数，传入用户自定义的配置和平台适配的配置（如模块、指令等）。

`server/create-renderer.js`的`createRenderer`函数的核心代码：

```javascript
/**
 * 创建renderer对象
 * @param {RenderOptions} options 配置对象
 * @returns renderer对象
 */
function createRenderer({ 
  modules, // vue模块
  directives, // vue指令
  isUnaryTag, // ()=> boolean，判断传入的节点名称是否为一元节点
  template, // 模板
  inject, // 是否给模板注入除插值和Vue实例渲染结果之外的内容，如CSS节点、JS节点
  cache, // 缓存对象，需要包含get和has方法
  shouldPreload, // (file: string, type: string) => boolean，判断指定文件是否需要作为`<link rel="preload" ...>`预加载资源（需要解析的）插入到模板中
  shouldPrefetch, // (file: string, type: string) => boolean，判断指定文件是否需要作为`<link rel="prefetch" ...>`预取资源（缓存资源，不需要解析）插入到模板中
  clientManifest, // `vue-ssr-client-manifest.json`文件
  serializer // 渲染上下文对象的`state`属性（默认是`state`属性）的值的序列化方法。默认使用serialize-javascript的序列化方法
}) {
  const render = createRenderFunction(modules, directives, isUnaryTag, cache) // 创建render方法
  const templateRenderer = new TemplateRenderer({...})  // 创建HTML模板渲染对象实例，用于将component渲染出的DOM字符串插入到HTMl模板（template）中，并生成HTML中的其他部分，如<head>、css、js节点等
  return {
    renderToString(component, context, cb) { // 将component渲染成为完整的HTML文件内容。component为Vue实例；context为渲染上下文，cb为回调函数
      // 处理只传入一个参数（参数类型为函数）的情况，此时，传入的参数为cb
      // 将templateRenderer的renderResourceHints、renderState、renderScripts、renderStyles、getPreloadFiles绑定所属对象（templateRenderer）后，植入到context中
      // 处理没有传入cb的情况，该函数返回Promise实例
      let result = '' // 渲染结果
      render(component, text => { result += text }, context, err => { // 渲染component，每一部分渲染完成后执行第二个参数，全部渲染完后执行第四个参数
        templateRenderer.render(result, context).then(html => cb(null, html)) // 将component的渲染结果插入到模板中，并在模板中插入其他资源，如head、css、js等。html为最终渲染完成的html文件内容
      })
    },
    renderToStream(component, context) {
      // 将templateRenderer的renderResourceHints、renderState、renderScripts、renderStyles、getPreloadFiles绑定所属对象（templateRenderer）后，植入到context中
      const renderStream = new RenderStream((write, done) => { // 创建可读取渲染结果的流对象。RenderSteam类继承自stream.Readable
        render(component, write, context, done) // 渲染component，每一部分渲染完成后执行第二个参数，全部渲染完后执行第四个参数
      })
      // 如果没有模板
      return renderStream // 返回可以读取component渲染结果的流对象
      // 如果有模板
      const templateStream = templateRenderer.createStream(context) // 创建模板的转换流对象
      renderStream.pipe(templateStream) // 将component渲染结果传入到模板的渲染流中
      return templateStream // 返回可以读取完整的HTML文件内容的流对象
    }
  }
}
function createRenderFunction(modules, directives, isUnaryTag, cache) {
  return function render(component, write, userContext, done) { // component为待渲染的Vue实例
    // 创建RenderContext实例`context`，作为渲染上下文。其中`write()`方法为`write`参数
    // 向Vue原型对象添加以`_ssr`开头的方法
    // 标准化处理component的`render()`方法，使用`web/server/compiler.js`中的`compileToFunctions()`函数创建`render()`函数
    // 执行component中配置的serverPrefetch钩子
    // 执行component的`_render()`方法，生成VNode树
    // 执行`renderNode(VNode树, true, context)`函数渲染VNode树
  }
}
class TemplateRenderer {
  render() {
    // 已注入方式渲染：
    /*
      1. 编译模板的</head>前的部分，使用渲染上下文对象填充
      2. 插入渲染上下文中的head的内容
      3. 添加preload和prelink的资源的节点
      4. 添加样式节点
      5. 编译模板的</head>到<!--vue-ssr-outlet-->前的部分
      6. 插入vue实例的渲染结果
      7. 插入state的渲染结果
      8. 插入js节点
      9. 编译模板的<!--vue-ssr-outlet-->之后的部分
      注：第1、5、9步的编译，使用的是lodash的template api，实现了向模板中插值的功能
     */
    // 非注入方式渲染，只包含上述步骤的1、5、6、9
  }
}
function renderNode(node, isRoot, context) {
  // 渲染VNode树（`node`），节点为字符串、组件、HTML节点、注释节点、其他。通过`context.write()`方法返回渲染结果。`context.next()`为`context.write()`的回调函数，待渲染结果处理完之后执行。
}
```

## 8. 目录结构

```javascript
vue                                                   
├─ benchmarks                                         
├─ examples                                           
├─ flow                                               
├─ packages                                           
│  ├─ vue-server-renderer                      // SSR的webpack插件    
│  │  ├─ client-plugin.js                      // SSR的webpack插件 - VueSSRClientPlugin插件  
│  │  └─ server-plugin.js                      // SSR的webpack插件 - VueSSRServerPlugin插件    
│  ├─ vue-template-compiler                           
│  ├─ weex-template-compiler                          
│  └─ weex-vue-framework                              
├─ scripts                                     // 项目脚本       
├─ src                                                
│  ├─ compiler                                 // 模板编译模块       
│  │  ├─ codegen                               // 根据AST树生成Render函数体       
│  │  ├─ directives                            // 指令解析       
│  │  ├─ parser                                // 指令的表达式处理       
│  │  └─ index.js                              // 编译模块的入口，开放编译器的工厂函数       
│  ├─ core                                     // 核心模块       
│  │  ├─ components                            // 组件，只有KeepAlive       
│  │  ├─ global-api                            // Vue的静态API       
│  │  ├─ instance                              // Vue实例       
│  │  ├─ observer                              // 响应式系统       
│  │  ├─ util                                  // 工具函数       
│  │  ├─ vdom                                  // 虚拟节点       
│  │  └─ index.js                              // 核心模块入口，开放Vue类       
│  ├─ platforms                                // 平台适配       
│  │  ├─ web                                   // web平台       
│  │  │  ├─ compiler                           // web平台-模板编译模块       
│  │  │  │  ├─ directives                             
│  │  │  │  ├─ modules                                
│  │  │  │  └─ index.js                                
│  │  │  ├─ runtime                            // web平台-运行时       
│  │  │  │  ├─ components                             
│  │  │  │  ├─ directives                             
│  │  │  │  ├─ modules                                
│  │  │  │  └─ index.js                     
│  │  │  ├─ server                             // web平台-SSR       
│  │  │  │  ├─ directives                             
│  │  │  │  ├─ modules                                
│  │  │  │  ├─ compiler.js                            
│  │  │  │  └─ util.js                                
│  │  │  ├─ util                               // web平台-工具函数       
│  │  │  ├─ entry-compiler.js                  // web平台-项目编译入口，编译模块       
│  │  │  ├─ entry-runtime-with-compiler.js     // web平台-项目编译入口，运行时+编译器模块       
│  │  │  ├─ entry-runtime.js                   // web平台-项目编译入口，运行时模块       
│  │  │  ├─ entry-server-basic-renderer.js     // web平台-项目编译入口，服务端渲染基础模块       
│  │  │  └─ entry-server-renderer.js           // web平台-项目编译入口，服务端渲染模块       
│  │  └─ weex                                  // weex平台       
│  │     ├─ compiler                                  
│  │     │  ├─ directives                             
│  │     │  ├─ modules                                
│  │     │  └─ index.js                               
│  │     ├─ runtime                                   
│  │     │  ├─ components                             
│  │     │  ├─ directives                             
│  │     │  ├─ modules                                
│  │     │  ├─ recycle-list                           
│  │     │  └─ index.js                           
│  │     ├─ util                                      
│  │     └─ *.js                  
│  ├─ server                                   // 服务端渲染       
│  ├─ sfc                                             
│  └─ shared                                   // 工具函数       
├─ test                                        // 测试模块       
│  ├─ e2e                                             
│  ├─ helpers                                         
│  ├─ ssr                                             
│  ├─ unit                                            
│  └─ weex                                            
└─ types                                       // TS类型   
```
