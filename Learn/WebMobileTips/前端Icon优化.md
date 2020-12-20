# 前端icon使用优化

## 位图icon和SVG的对比

- SVG比位图小，不失真
- 位图可以有丰富的颜色，而SVG一般是单色。现在好像支持多色了

**如果是单色图标多，建议优先使用SVG为先**

再来看下两者的使用

## 位图icon的使用

一般来说，就是使用雪碧图方案，把小icon拼接成一张图来加载。减少HTTP数量请求（浏览器最多6-8个并发）

## SVG的使用

### **iconfont的使用**

- 使用unicode
- 使用Class，本质上就是通过Class一一映射来加载对应的unicode
- 使用SVG格式指定iconfont的图标的Class来加载对应的iconfont的元素

**问题**

iconfont的缺点是不能按需加载，虽然说icon的本身很小。但是如果iconfont越来越大，可能会形成性能问题

所以，最好可以让页面的icon按需加载

### **按需加载SVG**

#### 使用[svg-sprite-loader](https://github.com/kisenka/svg-sprite-loader) 来按需加载SVG图标，

**[svg-sprite-loader](https://github.com/kisenka/svg-sprite-loader)原理**
在Vue文件编译的时候，根据当前那个页面使用到了SVG图标，然后在HTML页面上内联按需插入SVG图片数据，可以从[Inline SVG vs Icon Fonts](https://css-tricks.com/icon-fonts-vs-svg/)这里了解内联SVG

```html
<svg>
<symbol id='xx'>/** 在这里定义svg的icon数据 **/</symbol>
<symbol id='yy'>/** 在这里定义svg的icon数据 **/</symbol>
</svg>
```

然后使用use在指定的HTML元素上渲染SVG icon

```html
<svg>
<use xlink:href="#icon-QQ" x="50" y="50" />
</svg>
```

#### [svg-sprite-loader](https://github.com/kisenka/svg-sprite-loader) 的webpack配置

- 需要指定一个文件夹用来存放SVG图片，并且[svg-sprite-loader](https://github.com/kisenka/svg-sprite-loader) 仅仅处理者里面的icon图片。
- 然后全局的SVG加载器，需要忽略SVG类型的icon路径。

#### 使用[SVGO](https://github.com/svg/svgo)优化SVG的文件质量

它是一个可以对SVG文件不必要的内容进行删除，而不影响图片的渲染

#### 优化SVG的使用

##### 对svg-use进行封装

```html
<svg>
<use xlink:href="#icon-QQ" x="50" y="50" />
</svg>
```

如上图，我们的SVG都需要使用use加载，可以考虑封装一个组件来包装上面的SVG使用代码
这样子做有什么好处呢？

- 简化svg-use的使用
- 项目全局代码使用封装，如果需要对SVG进行未来的适配或者拓展更多的可用性，仅仅维护组件就可以了

我把参考文章的实例代码抄过来

```vue
//components/Icon-svg
<template>
  <svg class="svg-icon" aria-hidden="true">
    <use :xlink:href="iconName"></use>
  </svg>
</template>

<script>
export default {
  name: 'icon-svg',
  props: {
    iconClass: {
      type: String,
      required: true
    }
  },
  computed: {
    iconName() {
      return `#icon-${this.iconClass}`
    }
  }
}
</script>

<style>
.svg-icon {
  width: 1em;
  height: 1em;
  vertical-align: -0.15em;
  fill: currentColor;
  overflow: hidden;
}
</style>



作者：花裤衩
链接：https://juejin.cn/post/6844903517564436493
来源：掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```

```vue
//引入svg组件
import IconSvg from '@/components/IconSvg'

//全局注册icon-svg
Vue.component('icon-svg', IconSvg)

//在代码中使用
<icon-svg icon-class="password" />

作者：花裤衩
链接：https://juejin.cn/post/6844903517564436493
来源：掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```

##### [require-context](https://webpack.js.org/guides/dependency-management/#require-context)对引入的SVG图标进行全局引入

在使用SVG的文件的时候，我们都需要import导入到对应的代码代码引用环境中

**所以我们考虑使用批量导入，逻辑是在SVG ICON文件夹下的所有SVG文件都自动批量导入到代码引用环境中**

[require-context](https://webpack.js.org/guides/dependency-management/#require-context)特征

在某个文件夹下，使用正则匹配来导入多个文件，并且可以控制是否便利子文件夹，并且可以控制mode是同步导入，还是异步

- require('./template/' + name + '.ejs');
  是异步导入
- require.context(directory, useSubdirectories = true, regExp = /^\.\/.*$/, mode = 'sync');
  默认mode是同步导入，可以修改
- require.context的keys匹配出来所有可导入的模块。require.context返回一个函数，可以传keys中的对象，进行导入。这样就能理解下面的行为了，require.context返回函数根据key导入后，返回一个模块对象，缓存起来了

```js
const cache = {};

function importAll (r) {
  r.keys().forEach(key => cache[key] = r(key));
}

importAll(require.context('../components/', true, /\.js$/));
// At build-time cache will be populated with all required modules.
```

##### SVG使用总结

svg-use封装组件和require.context的导入逻辑，都需要在最外层的第一层控件下进行逻辑初始化方能使用。

**一切都是为了程序可用性提高，性能提高，提升代码书写规则减少维护难度，加快代码书写效率和拓展性。**

# 参考

https://juejin.cn/post/6844904158244372494

https://webpack.js.org/guides/dependency-management/#require-context

https://www.jianshu.com/p/c894ea00dfec

