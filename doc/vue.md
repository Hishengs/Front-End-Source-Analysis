## 版本说明
使用版本：[2.4.4](https://github.com/vuejs/vue/releases/tag/v2.2.4)

## 从 vue 实例讲起
由于 vue 整个框架目前包含的东西越来越多，很难剖析全面，我们先从最简单的 vue 实例讲起，之后再涉及其他方面。

### Vue 构造函数

vue 的构造函数定义在 `src/core/instance/index.js`

```js
import { initMixin } from './init'
import { stateMixin } from './state'
import { renderMixin } from './render'
import { eventsMixin } from './events'
import { lifecycleMixin } from './lifecycle'
import { warn } from '../util/index'

function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    // 如果未通过 new 实例化，会直接报错，`this._init` 不会被执行
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  // 在这个函数里，会分别调用 `init, state, render, event, lifecycle` 模块的初始化函数
  // 这个函数在下面的 initMixn 时被定义
  this._init(options) // 在使用构造函数实例化时调用
}

initMixin(Vue)
stateMixin(Vue)
eventsMixin(Vue)
lifecycleMixin(Vue)
renderMixin(Vue)

export default Vue

```
在实例化 Vue 时，即 `new Vue()` 时内部会调用 `this._init` 方法执行各个模块的初始化方法。

### initMixin

先看 `initMixin` 这个函数做了什么事情，打开 `src/core/instance/init/index.js`:
```js
// 暂时忽略其他代码，关注这个函数

export function initMixin (Vue: Class<Component>) {
  // 在这个函数里只做了一件事，就是实例化 vue 时唯一调用的函数：this._init
  Vue.prototype._init = function (options?: Object) {
    // 注意，在所有 vue 源码中，vm 代表当前的 vue 实例
    const vm: Component = this
    // a uid
    vm._uid = uid++ // 区分 vue 实例的标识 ID？？

    // 非生产环境版本的性能测试
    let startTag, endTag
    /* istanbul ignore if */
    if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
      startTag = `vue-perf-init:${vm._uid}`
      endTag = `vue-perf-end:${vm._uid}`
      mark(startTag)
    }

    // a flag to avoid this being observed
    // 用于标识该实例是否已被观察（？）
    vm._isVue = true
    // merge options
    if (options && options._isComponent) { // _isComponent 代表该实例是组件
      // optimize internal component instantiation
      // since dynamic options merging is pretty slow, and none of the
      // internal component options needs special treatment.
      initInternalComponent(vm, options)
    } else {
      vm.$options = mergeOptions(
        resolveConstructorOptions(vm.constructor),
        options || {},
        vm
      )
    }
    /* istanbul ignore else */
    if (process.env.NODE_ENV !== 'production') {
      initProxy(vm)
    } else {
      vm._renderProxy = vm
    }
    // expose real self
    vm._self = vm       // 暂时看不出有什么用
    // 以下这几个函数才是重点
    initLifecycle(vm)   // 初始化生命周期
    initEvents(vm)      // 初始化事件
    initRender(vm)      // 初始化渲染
    callHook(vm, 'beforeCreate') // 触发 beforeCreate 周期
    initInjections(vm) // resolve injections before data/props 看不出（？）
    initState(vm)       // 初始化 state
    initProvide(vm) // resolve provide after data/props 看不出（？）
    callHook(vm, 'created')     // 触发 created 周期

    /* istanbul ignore if */
    if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
      vm._name = formatComponentName(vm, false)
      mark(endTag)
      measure(`${vm._name} init`, startTag, endTag)
    }

    if (vm.$options.el) { // 如果指定了挂载的元素，则调用挂载函数
      vm.$mount(vm.$options.el)
    }
  }
}
```
在这个文件中，主要定义了 vue 实例中调用的 `this_.init` 方法，在这个方法中，重点在于以下几个函数：
```js
initLifecycle(vm)   // 初始化生命周期
initEvents(vm)      // 初始化事件
initRender(vm)      // 初始化渲染
callHook(vm, 'beforeCreate') // 触发 beforeCreate 周期
initInjections(vm) // resolve injections before data/props 看不出（？）
initState(vm)       // 初始化数据
initProvide(vm) // resolve provide after data/props 看不出（？）
callHook(vm, 'created')     // 触发 created 周期
```
我们接着依次进入这几个函数探究一番。

#### initLifecycle
`initLifecycle`，定义在 `src/core/instance/lifecycle.js`：
```js
// 暂时忽略其他代码
export function initLifecycle (vm: Component) {
  const options = vm.$options

  // locate first non-abstract parent
  // 找到最上层的父组件
  let parent = options.parent
  if (parent && !options.abstract) {
    while (parent.$options.abstract && parent.$parent) {
      parent = parent.$parent
    }
    parent.$children.push(vm)
  }

  vm.$parent = parent
  vm.$root = parent ? parent.$root : vm // 根实例

  vm.$children = []
  vm.$refs = {}

  vm._watcher = null
  vm._inactive = null
  vm._directInactive = false
  vm._isMounted = false // 是否已挂载
  vm._isDestroyed = false // 实例是否已销毁
  vm._isBeingDestroyed = false // 实例是否正在销毁
}
```

#### initEvents

`initEvents`，定义在 `src/core/instance/events.js`：
```js
export function initEvents (vm: Component) {
  // 创建一个完全无原型链的空对象，用于存在该实例所有的事件
  vm._events = Object.create(null)
  vm._hasHookEvent = false
  // init parent attached events
  // 父组件的相关事件监听
  const listeners = vm.$options._parentListeners
  if (listeners) {
    updateComponentListeners(vm, listeners)
  }
}
```

#### initRender

`initRender`，定义在 `src/core/instance/render.js`：
```js
export function initRender (vm: Component) {
  vm._vnode = null // the root of the child tree
  vm._staticTrees = null
  const parentVnode = vm.$vnode = vm.$options._parentVnode // the placeholder node in parent tree
  const renderContext = parentVnode && parentVnode.context
  vm.$slots = resolveSlots(vm.$options._renderChildren, renderContext)
  vm.$scopedSlots = emptyObject
  // bind the createElement fn to this instance
  // so that we get proper render context inside it.
  // args order: tag, data, children, normalizationType, alwaysNormalize
  // internal version is used by render functions compiled from templates
  vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)
  // normalization is always applied for the public version, used in
  // user-written render functions.
  vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true)

  // $attrs & $listeners are exposed for easier HOC creation.
  // they need to be reactive so that HOCs using them are always updated
  const parentData = parentVnode && parentVnode.data

  /* istanbul ignore else */
  if (process.env.NODE_ENV !== 'production') {
    defineReactive(vm, '$attrs', parentData && parentData.attrs || emptyObject, () => {
      !isUpdatingChildComponent && warn(`$attrs is readonly.`, vm)
    }, true)
    defineReactive(vm, '$listeners', vm.$options._parentListeners || emptyObject, () => {
      !isUpdatingChildComponent && warn(`$listeners is readonly.`, vm)
    }, true)
  } else {
    defineReactive(vm, '$attrs', parentData && parentData.attrs || emptyObject, null, true)
    defineReactive(vm, '$listeners', vm.$options._parentListeners || emptyObject, null, true)
  }
}
```

#### initState

`initState`，定义在 `src/core/instance/state.js`：
```js
export function initState (vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  // 如果组件定义了 props，则调用 props 的初始化工作
  if (opts.props) initProps(vm, opts.props)
  // 如果组件定义了 methods，则调用 methods 的初始化工作
  if (opts.methods) initMethods(vm, opts.methods)
  // 如果组件定义了 data，则调用 data 的初始化工作，如果没有，则将 data 设置为一个空对象
  if (opts.data) {
    initData(vm)
  } else {
    observe(vm._data = {}, true /* asRootData */)
  }
  // 如果组件定义了 computed，则调用 computed 的初始化工作
  if (opts.computed) initComputed(vm, opts.computed)
  // 如果组件定义了 watch，则调用 watch 的初始化工作
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
}
```

在这个函数中，又引申出了我们在使用 vue 实例时经常定义的一些属性及方法的初始化工作，包括：
> 1. data
2. props
3. computed
4. watch
5. methods

##### data > initData
```js
function initData (vm: Component) {
  let data = vm.$options.data
  // 这里对 data 的类型进行判断
  // 如果是函数，调用 getData 执行 data.call(vm) 得到返回的对象
  // 否则直接返回 data 或者 空对象
  data = vm._data = typeof data === 'function'
    ? getData(data, vm)
    : data || {}
  // 如果得到的 data 不是纯粹的对象，则直接将一个空的对象赋值给 data
  if (!isPlainObject(data)) {
    data = {}
    process.env.NODE_ENV !== 'production' && warn(
      'data functions should return an object:\n' +
      'https://vuejs.org/v2/guide/components.html#data-Must-Be-a-Function',
      vm
    )
  }
  // proxy data on instance
  const keys = Object.keys(data)
  const props = vm.$options.props
  const methods = vm.$options.methods
  let i = keys.length
  while (i--) {
    const key = keys[i]
    if (process.env.NODE_ENV !== 'production') {
      // data 是否与 methods 存在相同的键值
      if (methods && hasOwn(methods, key)) {
        warn(
          `Method "${key}" has already been defined as a data property.`,
          vm
        )
      }
    }
    if (props && hasOwn(props, key)) {
      // data 是否与 props 存在相同的键值
      process.env.NODE_ENV !== 'production' && warn(
        `The data property "${key}" is already declared as a prop. ` +
        `Use prop default value instead.`,
        vm
      )
    } else if (!isReserved(key)) { // 是否是保留的键值
      proxy(vm, `_data`, key)
    }
  }
  // observe data
  // 这一步很关键，实现对 data 数据的观察，也就是通过 getter/setter 监听数据变动，从而收集依赖，触发视图更新等
  observe(data, true /* asRootData */)
}
```

##### props > initProps
```js
function initProps (vm: Component, propsOptions: Object) {
  const propsData = vm.$options.propsData || {}
  const props = vm._props = {}
  // cache prop keys so that future props updates can iterate using Array
  // instead of dynamic object key enumeration.
  const keys = vm.$options._propKeys = []
  const isRoot = !vm.$parent
  // root instance props should be converted
  observerState.shouldConvert = isRoot
  for (const key in propsOptions) {
    keys.push(key)
    // 这里会对 props 的值进行检验，返回或者设置正确的默认值
    const value = validateProp(key, propsOptions, propsData, vm)
    /* istanbul ignore else */
    if (process.env.NODE_ENV !== 'production') {
      if (isReservedAttribute(key) || config.isReservedAttr(key)) {
        warn(
          `"${key}" is a reserved attribute and cannot be used as component prop.`,
          vm
        )
      }
      defineReactive(props, key, value, () => {
        if (vm.$parent && !isUpdatingChildComponent) {
          warn(
            `Avoid mutating a prop directly since the value will be ` +
            `overwritten whenever the parent component re-renders. ` +
            `Instead, use a data or computed property based on the prop's ` +
            `value. Prop being mutated: "${key}"`,
            vm
          )
        }
      })
    } else {
      defineReactive(props, key, value)
    }
    // static props are already proxied on the component's prototype
    // during Vue.extend(). We only need to proxy props defined at
    // instantiation here.
    if (!(key in vm)) {
      proxy(vm, `_props`, key)
    }
  }
  observerState.shouldConvert = true
}
```

##### computed > initComputed
```js
function initComputed (vm: Component, computed: Object) {
  // 这里 checkOptionType 方法主要用于检测实例下的配置项是否是对象
  process.env.NODE_ENV !== 'production' && checkOptionType(vm, 'computed')
  const watchers = vm._computedWatchers = Object.create(null)
  // computed properties are just getters during SSR
  const isSSR = isServerRendering()

  for (const key in computed) {
    const userDef = computed[key]
    const getter = typeof userDef === 'function' ? userDef : userDef.get
    if (process.env.NODE_ENV !== 'production' && getter == null) {
      warn(
        `Getter is missing for computed property "${key}".`,
        vm
      )
    }

    if (!isSSR) {
      // create internal watcher for the computed property.
      watchers[key] = new Watcher(
        vm,
        getter || noop,
        noop,
        computedWatcherOptions
      )
    }

    // component-defined computed properties are already defined on the
    // component prototype. We only need to define computed properties defined
    // at instantiation here.
    if (!(key in vm)) {
      defineComputed(vm, key, userDef)
    } else if (process.env.NODE_ENV !== 'production') {
      // 检查键值是否已在 data 和 props 中已定义过
      if (key in vm.$data) {
        warn(`The computed property "${key}" is already defined in data.`, vm)
      } else if (vm.$options.props && key in vm.$options.props) {
        warn(`The computed property "${key}" is already defined as a prop.`, vm)
      }
    }
  }
}
```

##### watch > initWatch
```js
function initWatch (vm: Component, watch: Object) {
  process.env.NODE_ENV !== 'production' && checkOptionType(vm, 'watch')
  for (const key in watch) {
    const handler = watch[key]
    if (Array.isArray(handler)) {
      for (let i = 0; i < handler.length; i++) {
        createWatcher(vm, key, handler[i])
      }
    } else {
      createWatcher(vm, key, handler)
    }
  }
}
```

##### methods > initMethods
```js
function initMethods (vm: Component, methods: Object) {
  process.env.NODE_ENV !== 'production' && checkOptionType(vm, 'methods')
  const props = vm.$options.props
  for (const key in methods) {
    if (process.env.NODE_ENV !== 'production') {
      if (methods[key] == null) {
        warn(
          `Method "${key}" has an undefined value in the component definition. ` +
          `Did you reference the function correctly?`,
          vm
        )
      }
      if (props && hasOwn(props, key)) {
        warn(
          `Method "${key}" has already been defined as a prop.`,
          vm
        )
      }
      if ((key in vm) && isReserved(key)) {
        warn(
          `Method "${key}" conflicts with an existing Vue instance method. ` +
          `Avoid defining component methods that start with _ or $.`
        )
      }
    }
    // 对每个方法进行 this 绑定，context => vm
    vm[key] = methods[key] == null ? noop : bind(methods[key], vm)
  }
}
```



### stateMixin
```js
export function stateMixin (Vue: Class<Component>) {
  // flow somehow has problems with directly declared definition object
  // when using Object.defineProperty, so we have to procedurally build up
  // the object here.
  const dataDef = {}
  dataDef.get = function () { return this._data }
  const propsDef = {}
  propsDef.get = function () { return this._props }
  if (process.env.NODE_ENV !== 'production') {
    dataDef.set = function (newData: Object) {
      warn(
        'Avoid replacing instance root $data. ' +
        'Use nested data properties instead.',
        this
      )
    }
    propsDef.set = function () {
      warn(`$props is readonly.`, this)
    }
  }
  Object.defineProperty(Vue.prototype, '$data', dataDef)
  Object.defineProperty(Vue.prototype, '$props', propsDef)

  Vue.prototype.$set = set
  Vue.prototype.$delete = del

  Vue.prototype.$watch = function (
    expOrFn: string | Function,
    cb: any,
    options?: Object
  ): Function {
    const vm: Component = this
    if (isPlainObject(cb)) {
      return createWatcher(vm, expOrFn, cb, options)
    }
    options = options || {}
    options.user = true
    const watcher = new Watcher(vm, expOrFn, cb, options)
    if (options.immediate) {
      cb.call(vm, watcher.value)
    }
    return function unwatchFn () {
      watcher.teardown()
    }
  }
}
```

### eventsMixin
这个函数主要定义了 Vue 事件相关的方法：`$on`, `$emit`, `$once`, `$off`。
```js
export function eventsMixin (Vue: Class<Component>) {
  const hookRE = /^hook:/
  Vue.prototype.$on = function (event: string | Array<string>, fn: Function): Component {
    const vm: Component = this
    if (Array.isArray(event)) {
      for (let i = 0, l = event.length; i < l; i++) {
        this.$on(event[i], fn)
      }
    } else {
      (vm._events[event] || (vm._events[event] = [])).push(fn)
      // optimize hook:event cost by using a boolean flag marked at registration
      // instead of a hash lookup
      if (hookRE.test(event)) {
        vm._hasHookEvent = true
      }
    }
    return vm
  }

  Vue.prototype.$once = function (event: string, fn: Function): Component {
    const vm: Component = this
    function on () {
      vm.$off(event, on)
      fn.apply(vm, arguments)
    }
    on.fn = fn
    vm.$on(event, on)
    return vm
  }

  Vue.prototype.$off = function (event?: string | Array<string>, fn?: Function): Component {
    const vm: Component = this
    // all
    if (!arguments.length) {
      vm._events = Object.create(null)
      return vm
    }
    // array of events
    if (Array.isArray(event)) {
      for (let i = 0, l = event.length; i < l; i++) {
        this.$off(event[i], fn)
      }
      return vm
    }
    // specific event
    const cbs = vm._events[event]
    if (!cbs) {
      return vm
    }
    if (arguments.length === 1) {
      vm._events[event] = null
      return vm
    }
    if (fn) {
      // specific handler
      let cb
      let i = cbs.length
      while (i--) {
        cb = cbs[i]
        if (cb === fn || cb.fn === fn) {
          cbs.splice(i, 1)
          break
        }
      }
    }
    return vm
  }

  Vue.prototype.$emit = function (event: string): Component {
    const vm: Component = this
    if (process.env.NODE_ENV !== 'production') {
      const lowerCaseEvent = event.toLowerCase()
      if (lowerCaseEvent !== event && vm._events[lowerCaseEvent]) {
        tip(
          `Event "${lowerCaseEvent}" is emitted in component ` +
          `${formatComponentName(vm)} but the handler is registered for "${event}". ` +
          `Note that HTML attributes are case-insensitive and you cannot use ` +
          `v-on to listen to camelCase events when using in-DOM templates. ` +
          `You should probably use "${hyphenate(event)}" instead of "${event}".`
        )
      }
    }
    let cbs = vm._events[event]
    if (cbs) {
      cbs = cbs.length > 1 ? toArray(cbs) : cbs
      const args = toArray(arguments, 1)
      for (let i = 0, l = cbs.length; i < l; i++) {
        try {
          cbs[i].apply(vm, args)
        } catch (e) {
          handleError(e, vm, `event handler for "${event}"`)
        }
      }
    }
    return vm
  }
}
```

### lifecycleMixin
```js
export function lifecycleMixin (Vue: Class<Component>) {
  Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
    const vm: Component = this
    if (vm._isMounted) {
      callHook(vm, 'beforeUpdate')
    }
    const prevEl = vm.$el
    const prevVnode = vm._vnode
    const prevActiveInstance = activeInstance
    activeInstance = vm
    vm._vnode = vnode
    // Vue.prototype.__patch__ is injected in entry points
    // based on the rendering backend used.
    if (!prevVnode) {
      // initial render
      vm.$el = vm.__patch__(
        vm.$el, vnode, hydrating, false /* removeOnly */,
        vm.$options._parentElm,
        vm.$options._refElm
      )
      // no need for the ref nodes after initial patch
      // this prevents keeping a detached DOM tree in memory (#5851)
      vm.$options._parentElm = vm.$options._refElm = null
    } else {
      // updates
      vm.$el = vm.__patch__(prevVnode, vnode)
    }
    activeInstance = prevActiveInstance
    // update __vue__ reference
    if (prevEl) {
      prevEl.__vue__ = null
    }
    if (vm.$el) {
      vm.$el.__vue__ = vm
    }
    // if parent is an HOC, update its $el as well
    if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
      vm.$parent.$el = vm.$el
    }
    // updated hook is called by the scheduler to ensure that children are
    // updated in a parent's updated hook.
  }

  Vue.prototype.$forceUpdate = function () {
    const vm: Component = this
    if (vm._watcher) {
      vm._watcher.update()
    }
  }

  Vue.prototype.$destroy = function () {
    const vm: Component = this
    if (vm._isBeingDestroyed) {
      return
    }
    callHook(vm, 'beforeDestroy')
    vm._isBeingDestroyed = true
    // remove self from parent
    const parent = vm.$parent
    if (parent && !parent._isBeingDestroyed && !vm.$options.abstract) {
      remove(parent.$children, vm)
    }
    // teardown watchers
    if (vm._watcher) {
      vm._watcher.teardown()
    }
    let i = vm._watchers.length
    while (i--) {
      vm._watchers[i].teardown()
    }
    // remove reference from data ob
    // frozen object may not have observer.
    if (vm._data.__ob__) {
      vm._data.__ob__.vmCount--
    }
    // call the last hook...
    vm._isDestroyed = true
    // invoke destroy hooks on current rendered tree
    vm.__patch__(vm._vnode, null)
    // fire destroyed hook
    callHook(vm, 'destroyed')
    // turn off all instance listeners.
    vm.$off()
    // remove __vue__ reference
    if (vm.$el) {
      vm.$el.__vue__ = null
    }
  }
}
```

### renderMixin
```js
export function renderMixin (Vue: Class<Component>) {
  Vue.prototype.$nextTick = function (fn: Function) {
    return nextTick(fn, this)
  }

  Vue.prototype._render = function (): VNode {
    const vm: Component = this
    const {
      render,
      staticRenderFns,
      _parentVnode
    } = vm.$options

    if (vm._isMounted) {
      // if the parent didn't update, the slot nodes will be the ones from
      // last render. They need to be cloned to ensure "freshness" for this render.
      for (const key in vm.$slots) {
        const slot = vm.$slots[key]
        if (slot._rendered) {
          vm.$slots[key] = cloneVNodes(slot, true /* deep */)
        }
      }
    }

    vm.$scopedSlots = (_parentVnode && _parentVnode.data.scopedSlots) || emptyObject

    if (staticRenderFns && !vm._staticTrees) {
      vm._staticTrees = []
    }
    // set parent vnode. this allows render functions to have access
    // to the data on the placeholder node.
    vm.$vnode = _parentVnode
    // render self
    let vnode
    try {
      vnode = render.call(vm._renderProxy, vm.$createElement)
    } catch (e) {
      handleError(e, vm, `render function`)
      // return error render result,
      // or previous vnode to prevent render error causing blank component
      /* istanbul ignore else */
      if (process.env.NODE_ENV !== 'production') {
        vnode = vm.$options.renderError
          ? vm.$options.renderError.call(vm._renderProxy, vm.$createElement, e)
          : vm._vnode
      } else {
        vnode = vm._vnode
      }
    }
    // return empty vnode in case the render function errored out
    if (!(vnode instanceof VNode)) {
      if (process.env.NODE_ENV !== 'production' && Array.isArray(vnode)) {
        warn(
          'Multiple root nodes returned from render function. Render function ' +
          'should return a single root node.',
          vm
        )
      }
      vnode = createEmptyVNode()
    }
    // set parent
    vnode.parent = _parentVnode
    return vnode
  }

  // internal render helpers.
  // these are exposed on the instance prototype to reduce generated render
  // code size.
  Vue.prototype._o = markOnce
  Vue.prototype._n = toNumber
  Vue.prototype._s = toString
  Vue.prototype._l = renderList
  Vue.prototype._t = renderSlot
  Vue.prototype._q = looseEqual
  Vue.prototype._i = looseIndexOf
  Vue.prototype._m = renderStatic
  Vue.prototype._f = resolveFilter
  Vue.prototype._k = checkKeyCodes
  Vue.prototype._b = bindObjectProps
  Vue.prototype._v = createTextVNode
  Vue.prototype._e = createEmptyVNode
  Vue.prototype._u = resolveScopedSlots
  Vue.prototype._g = bindObjectListeners
}
```