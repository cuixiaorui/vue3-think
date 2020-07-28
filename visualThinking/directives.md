# directives
今天来看看 vue3 指令是如何实现的

## 看的见的思考

首先第一步还是找到对应的测试文件看起

open directives.spec.ts 文件

```js
  it('should work', async () => {})
```

这个测试吓到我了，怎么那么多逻辑

大概有 130 行代码

那就一点点的来拆解吧

```js
    const beforeMount = jest.fn(((el, binding, vnode, prevVNode) => {
    }) as DirectiveHook)

    const mounted = jest.fn(((el, binding, vnode, prevVNode) => {
    }) as DirectiveHook)

    const beforeUpdate = jest.fn(((el, binding, vnode, prevVNode) => {
    }) as DirectiveHook)

    const updated = jest.fn(((el, binding, vnode, prevVNode) => {
    }) as DirectiveHook)

    const beforeUnmount = jest.fn(((el, binding, vnode, prevVNode) => {
    }) as DirectiveHook)

    const unmounted = jest.fn(((el, binding, vnode, prevVNode) => {
    }) as DirectiveHook)

```

着部分就是定义一些指令的生性周期函数

```js
	    const Comp = {
      setup() {
        _instance = currentInstance
      },
      render() {
        _prevVnode = _vnode
        _vnode = withDirectives(h('div', count.value), [
          [
            dir,
            // value
            count.value,
            // argument
            'foo',
            // modifiers
            { ok: true }
          ]
        ])
        return _vnode
      }
    }
```
然后定义个组件，等会方便测试

这里有个和普通组件不同的点，是在 render 函数内，有一个 withDirectives 函数，它接受两个参数，第一个参数看起来就是 h() 的返回值 vnode ，然后还接受一个数组，这个数组有意思，数组里面可以包含 n 个数组，看来这个就是对应的指令的数据结构了

```js

    const root = nodeOps.createElement('div')
    render(h(Comp), root)

    expect(beforeMount).toHaveBeenCalledTimes(1)
    expect(mounted).toHaveBeenCalledTimes(1)

    count.value++
    await nextTick()
    expect(beforeUpdate).toHaveBeenCalledTimes(1)
    expect(updated).toHaveBeenCalledTimes(1)

    render(null, root)
    expect(beforeUnmount).toHaveBeenCalledTimes(1)
    expect(unmounted).toHaveBeenCalledTimes(1)

```
这里倒是很简单

就是组件在 mount 的时候，会调用指令的 beforeMount 和 mounted 两个生命周期

在组件更新的时候回调用 beforeUpdate 和 updated 两个生命周期

以及在组件销毁的时候会调用 beforUnmount 和 unmounted 

这个测试的大框架基本就是这些

接着看看每个具体的生命周期函数都做了什么吧

beforeMount 

```js
    const beforeMount = jest.fn(((el, binding, vnode, prevVNode) => {
      expect(el.tag).toBe('div')
      // should not be inserted yet
      expect(el.parentNode).toBe(null)
      expect(root.children.length).toBe(0)

      assertBindings(binding)

      expect(vnode).toBe(_vnode)
      expect(prevVNode).toBe(null)
    }) as DirectiveHook)
```

这哥们接受四个参数

- el 
	- 就是和当前指令绑定的 element dom 元素
- bingding
	- 这个对象挺有意思，看起来它和下面传给 withDirectives 第二个数组内的第一个数组的数据结构很类似、
	- dir 是之前声明好的几个生命周期对象
	- value 是 count.value 的值 ，这里的值肯定是绑定给指令的值咯
	- argument:  是 foo 
	- modifiers 是个对象 {ok:true} 原来是个修饰符对象

- vnode 
	- 新的虚拟节点
- prevVNode
	- 之前的虚拟节点

这个 bingding 可以看看之前 vue2 的时候是个什么东西 [指令-钩子函数参数](https://cn.vuejs.org/v2/guide/custom-directive.html#%E9%92%A9%E5%AD%90%E5%87%BD%E6%95%B0%E5%8F%82%E6%95%B0)


然后对于 bingdings 的测试看这个

```js
    function assertBindings(binding: DirectiveBinding) {
      expect(binding.value).toBe(count.value)
      expect(binding.arg).toBe('foo')
      expect(binding.instance).toBe(_instance && _instance.proxy)
      expect(binding.modifiers && binding.modifiers.ok).toBe(true)
    }
```

这里就能发现 bingding 着哥们都有啥了

可以取到指令的 value、arg、instance、以及 modifiers

那好了，那我现在知道了，着几个值是在哪里设置的

当解析 template 的时候，会生成对应的数据，给到 withDirectives 的第二个数组内

来，在回顾一下

```js
    const Comp = {
      setup() {
        _instance = currentInstance
      },
      render() {
        _prevVnode = _vnode
        _vnode = withDirectives(h('div', count.value), [
          [
            dir,
            // value
            count.value,
            // argument
            'foo',
            // modifiers
            { ok: true }
          ]
        ])
        return _vnode
      }
    }

```
那这里在大胆的猜一猜，为什么最外层是个数组，那肯定是因为一个 dom 可以指定好几个指令呗。

来，让我去验证验证 [demo](https://vue-next-template-explorer.netlify.app/#%7B%22src%22%3A%22%3Cdiv%20id%3D%5C%22app%5C%22%3E%5Cn%20%20%3Cdiv%20v-test%3D%5C%22a%5C%22%20v-hahah%3D%5C%22b%5C%22%3E111%3C%2Fdiv%3E%5Cn%3C%2Fdiv%3E%5Cn%22%2C%22ssr%22%3Afalse%2C%22options%22%3A%7B%22mode%22%3A%22module%22%2C%22prefixIdentifiers%22%3Afalse%2C%22optimizeImports%22%3Afalse%2C%22hoistStatic%22%3Atrue%2C%22cacheHandlers%22%3Atrue%2C%22scopeId%22%3Anull%2C%22ssrCssVars%22%3A%22%7B%20color%20%7D%22%2C%22optimizeBindings%22%3Afalse%7D%7D)

果然是这个样子的 

```js
//template
<div id="app">
  <div v-test="a" v-hahah="b">111</div>
</div>

// render 
import { resolveDirective as _resolveDirective, createVNode as _createVNode, withDirectives as _withDirectives, openBlock as _openBlock, createBlock as _createBlock } from "vue"

const _hoisted_1 = { id: "app" }

export function render(_ctx, _cache) {
  const _directive_test = _resolveDirective("test")
  const _directive_hahah = _resolveDirective("hahah")

  return (_openBlock(), _createBlock("div", _hoisted_1, [
    _withDirectives(_createVNode("div", null, "111", 512 /* NEED_PATCH */), [
      [_directive_test, _ctx.a],
      [_directive_hahah, _ctx.b]
    ])
  ]))
}

```

那其实到着我已经大概知道指令的工作了

那其实接下来的核心点就是看看 withDirectives 做了啥

还有这个 resolveDirective 

啊哦，突然又想起来个点

在测试文档里面第二个参数是手写的
```js
	      render() {
        _prevVnode = _vnode
        _vnode = withDirectives(h('div', count.value), [
          [
            dir,
            // value
            count.value,
            // argument
            'foo',
            // modifiers
            { ok: true }
          ]
        ])
        return _vnode
      }
    }
```

而通过 template-explorer 生成的 render 函数里面是用的 _resolveDirective

```js
export function render(_ctx, _cache) {
  const _directive_test = _resolveDirective("test")
  const _directive_hahah = _resolveDirective("hahah")

  return (_openBlock(), _createBlock("div", _hoisted_1, [
    _withDirectives(_createVNode("div", null, "111", 512 /* NEED_PATCH */), [
      [_directive_test, _ctx.a],
      [_directive_hahah, _ctx.b]
    ])
  ]))
}
```

那是不是也就能说明 resolveDirective 着哥们就是返回一个和测试里面相似的数组呗

嗯，好点子，稍等去验证验证

先继续往下看测试

看看还有啥可以特别关注的点嘛

发现没啥可关注的点了

去研究下 resolveDirective 

```js
export function resolveDirective(name: string): Directive | undefined {
  return resolveAsset(DIRECTIVES, name)
}
// 
function resolveAsset(
  type: typeof COMPONENTS | typeof DIRECTIVES,
  name: string,
  warnMissing = true
) {
  const instance = currentRenderingInstance || currentInstance
  if (instance) {
    const Component = instance.type

    // self name has highest priority
    if (type === COMPONENTS) {
      const selfName =
        (Component as FunctionalComponent).displayName || Component.name
      if (
        selfName &&
        (selfName === name ||
          selfName === camelize(name) ||
          selfName === capitalize(camelize(name)))
      ) {
        return Component
      }
    }

    const res =
      // local registration
      resolve((Component as ComponentOptions)[type], name) ||
      // global registration
      resolve(instance.appContext[type], name)
    if (__DEV__ && warnMissing && !res) {
      warn(`Failed to resolve ${type.slice(0, -1)}: ${name}`)
    }
    return res
  } else if (__DEV__) {
    warn(
      `resolve${capitalize(type.slice(0, -1))} ` +
        `can only be used in render() or setup().`
    )
  }
}
```

我发现这哥们就是把组件名字给处理一下，然后查到相对应的组件或者 Directive，然后返回啊

所以它的作用就是处理一下名字，然后返回对应的组件或者 Directive

这个 directive 是存在哪里呢？

```js
 const res =
      // local registration
      resolve((Component as ComponentOptions)[type], name) ||
      // global registration
      resolve(instance.appContext[type], name)
```

这里如果 type 是 directives 的话，那么就是取的 component instance 的 directives 字段

看看这个字段是个什么类型的数据

```js

// 在 componentOptions.js 内
  directives?: Record<string, Directive
```

那看看 Directive 都是个啥东西

```js
export type DirectiveHook<T = any, Prev = VNode<any, T> | null, V = any> = (
  el: T,
  binding: DirectiveBinding<V>,
  vnode: VNode<any, T>,
  prevVNode: Prev
) => void

export type SSRDirectiveHook = (
  binding: DirectiveBinding,
  vnode: VNode
) => Data | undefined

export interface ObjectDirective<T = any, V = any> {
  beforeMount?: DirectiveHook<T, null, V>
  mounted?: DirectiveHook<T, null, V>
  beforeUpdate?: DirectiveHook<T, VNode<any, T>, V>
  updated?: DirectiveHook<T, VNode<any, T>, V>
  beforeUnmount?: DirectiveHook<T, null, V>
  unmounted?: DirectiveHook<T, null, V>
  getSSRProps?: SSRDirectiveHook
}

export type FunctionDirective<T = any, V = any> = DirectiveHook<T, any, V>
```

要不就是一个DirectiveHook函数，要不就是个都是 DirectiveHook 的数组

那它我明白了 ，继续看看 withDirectives 都干了啥

看之前猜想一下，在 render 内是用 withDirectives 包裹了 vnode ，那是不是往 vnode 里面添加 directive 呢？

插一嘴，其实这个 resolveDirective 就是为了收集用户在当前组件内，或者在全局定义好的指令对象/函数


---
withDirectives

```js
/**
 * Adds directives to a VNode.
 */
export function withDirectives<T extends VNode>(
  vnode: T,
  directives: DirectiveArguments
): T {
  const internalInstance = currentRenderingInstance
  const instance = internalInstance.proxy
  const bindings: DirectiveBinding[] = vnode.dirs || (vnode.dirs = [])
  for (let i = 0; i < directives.length; i++) {
    let [dir, value, arg, modifiers = EMPTY_OBJ] = directives[i]
    if (isFunction(dir)) {
      dir = {
        mounted: dir,
        updated: dir
      } as ObjectDirective
    }
    bindings.push({
      dir,
      instance,
      value,
      oldValue: void 0,
      arg,
      modifiers
    })
  }
  return vnode
}

```

这里注意 directives 还有 vnode.dirs 

其实就是遍历 directives 里面所有的数据，然后包装一下添加到 vnode.dirs 内

小细节就是，如果指令是个函数的话，那么给它转成对象，函数默认是 mounted 和 updated 声明周期函数

这里还需要看看是怎么用 withDirectives 函数的 (上面有)

```js
export function render(_ctx, _cache) {
  const _directive_test = _resolveDirective("test")
  const _directive_hahah = _resolveDirective("hahah")

  return (_openBlock(), _createBlock("div", _hoisted_1, [
    _withDirectives(_createVNode("div", null, "111", 512 /* NEED_PATCH */), [
      [_directive_test, _ctx.a],
      [_directive_hahah, _ctx.b]
    ])
  ]))
}
```

这里其中的第一个值就是指令的实现，后面是一些附加的值，比如说：
- dir(第一个值，指令的实现)
- instance
- value
- oldValue
- arg
- modifiers



这里的逻辑就是收集定义好的指令

所有的指令都已经收集到了 vnode.dirs 内了

然后我们看看都在什么时候调用的吧


---
invokeDirectiveHook

```js
export function invokeDirectiveHook(
  vnode: VNode,
  prevVNode: VNode | null,
  instance: ComponentInternalInstance | null,
  name: keyof ObjectDirective
) {
  const bindings = vnode.dirs!
  const oldBindings = prevVNode && prevVNode.dirs!
  for (let i = 0; i < bindings.length; i++) {
    const binding = bindings[i]
    if (oldBindings) {
      binding.oldValue = oldBindings[i].value
    }
    const hook = binding.dir[name] as DirectiveHook | undefined
    if (hook) {
      callWithAsyncErrorHandling(hook, instance, ErrorCodes.DIRECTIVE_HOOK, [
        vnode.el,
        binding,
        vnode,
        prevVNode
      ])
    }
  }
}
```

从 vnode.dirs 内取出定义好的指令对象/函数，然后基于类型来调用

比如我要调用 mounted 的钩子函数，那么 name 就等于 mounted

来看看是如何调用的

```js
      invokeDirectiveHook(vnode, null, parentComponent, 'beforeMount')
```

这里最后一个参数就是告诉它，需要调用 beforeMount 的 hook

这里的调用时期其实和初始化还有更新逻辑是一样的

至此， 所有的逻辑就都已经修改完了

---

稍等一下，写到这里的时候，去洗了个澡，然后顿悟了

突然就想明白了所有的流程

我先去写一个总结 ，然后在回过头来分析细节

---
## 总结
指令的实现分为三个阶段
- 定义指令
- 收集指令
- 执行指令
### 定义指令
这里分为两种，一种是组件内通过 directives 选项来定义好指令，还有一种是全局指令，可以把指令函数添加到 appContext.directives 上。
> 指令可以是个函数，也可以是个对象，对象上可以添加各种生命周期函数


### 收集指令
在 runtime 的时候就要收集指令了，那怎么收集指令呢？ 其实就是通过 withDirectives 这个函数来收集，把收集好的指令添加到 vnode.dirs 数组内，然后等待执行
> 具体细节还没有看

### 执行指令
最后在 runtime 的时候就需要执行指令了，那在什么时候执行呢？ 暂时还没看哈哈哈，不过应该是调用 invokeDirectiveHook 函数。

稍微看看在哪里调用的

啊哦，明白了。调用指令是基于  invokeDirectiveHook 函数来调用没错，但是这里调用的时机是不同的时候，比如 mounted 的时候，比如 update 的时候，所以在执行到相关逻辑的时候会调用 invokeDirectiveHook 函数，然后触发相对应的那个生命周期函数

---

哇哦，顿悟好神奇，整个思路简直太清晰了


## 参考资料
- [自定义指令](https://cn.vuejs.org/v2/guide/custom-directive.html#%E9%92%A9%E5%AD%90%E5%87%BD%E6%95%B0%E5%8F%82%E6%95%B0)
- [dynamic-directive-arguments](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0003-dynamic-directive-arguments.md)





