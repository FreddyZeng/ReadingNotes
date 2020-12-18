# Vue知识点

Vue的知识点记录

# 源码理解

## V-Model实现细节

首先，此文是我阅读网络知识，从中梳理的来的。并且加入了自己的一些理解。如果有更有深度的想法或者我的理解存在错误（这可能会存在误人子弟。。很严重可以反馈给小弟，小弟改正。。）

首先我们的Vue文件一般是由3部分组成，HTML，CSS和JS。

而真正的逻辑，都在HTML和JS部分。Vue的编译器解析Vue模板的时候，本质上就是根据HTML元素的标签生成对应的ES5 JavaScript代码，然后再用JS动态创建HTML元素。（当然可以选择非编译模式，这样下载Vue文件，然后再用JS直接在线解析Vue语法，性能差一点）

### **Vue大致代码转换流程就是：**

**Vue编译器->编译出可执行的JS**（根据不同环境生成不同的JS）

### 为什么需要语法转换呢？

如果不用语法转换，也就是Vue的语法糖。

我们需要写很多JS代码去进行绑定。我们都知道render函数是触发依赖收集的，可是我们仅仅一个指令就能绑定响应，然后让它进行响应式了。

使用语法糖，然后底层自动转换成对应的ES5代码（也可以直接解析Vue文件）。这样方便我们开发者书写代码，提升效率。

### v-model的执行代码

所以我们直接思考V-Model的最终ES5执行代码，就能知道大概这个V-model做了什么。

**codegen->genData->genDirectives->directives/model.js**

- 在model.js会区分当前元素是input和select，还是自定义组件

**这时，我们需要区分对待，比较它们的最终生成结果**

当前的所有directives指令都会被执行，也就是执行对应的逻辑转换（为了把vue的语法糖转换成ES5代码）

**首先给出系统控件select和input的v-model ES5执行代码：**

```js
with(this) {
  return _c('div',[_c('input',{
    directives:[{
      name:"model",
      rawName:"v-model",
      value:(message),
      expression:"message"
    }],
    attrs:{"placeholder":"edit me"},
    domProps:{"value":(message)},
    on:{"input":function($event){
      if($event.target.composing)
        return;
      message=$event.target.value
    }}}),_c('p',[_v("Message is: "+_s(message))])
    ])
}
```

**我们再看自定义组件的v-model ES5执行代码：**

```js
with(this){
  return _c('div',[_c('child',{
    model:{
      value:(message),
      callback:function ($$v) {
        message=$$v
      },
      expression:"message"
    }
  }),
  _c('p',[_v("Message is: "+_s(message))])],1)
}
```

从上面可以得到：

- 原生的双向绑定是以directives为数据载体。在渲染原生input的时候render函数，激活expression的收集，并且，on为事件回调（类似依赖注入）
- 自定义的双向绑定是以model为数据载体，在渲染自定义组件render函数，从model取出expression，并且，callback为事件回调（类似依赖注入）

所以本质上，还是需要配合render函数，才能理解整个过程。

**而v-model，仅仅是一个vue语法糖转换到ES5的render调用过程的简化。**

具体的细节过程，可以参考引用的原文

### 参考

https://ustbhuangyi.github.io/vue-analysis/v2/extend/v-model.html#%E7%BB%84%E4%BB%B6