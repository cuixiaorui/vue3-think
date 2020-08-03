
# 集成VTU

- 先安装 jest 和 @tests/jest
	-   "@types/jest": "^26.0.7",
	-    "jest": "^26.1.0",
	这是结合 jest 来测试组件
	
- 在安装 @vue/test-utils@next
	-  "@vue/test-utils": "^2.0.0-beta.0",
	这是最新的 vtu ，支持 vue3
	
- 安装 vue-jest
	- "vue-jest": "vuejs/vue-jest#next",

	一定要安装 #next 版本，不然有坑
	为了告诉 jest 如何处理 *.vue 的文件
	
- 创建 jest.config.js
	
	```js
	module.exports = {
    testEnvironment: 'jsdom',
    transform: {
	//  用 `vue-jest` 处理 `*.vue` 文件
      "^.+\\.vue$": "vue-jest",
      "^.+\\js$": "babel-jest"
    },
	// 告诉 Jest 处理 `*.vue` 文件
    moduleFileExtensions: ['vue', 'js', 'json', 'jsx', 'ts', 'tsx', 'node'],
  }
	```

- 安装 typescript
	- "typescript": "^3.7.5"

- 安装 babel-jest
	-  "babel-jest": "^26.1.0",

	尽管最新版本的 Node 已经支持绝大多数的 ES2015 特性，你可能仍然想要在你的测试中使用 ES modules 语法和 stage-x 的特性。为此我们需要安装 babel-jest
	
- 可以告诉 babel-preset-env 面向我们使用的 Node 版本。这样做会跳过转译不必要的特性使得测试启动更快

- 配置 .babelrc 文件

```js
{
  "presets": [["env", { "modules": false }]],
  "env": {
    "test": {
      "presets": [["env", { "targets": { "node": "current" } }]]
    }
  }
}
```

## 总结
其实就是按照参考资料一点点的安装依赖即可

这里唯一的坑就是需要安装  "vue-jest": "vuejs/vue-jest#next", 这个 next 版本


## 参考资料
- [Vue Test Utils](https://vue-test-utils.vuejs.org/zh/installation/)