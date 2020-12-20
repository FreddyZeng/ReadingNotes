# 浏览器兼容

## [browserhacks](http://browserhacks.com/)

browserhacks 是一些仅仅在特别版本的浏览器上生效的CSS，为了在特定版本处理特殊的CSS异样行为达到兼容效果。

也就是说仅仅让部分CSS代码或者JS代码在某些环境生效和执行。也可以用UA来判断

- IE9及一下有IF语句判断浏览器版本，进行适配
- !important不算hack，算是不够友好的CSS代码。因为所有浏览器都会走这个逻辑
- Vendor prefixes大部分都不是hack，除了_:-ms-fullscreen某些特定的前缀，可以参考[browserhacks](http://browserhacks.com/)

[browserhacks](http://browserhacks.com/)除了有CSS hack，在IE环境下，也有JS hack判断版本号，或者指定CSS属性

# 参考

https://blog.csdn.net/freshlover/article/details/12132801

https://www.sitepoint.com/what-is-the-definition-of-a-css-hack/

http://browserhacks.com/#hack-f4ade0540d8e891e8190065f75acb186