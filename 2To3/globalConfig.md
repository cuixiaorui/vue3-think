
之前在 app.config.globalProperties 上配置完的属性

在组件内通过 this.xxx 即可访问到

举个例子：
```js
// 全局配置
  app.config.globalProperties.$ELEMENT = {
    size: opts.size || "",
    zIndex: opts.zIndex || 2000,
  };
  
 // 组件内获取
 (this.$ELEMENT || {}).size;
```


但是到了 vue3 ，已经没有 this 了，所以你想要获取到全局配置的话需要通过

```js
 (getCurrentInstance().proxy.$ELEMENT || {}).size
```

吐槽一下，有点麻烦有点丑

原因就在于 vue3 没有直接暴露出 ComponentPublicInstance 类型

这里 getCurrentInstance().proxy 就是 ComponentPublicInstance 类型

而 getCurrentInstance() 获取到的是 ComponentInternalInstance 类型

所以需要额外的访问 proxy 来访问到 ComponentPublicInstance 

