# vue响应式原理
## initState
new Vue() => _init() => initState:
```javascript
function initState (vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  if (opts.props) initProps(vm, opts.props)
  if (opts.methods) initMethods(vm, opts.methods)
  if (opts.data) {
    initData(vm)
  } else {
    observe(vm._data = {}, true /* asRootData */)
  }
  if (opts.computed) initComputed(vm, opts.computed)
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
}
```

判断该vue实例是否存在`props`、`methods`、`data`、`initComputed`、`initWatch`进行调用相应的初始化函数
`initProps`与`initData`主要工作是调用`defineProperty`给属性分别挂载get(触发该钩子时，会将当前属性的dep实例推入当前的Dep.target也就是当前watcher的deps中即它订阅的依赖，Dep.target下文会讲到。且该dep实例也会将当前watcher即观察者推入其subs数组中)、set方法（通知该依赖subs中所有的观察者watcher去调用他们的update方法）。

### **initComputed**
它的作用是将computed对象中所有的属性遍历，并给该属性new一个computed watcher（计算属性中定义了个dep依赖，给需要使用该计算属性的watcher订阅）。也会通过调用`defineProperty`给computed挂载get（get方法）、set方法（set方法会判断是否传入，如果没传入会设置成noop空函数）

`computed`属性的get方法是下面函数的返回值函数
```javascript
function createComputedGetter (key) {
  return function computedGetter () {
    const watcher = this._computedWatchers && this._computedWatchers[key]
    if (watcher) {
      watcher.depend()
      return watcher.evaluate()
    }
  }
}
```
注意其中的`watcher.depend()`,该方法让用到该属性的watcher观察者订阅该watcher中的依赖，且该计算属性watcher会将订阅它的watcher推入他的subs中(当计算属性值改变的时候，通知订阅他的watcher观察者)

`watcher.evaluate()`，该方法是通过调用watcher的get方法(其中需要注意的是watcher的get方法会调用pushTarget将之前的Dep.target实例入栈，并设置Dep.target为该computed watcher,被该计算属性依赖的响应式属性会将该computed watcher推入其subs中，所以当被依赖的响应式属性改变时，会通知订阅他的computed watcher,computed watcher 再通知订阅该计算属性的watcher调用update方法)，get方法中调用计算属性key绑定的handler函数计算出值。

### **initWatch**
该watcher 为user watcher（开发人员自己在组件中自定义的）。

initWatch的作用是遍历watch中的属性，并对每个watch监听的属性调用定义的$watch
```javascript
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
    options.user = true // 代表该watcher是用户自定义watcher
    const watcher = new Watcher(vm, expOrFn, cb, options)
    if (options.immediate) {
      cb.call(vm, watcher.value)
    }
    return function unwatchFn () {
      watcher.teardown()
    }
  }
```
代码中调用new Watcher的时候，也会同render watcher一样，执行下watcher的get方法，调用`pushTarget`将当前user watcher赋值给Dep.target,get()中`value = this.getter.call(vm, vm)`这个语句会触发该自定义watcher监听的响应式属性的get方法，并将当前的user watcher推入该属性依赖的subs中，所以当user watcher监听的属性set触发后，通知订阅该依赖的watcher去触发update，也就是触发该watch绑定的key对应的handler。然后就是调用popTarget出栈并赋值给Dep.target。

initState初始化工作大致到这里过，接下去会执行$mount开始渲染工作
$mount主要工作：new了一个渲染Watcher，并将updateCompent作为callback传递进去并执行
```javascript
updateComponent = () => {
      vm._update(vm._render(), hydrating)
    }

new Watcher(vm, updateComponent, noop, {
    before () {
      if (vm._isMounted) {
        callHook(vm, 'beforeUpdate')
      }
    }
  }, true /* isRenderWatcher */)
```
三种watcher中new Watcher的时候，只有computed watcher不会一开始就执行它的get()方法。$mount里面new的这个render watcher会调用`get()`方法，调用`pushTarget`将当前render watcher赋值给Dep.target。接下去重头戏来了，调用`updateComponent`,该方法会执行`vm._update(vm._render(), hydrating)`，其中render函数会触发html中使用到的响应式属性的get钩子。get钩子会让该响应式属性的依赖实例dep将当前的render watcher推入其subs数组中，所以当依赖的响应式属性改变之后，会遍历subs通知订阅它的watcher去调用update()。

## 例子
可能大家对watcher和dep调来调去一头雾水，我讲个实例
```javascript
<div id="app">
      <div>{{a}}</div>
      <div>{{b}}</div>
    </div>

new Vue({
  el: "#app",
  data() {
    return {
      a:1,
    }
  },
  computed:{
    b() {
      return a+1
    }
  },
})
```
我直接从渲染开始讲，只讲跟dep跟watcher有关的

**$mount**：new一个渲染watcher（watcher的get方法中会将渲染watcher赋值给Dep.target）的时候会触发 `vm._update(vm._render(), hydrating)`，render的时候会获取html中用到的响应式属性，上面例子中先用到了a,这时会触发a的get钩子,其中`dep.depend()`会将当前的渲染watcher推入到a属性的dep的subs数组中。
接下去继续执行，访问到b（b是计算属性的值），会触发计算属性的get方法。计算属性的get方法是调用`createComputedGetter`函数后的返回函数`computedGetter`，`computedGetter`函数中会执行`watcher.depend()`。Watcher的depend方法是专门留给computed watcher使用的。刚才上面说过了除了computed watcher，其他两种watcher在new 完之后都会执行他们的get方法，那么computed watcher在new完之后干嘛呢，它会new一个dep。回到刚才说的专门为computed watcher开设的方法`watcher.depend()`，他的作用是执行`this.dep.depend()`（computed watcher定义的dep就是在这里使用到的）。`this.dep.depend()`会让当前的渲染watcher订阅该计算属性依赖，该计算属性也会将渲染watcher推入到它自己的subs（[render watcher]）中，当计算属性的值修改之后会通知subs中的watcher调用`update()`,所以计算属性值变了页面能刷新。回到前面说的触发b计算属性的get钩子那里，get钩子最后会执行`watcher.evaluate()`,`watcher.evaluate()`会执行computed watcher的`get()`方法。这时候重点来了，会将Dep.target（render watcher）推入targetStack栈中（存入之后以便待会儿取出继续用），然后将这个计算属性的computed watcher赋值给Dep.target。get方法中`value = this.getter.call(vm, vm)`,会执行computed属性绑定的handler。如上面例子中return a + 1。使用了a那么就一定会触发a的get钩子，get钩子又会调用`dep.depend()`，dep.depend()会让computed watcher将dep存入它的deps数组中，a的dep会将**当前的Dep.target(computed watcher)存入其subs数组中，当前例子中a的subs中就会是[render watcher,computed watcher]**,所以a值变化会遍历a的subs中的watcher调用`update()`方法，html中用到的a会刷新，计算属性watcher调用`update()`方法会通知他自己的subs（[render watcher]）中render watcher去调用update方法，html中用到的计算属性b才会刷新dom（这里提个醒，我只是粗略的讲，计算属性依赖的属性变化后他不一定会触发更新，他会比较计算完之后的值是否变化）。computed watcher的get()方法最后会调用`popTarget()`,将之前存入render watcher出栈并赋值给Dep.target，这时候我例子中targetStack就变成空数组了。render watcher的get方法执行到最后也会出栈，这时候会将Dep.target赋值会空。