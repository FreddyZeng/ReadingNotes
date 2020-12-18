# jQuery

jQuery是一个JS库，本质上是封装元素ES5 JS的一些操作，如获取到HTML元素，事件回调，网络请求，HTML动画等

- 插件支持，如常用的[jquery-cookie](https://github.com/carhartl/jquery-cookie)管理cookie
  [jquery-plugin-validate](https://www.runoob.com/jquery/jquery-plugin-validate.html)，管理表单的验证

## 行为

CSS选择器查询HTML元素

根据一定的条件过滤查询到的HTML元素

设置逻辑事件回调

- 事件函数以及带有逻辑事件的函数如one

- 事件回调函数中的事件对象event

ajax络请求及配置网络请求状态的UI

操作当前的HTML元素的内部属性，如attr，CSS。或者访问HTML元素内容，获取HTML宽高等属性

操作HTML元素在HTML document树中的结构

操作HTML的元素过度动画



# jQuery插件

## jquery-plugin-validate

validate它是封装了一些常用的表单验证逻辑，以及展现错误或成功提示，还有回调拦截。并且简化验证代码的逻辑，让其清晰可维护。类似的组件有vee，[mobileValidate](https://github.com/efri-yang/mobileValidate)， [validatejs](https://validatejs.org/validatejs)

- 表单检验逻辑是发生在提交网络请求之前的行为
- 表单检验逻辑的意义是用户提交正确的数据格式和不缺少字段，并且做到提示作用，指引用户交互成功

### 组成

validate是由method和rule组成的。

method是一些检验逻辑函数，而rule就是把method逻辑函数按规则匹配到HTML元素上。

**所以如果我们需要自己写一个控件，可以参考这个库的思想，1是逻辑检验函数的定义，2是把表单的input输入选项用一定的规则和逻辑检验函数关联起来。一般来说，只有移动端Web和小程序需要注意库的引入大小，此时考虑裁剪第三方库，或者自己写一个简易的库**

### 定义验证配置

#### 替换默认的全局提示

```js
$.extend($.validator.messages, {
    required: "这是必填字段",
    remote: "请修正此字段",
    email: "请输入有效的电子邮件地址",
    url: "请输入有效的网址",
    date: "请输入有效的日期",
    dateISO: "请输入有效的日期 (YYYY-MM-DD)",
    number: "请输入有效的数字",
    digits: "只能输入数字",
    creditcard: "请输入有效的信用卡号码",
    equalTo: "你的输入不相同",
    extension: "请输入有效的后缀",
    maxlength: $.validator.format("最多可以输入 {0} 个字符"),
    minlength: $.validator.format("最少要输入 {0} 个字符"),
    rangelength: $.validator.format("请输入长度在 {0} 到 {1} 之间的字符串"),
    range: $.validator.format("请输入范围在 {0} 到 {1} 之间的数值"),
    max: $.validator.format("请输入不大于 {0} 的数值"),
    min: $.validator.format("请输入不小于 {0} 的数值")
});
```

#### validate检验规则可以写在HTML标签上

```js
<input id="cname" name="name" minlength="2" type="text" required>
```

上面的HTML标签的**minlength**和**required**当作规则，插件会兼容。这种方式不是特别好，最好用JS的rule字段和message字段控制

#### validate检验规则可以写在HTML标签上

```js
<input id="firstname" name="firstname" type="text">

$("#signupForm").validate({
    rules: {
      username: {
        required: true,
        minlength: 2
      },
    messages: {
      username: {
        required: "请输入用户名",
        minlength: "用户名必需由两个字符组成"
      },
    }
})
```

如上面所示，仅仅只需要绑定ID就可以JS层配置控制数据检验。逻辑清晰，高内聚

#### 自定义规则

addMethod(name,method,message)方法

### 选择元素

#### 可以对部分元素，进行忽略检验

```
ignore: ".ignore" // 对.ignore类的元素忽略
```

### 触发验证

| onsubmit     | Boolean | 提交时验证。设置为 false 就用其他方法去验证。                | true  |
| ------------ | ------- | ------------------------------------------------------------ | ----- |
| onfocusout   | Boolean | 失去焦点时验证（不包括复选框/单选按钮）。                    | true  |
| onkeyup      | Boolean | 在 keyup 时验证。                                            | true  |
| onclick      | Boolean | 在点击复选框和单选按钮时验证。                               | true  |
| focusInvalid | Boolean | 提交表单后，未通过验证的表单（第一个或提交之前获得焦点的未通过验证的表单）会获得焦点。 | true  |
| focusCleanup | Boolean | 如果是 true 那么当未通过验证的元素获得焦点时，移除错误提示。避免和 focusInvalid 一起用。 | false |

### 显示验证提示

#### 不通过验证

**自定义错误显示位置**errorPlacement：Callback

error的值可以是string类型，此时是成功后的样式。如果是function类型，成功后会执行这个回调函数

**自定义错误显示样式**error（标签会自动带上error class）

#### 通过验证

success的值可以是string类型，此时是成功后的样式。如果是function类型，成功后会执行这个回调函数

### 回调事件拦截

```js
$().ready(function() {
 $("#signupForm").validate({
        submitHandler:function(form){
            alert("提交事件!");   
            form.submit();
        }    
    });
});
// 或者使用ajax发出请求
 $(".selector").validate({     
 submitHandler: function(form) 
   {      
      $(form).ajaxSubmit();     
   }  
 }) 
```

### 主动的行为事件

**重置表单**

```
validator.resetForm
```

# 参考

https://jquery.cuishifeng.cn/

https://jqueryvalidation.org/documentation/

https://www.runoob.com/jquery/jquery-plugin-validate.html