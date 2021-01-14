# Vue的初始化

- Vue是一个类，限制new初始化。执行_init初始化

## _init函数

### 初始化过程

- uid标记
- _isVue判断是否启动响应式功能
- 初始化支持Vue的响应式功能，_isComponent，初始化的options参数，是否为组件级别的参数
- 调用一些钩子函数，创建前，创建后，挂载前，挂载后。

### mount结构

- Vue.prototype.$mount。**每个平台都不同**和**是否编译环境运行**决定了mount逻辑，会导致mount的逻辑存在一定差异。通过function闭包特征进行向上调用通用的逻辑。

### 挂载过程

**配置更新逻辑**->vm.update和vm.render，**配置更新逻辑钩子**。通知完成挂载阶段

- options.template是一个字符串转换为模板。el是DOM对象，或者通过query把字符串转换为DOM对象。
- compileToFunctions：template转换为渲染函数，并且设置再options的render和staticRenderFns上
- mountComponent执行公共的挂载逻辑
  - 设置updateComponent组件更新逻辑
  - 对组件添加watch功能，并且设置组件的逻辑，配置更新前的逻辑
- 通过虚拟节点$vnode判断是否满足挂载逻辑，调用已经完成挂载
- 标记该组件已经渲染

给组件vm指定DOM对象->配置vm组件的更新回调函数(调用组件vm.update->vm.render函数)->根据组件vm创建的watcher对象设置渲染回调和数据层变化时的回调。watcher是否主动触发渲染。->通知钩子生命周期mounted完成挂载。

## render











