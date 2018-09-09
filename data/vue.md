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

**initComputed**
它的作用是将computed对象中所有的属性遍历，并给该属性new一个computed watcher（计算属性中定义了个dep依赖，给需要使用该计算属性的watcher订阅）。也会通过调用`defineProperty`给computed挂在get（get方法）、set方法（set方法会判断是否传入，如果没传入会设置成noop空函数）

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

`watcher.evaluate()`，该方法是通过调用watcher的get方法(其中需要注意的是watcher的get方法会调用pushTarget将该watcher实例入栈，并设置Dep.target为该computed watcher,被依赖的响应式属性会将该computed watcher推入其)后调用计算属性绑定的函数，当调用该方法时候，如果其中依赖了其他响应式属性，那么就会触发他们的get方法，




