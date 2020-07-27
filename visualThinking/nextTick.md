
# nextTick - 看的见的思考
## 问题
1. 为什么需要先执行父组件

## 看的见的思考

vue3 的 nextTick 的实现是在 scheduler.ts 实现的

scheduler 中文翻译过来是调度器，但是这个调度器这个词还是有点抽象啊。[[调度器]]

先找到核心执行逻辑

```js
function flushJobs(seen?: CountMap) { isFlushPending = false
  isFlushing = true
  let job
  if (__DEV__) {
    seen = seen || new Map()
  }

  // Sort queue before flush.
  // This ensures that:
  // 1. Components are updated from parent to child. (because parent is always
  //    created before the child so its render effect will have smaller
  //    priority number)
  // 2. If a component is unmounted during a parent component's update,
  //    its update can be skipped.
  // Jobs can never be null before flush starts, since they are only invalidated
  // during execution of another flushed job.
  queue.sort((a, b) => getId(a!) - getId(b!))

  while ((job = queue.shift()) !== undefined) {
    if (job === null) {
      continue
    }
    if (__DEV__) {
      checkRecursiveUpdates(seen!, job)
    }
    callWithErrorHandling(job, null, ErrorCodes.SCHEDULER)
  }
  flushPostFlushCbs(seen)
  isFlushing = false
  // some postFlushCb queued jobs!
  // keep flushing until it drains.
  if (queue.length || postFlushCbs.length) {
    flushJobs(seen)
  }
}
```

这里有几个关键的点，
1. queue
2. flushPostFlushCbs

先看看 queue 这个队列是干什么的？ 队列里面存的是什么？在什么时候存的

```js

export function queueJob(job: Job) {
  if (!queue.includes(job)) {
    queue.push(job)
    queueFlush()
  }
}
```

是通过 queueJob 来收集 job 的，那接着看看 job 都是什么东西

```js
const prodEffectOptions = {
  scheduler: queueJob
}

function createDevEffectOptions(
  instance: ComponentInternalInstance
): ReactiveEffectOptions {
  return {
    scheduler: queueJob,
    onTrack: instance.rtc ? e => invokeArrayFns(instance.rtc!, e) : void 0,
    onTrigger: instance.rtg ? e => invokeArrayFns(instance.rtg!, e) : void 0
  }
}
```

```js
    instance.update = effect(function componentEffect() {
```

啊哦，原来是传给了做 update 时的 effect 的配置里面

注意这个 scheduler:queueJob

在回顾一下 如果给了 effect options 里面有 scheduler 的话，effect 会有什么行为

```js

  const run = (effect: ReactiveEffect) => {
    if (__DEV__ && effect.options.onTrigger) {
      effect.options.onTrigger({
        effect,
        target,
        key,
        type,
        newValue,
        oldValue,
        oldTarget
      })
    }
    if (effect.options.scheduler) {
      effect.options.scheduler(effect)
    } else {
      effect()
    }
  }

```

啊哦，当响应式对象发生改变后，执行 effect 的时候，如果有 scheduler 这个 option 的话，会执行这个 scheduler 函数，并且把 effect 传入

那其实这里的 scheduler 函数就是 上面的 queueJob 函数，并且 effect 就是 update 函数

那也就是说，每次更新的时候都会把 update 这个函数推入到 queueJob 内

那推入后呢？ 

在回顾一下 queueJob 函数

```js
export function queueJob(job: Job) {
  if (!queue.includes(job)) {
    queue.push(job)
    queueFlush()
  }
}
```

这里的 job 就是 update 函数，那么这里只会推入一次，也就是说，当后续执行 queueFlush 之后，才会执行 update 函数。

嗯，这样的话就可以避免修改了数据之后就会立马渲染页面了！ 棒！

其实 nextTick 的目的也是如此，而且这个做法在游戏里面也有，我自己在写物理引擎的时候也用过，但是没有现在理解的更清晰

继续继续，看看接下来做什么了

```js
function queueFlush() {
  if (!isFlushing && !isFlushPending) {
    isFlushPending = true
    nextTick(flushJobs)
  }
}
```

那其实接着就是执行 queueFlush 了，这里的关键是调用了 nextTick 给了 flushJobs 函数

而 flushJobs 函数就是我们最开始看到的入口函数，在那里会最终执行我们传入的 job 函数

在看 flushJobs 之前 ，先看看 nextTick 是怎么实现的

```js
const p = Promise.resolve()

export function nextTick(fn?: () => void): Promise<void> {
  return fn ? p.then(fn) : p
}
```

这里还挺简单的,就是用的 Promise.resolve() ，把要执行的函数延迟到 微任务队列里面执行。

接着我们看看在最终执行到微任务队列的时候是怎么执行 flushJobs 函数的把
这里暂时只关注和 queue 队列有关的逻辑点

```js
  queue.sort((a, b) => getId(a!) - getId(b!))

  while ((job = queue.shift()) !== undefined) {
    if (job === null) {
      continue
    }
    if (__DEV__) {
      checkRecursiveUpdates(seen!, job)
    }
    callWithErrorHandling(job, null, ErrorCodes.SCHEDULER)
  }
    isFlushing = false
  // some postFlushCb queued jobs!
  // keep flushing until it drains.
  if (queue.length || postFlushCbs.length) {
    flushJobs(seen)
  }
```

看起来很简单，先排序（为什么需要排序呢？ 这里的 id 是什么时候给的，）

然后取队列的头部 job 执行就完事了

这里是个递归的操作

好了，那接着要搞懂的问题就是这个 id 的问题

在 sort 的逻辑上有详细的注释

	  // Sort queue before flush.
	  // This ensures that:
	  // 1. Components are updated from parent to child. (because parent is always
	  //    created before the child so its render effect will have smaller
	  //    priority number)
	  // 2. If a component is unmounted during a parent component's update,
	  //    its update can be skipped.
	  // Jobs can never be null before flush starts, since they are only invalidated
	  // during execution of another flushed job.

尝试着翻译翻译，然后理解一下是啥意思

有可能会涉及到设置 id 的信息

1. 组件更新从父级到孩子，因为父级总是在子组件之前创建好的，所以它渲染 effect 有较小的优先级（因为初始化的时候先创建的父组件，所以父组件的 id 是小的，换句话说也就是会先执行父组件）
2. 如果一个组件在父级组件更新时是 unmounted 的，那么它的更新会被跳过（这个没太理解，找一找对应的 demo 来验证一下）

好，到这里的时候其实我们能理解调用的优先级是先调用父组件

因为创建 update = effect(fn) 的时候，这里的 effect 的 id 是从零开始计算的，又因为先初始化父级组件，所以后面更新的时候基于 id 排序，会先执行父级组件的 update

那其实我想知道的是，为什么需要先执行父组件？

暂时想不出来 先记录一下


---
还有一个队列就是 queuePostRenderEffect 的应用了

```js
export function queuePostFlushCb(cb: Function | Function[]) {
  if (!isArray(cb)) {
    postFlushCbs.push(cb)
  } else {
    postFlushCbs.push(...cb)
  }
  queueFlush()
}
```

和 queue 队列不同，它把收集的 job 都添加到 postFlushCbs 数组里面去

```js
  queue.sort((a, b) => getId(a!) - getId(b!))

  // 处理 queue 队列
  while ((job = queue.shift()) !== undefined) {
    if (job === null) {
      continue
    }
    if (__DEV__) {
      checkRecursiveUpdates(seen!, job)
    }
    callWithErrorHandling(job, null, ErrorCodes.SCHEDULER)
  }
  // 处理完 queue 队列后处理 postFlushCbs 队列
  flushPostFlushCbs(seen)
  isFlushing = false
  // some postFlushCb queued jobs!
  // keep flushing until it drains.
  if (queue.length || postFlushCbs.length) {
    flushJobs(seen)
  }
```

然后是在处理完 queue 之后再处理 postFlushCbs  ，从命名上也能体现出来 post （后刷新）

接着的重点是看看都是把什么任务添加到这个 postFlushCbs 队列来呢？

```js
export const queuePostRenderEffect = __FEATURE_SUSPENSE__
  ? queueEffectWithSuspense
  : queuePostFlushCb
```

在 renderer.js 使用的时候给它改了个名称叫做 queuePostRenderEffect ，从命名上我们能猜到是在 渲染 effect 之后调用的队列

> 对 Suspense 组件的处理我们以后单独搞一个章节来分析

```js
 queuePostRenderEffect(() => {
        vnodeHook && invokeVNodeHook(vnodeHook, parentComponent, vnode)
        transition && !transition.persisted && transition.enter(el)
        dirs && invokeDirectiveHook(vnode, null, parentComponent, 'mounted')
      }, parentSuspense)
```

```js
 if ((vnodeHook = newProps.onVnodeUpdated) || dirs) {
      queuePostRenderEffect(() => {
        vnodeHook && invokeVNodeHook(vnodeHook, parentComponent, n2, n1)
        dirs && invokeDirectiveHook(n2, n1, parentComponent, 'updated')
      }, parentSuspense)
    }
```
```js
  // mounted hook
        if (m) {
          queuePostRenderEffect(m, parentSuspense)
        }
        // onVnodeMounted
        if ((vnodeHook = props && props.onVnodeMounted)) {
          queuePostRenderEffect(() => {
            invokeVNodeHook(vnodeHook!, parent, initialVNode)
          }, parentSuspense)
        }
        // activated hook for keep-alive roots.
        if (
          a &&
          initialVNode.shapeFlag & ShapeFlags.COMPONENT_SHOULD_KEEP_ALIVE
        ) {
          queuePostRenderEffect(a, parentSuspense)
        }
        instance.isMounted = true
```

会发现在处理一些 hook 的时候都会用这个方法，也就是说 hook 的处理都要等到处理完 update 逻辑。

至于为什么，我还没有猜到

补充：

我发现如果是触发 beforeXxx 的逻辑的时候直接调用 hook 即可

而如果是触发 xxxed 的逻辑的时候需要等到渲染之后

那么这样就能理解了为什么要等到渲染之后

不渲染完的话，怎么能叫 xxxed 呢？ 哈哈 

> 这里的 xxxed 指的是 mounted updated 等等

---

到此为止，其实 nextTick 已经被分析的差不多了，还剩下几个为什么这么做的问题，当然了着就是思考的价值所在，也是看源码的意义，就是它是解决什么问题的，也就是 why 层面的东西。

---

## 总结

### 问题

我们先聊一个场景，当响应式对象发生改变会，会触发 update 逻辑，当触发 update 逻辑后立马重新渲染视图的话 ok 没有问题

但是我们需要考虑这么一个场景

```js
var count = ref(10)

for(let i=0; i<100; i++){
	count = i
}
```

我把响应式对象在 for 循环中（同一帧）更新了 100 次，如果按照我们上面的更新策略，那么就需要更新 100 次视图（响应式数据变更就会触发重新渲染视图）

那我们怎么去优化这个问题呢？

其实通过观察我们会发现，最后的结果就是渲染 count 为 100 的情况。

那么我们就要想办法做到响应式完全变更完之后再渲染视图了

那怎么做呢？

### 怎么做

我们可以利用 js 的事件循环机制，上面的这个 for 循环的操作是在当前执行栈内执行（同步的），当前执行栈执行完成后，js 会检查事件队列里面的异步任务 （会先执行 微任务，然后在执行宏任务），所以我们完全可以把渲染的逻辑延迟到微任务里执行

这样就可以解决上面说的问题了，而着其实就是 vue.nextTick 要解决的问题，以及它的解决方案

vue 中维护了一个队列，当响应式对象发生变更后，会把 update 函数 push 到队列内，在入队列的时候还做了个检查，如果添加过就不会在添加了。确保同一个 update 函数只执行一次。然后利用 Promise.resolve()， 在微任务执行的时候在去执行这个队列里面所有的函数。

