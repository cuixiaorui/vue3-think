
# vue3-emit 源码分析
这次来搞 emit 

看看 vue3 的 emit 和之前有什么变动

## 看的见的思考

先打开 componentEmits.ts 文件

先看 emit 函数


```js
export function emit(
  instance: ComponentInternalInstance,
  event: string,
  ...args: any[]
) {
  const props = instance.vnode.props || EMPTY_OBJ

  let handlerName = `on${capitalize(event)}`
  let handler = props[handlerName]
  // for v-model update:xxx events, also trigger kebab-case equivalent
  // for props passed via kebab-case
  if (!handler && event.startsWith('update:')) {
    handlerName = `on${capitalize(hyphenate(event))}`
    handler = props[handlerName]
  }
  if (!handler) {
    handler = props[handlerName + `Once`]
    if (!instance.emitted) {
      ;(instance.emitted = {} as Record<string, boolean>)[handlerName] = true
    } else if (instance.emitted[handlerName]) {
      return
    }
  }
  if (handler) {
    callWithAsyncErrorHandling(
      handler,
      instance,
      ErrorCodes.COMPONENT_EVENT_HANDLER,
      args
    )
  }
}

```

这里注意我们分析的时候会去掉 dev 坏境的代码处理，先只关注核心逻辑

这里的第一步是先取 vnode.props 里面的值

看看用来做什么

哦，这里是把传入的 evnet 参数加了一个  on 前缀来看看 props 内有没有

```js
  let handlerName = `on${capitalize(event)}`
  let handler = props[handlerName

```

这样的话，其实就直接从 props 里面取定义的事件函数了

这里我就知道了，只要加个 on前缀即可，后面在发起 emit 的时候 vue 会自己去找对应的函数

```js
  if (!handler && event.startsWith('update:')) {
    handlerName = `on${capitalize(hyphenate(event))}`
    handler = props[handlerName]
  }
```

这里不光是处理事件 ，还找了一个基于 v-model 的逻辑，现在的 v-model 都是用 update: 来做前缀的了

```js
  if (!handler) {
    handler = props[handlerName + `Once`]
    if (!instance.emitted) {
      ;(instance.emitted = {} as Record<string, boolean>)[handlerName] = true
    } else if (instance.emitted[handlerName]) {
      return
    }
  }

```

除了上面的情况,还可以自动加后缀 Once ，只执行一次的函数回调

然后存了一个状态，看看之前调用过没，调用过的话，就设置为 true 了

```js
  if (handler) {
    callWithAsyncErrorHandling(
      handler,
      instance,
      ErrorCodes.COMPONENT_EVENT_HANDLER,
      args
    )
  }

```

最后的话，看看有没有这个  handler  有的话调用

---

总结

要做的事就是从 props 内取出对应的 handle 回调函数，现在有下面三种情况：
- 事件监听，有的话就一直触发，有对应的前缀 on
- v-model 的处理，必须有 update: 前缀
- 临时的事件监听，后缀有 Once 的

找到对应的 handle 回调后，调用即可

---

别的逻辑看起来是属于工具函数，暂时先不关注
