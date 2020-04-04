# Vue 网络上来源的例子

## 生命周期

```
<template>
	<div>
		helloworld
	</div>
</template>


<script>
  export default{
    data(){
      return{

      }
    },
    beforeCreate(){

	},
	created(){

	},
	beforeMount(){

	},
	mounted(){

	}
  }

</script>
```



## 数据绑定，computed

```
<template>
	<div>
		<div>选项/数据</div>
		<div>----------------------------------------------</div>
		<div>data: {{msg}}</div>
		<div>----------------------------------------------</div>
		<div>computed: {{aDouble}}</div>
		<div>----------------------------------------------</div>
		<div @click="say('hi')">点击我</div>
	</div>
</template>
<script>
	export default {
		data () {
		    return {
		        msg: 'Welcome to Your Vue.js App',
		        num: 1
		    }
	    },
	    computed:{
	    	aDouble(){
	    		return this.num*2;
	    	}
	    },
	    methods: {
	    	say(h){
	    		alert(h);
	    	}
	    }
	}
</script>
```



## 指令v-html=""		v-bind:[属性]=""		v-on:点击函数=""		message | capitalize是过滤

```
<template>
	<div>
		<div>模板语法</div>
		<div>-----------------------------------------</div>
		<div>
			data: {{msg}}
		</div>

		<div>------------------------------------------</div>

		<div>{{ number + 1 }}</div>

		<div>------------------------------------------</div>

		<div v-html="rawHtml"></div>
		<div>------------------------------------------</div>
		
		<div v-bind:class="red"></div>

		<div v-on:click="say('hi')">点击我</div>

		<div>------------------------------------------</div>

		<!--简写方式-->
		<div @click="change">修改颜色</div>

		<div>-------------------------------------------</div>

		<div> {{ message | capitalize }} </div>

		

	</div>
</template>
<script>	
	export default{
		data(){
			return {
				msg: '数据绑定',
				number: 5,
				rawHtml: '<span>123</span>',
				red: 'red active',
				message:'message'
			}
		},
		methods:{
			change(){
				this.red='blue';
			},
			say(h){
				alert(h);
			}
		},
		filters:{
			capitalize(){
				return '123';
			}
		}
	}
</script>
```



## 计算属性

```
<template>
	<div>
		<div>计算属性</div>
		<div>-------------------------------------------</div>
		<div>{{reversedMessage}}</div>

	</div>
</template>

<script>
	export default {
		data(){
			return {
				msg : 'helloworld'
			}
		},
		computed:{
			reversedMessage(){
				return this.msg.split('').reverse().join('')
			}
		}
	}
</script>
```

## 绑定方式，动态绑定三种方式

```
<template>
	<div>
		<div>Class与style绑定</div>
		<div>----------------------------------</div>

		<!-- <div class="static"
			v-bind:class="{ 'active': isActive, 'text-danger': hasError }">
			class1
		</div> -->
		
		<!-- 第一种绑定方式 -->
		<div v-bind:class="{'active' : isActive,'text-danger': hasError}">
			class1
		</div>
		
		<!-- 第二种绑定方式 -->
		<div :class="classObject">class2</div>
		
		<!-- 第三种绑定方式 -->
		<div :class="[activeClass, errorClass]">class3</div>
		

		<div>----------------------------------</div>
		
		<div style="color: red; fontSize: 36px">123</div>

		<div v-bind:style="{ 'color': activeColor, 'fontSize': fontSize + 'px' }">style1</div>

		<div v-bind:style="styleObject">style2</div>

		<div v-bind:style="[baseStyles, overridingStyles]">style3</div>
	</div>
</template>


<script>
	export default{
		data(){
			return{
				isActive: false,
				hasError: true,
				classObject: {
					'active': true,
    				'text-danger': true
				},
				activeClass: 'active',
				errorClass: 'text-danger',

				activeColor: 'black',
				fontSize: 30,

				styleObject:{
					'color': 'red',
					'fontSize': '43px'
				},


				baseStyles:{
					color: 'red'
				},
				overridingStyles:{
					fontSize: '53px'
				}
			}
		}

	}
</script>
```

## 条件渲染 v-if 和 v-show

```
<template>
	<div>
		<div>条件渲染</div>
		<div>-----------------------------------</div>

		<h1 v-if="ok">Yes</h1>

		<div>-----------------------------------</div>
		
		<div v-if="type === 'A'">
		    A
		</div>
		<div v-else-if="type === 'B'">
		    B
		</div>
		<div v-else-if="type === 'C'">
		    C
		</div>
		<div v-else>
		    Not A/B/C
		</div>
		
		<!-- var type = 'A'
		if( type === 'A' ){
			A
		}else if( type === 'B' ){
			B
		}else if( type === 'C' ){
			C
		}else{
			Not A/B/C	
		} -->
		

		<div>----------------------------------------</div>

		<div v-show="isShow">123</div>
	</div>
</template>
<script>
	export default{
		data(){
			return{
				ok : false,
				type: 'D',
				isShow: true
			}
		}
	}
</script>
```

## v-for循环指令，创建列表

```
<template>
  <div>
    <div>列表渲染</div>
    <div>---------------------------------------</div>
    <ul id="example-1">
      <li v-for="item in items">
        {{ item.message }}
      </li>
    </ul>
    <div>---------------------------------------</div>
    <ul id="example-2">
      <li v-for="(item, index) in items">
        {{ index }} - {{ item.message }}
      </li>
    </ul>
    <div>---------------------------------------</div>
    <ul id="example-3">
      <li v-for="(value, key) in obj">
        {{ key }} : {{ value }}
      </li>
    </ul>
  </div>
</template>
<script>
export default {
  data() {
    return {
      items: [
        { message: 'Foo' },
        { message: 'Bar' }
      ],
      obj: {
        firstName: 'John',
        lastName: 'Doe',
        age: 30
      }
    }
  }
}

</script>

```

## 表单绑定

```
<template>
	<div>
		<div>表单控件绑定</div>
		<div>--------------------------------------</div>

		<input v-model="message" >
		<p>Message is: {{ message }}</p>
		

		<div>---------------------------------------</div>

		<!-- 多个勾选框 -->
		<input type="checkbox" value="Jack" v-model="checkedNames">

		<input type="checkbox" value="Boy" v-model="checkedNames">

		<input type="checkbox" value="Bob" v-model="checkedNames"> 
		 
		<br>
		<span>Checked names: {{ checkedNames }}</span>


		<div>--------------------------------</div>
		<!-- 选择列表 -->
		<div>
			<select v-model="selected">
				<option disabled value="">请选择</option>
				<option>A</option>
				<option>B</option>
				<option>C</option>
			</select>
			<span>Selected: {{ selected }}</span>
		</div>

	</div>
</template>

<script>
	export default{
		data(){
			return {
				message: 'helloworld',
				checkedNames: [],
				selected:''	
			}
		},
		mounted(){
			// alert(this.$route.params.userId)

			alert(this.$route.query.plan)
		}
	}
</script>
```

## 自定义组件

外接传值给组件  props

$emit，调用父组件的函数，进行回调通知

```
<template>
	<p :style="{color: col}">{{time}}</p>
</template>


<script>
	export default{
		data(){
			return{
				time: 10
			}
		},
		mounted(){
			var vm = this;
			var t = setInterval(function(){
				vm.time--;
				if(vm.time == 0){
					vm.$emit("end");
					clearInterval(t);
				}
			},1000)
		},
		props:{
			col:{
				type: String,
				default: '#000'
			}
		}
	}
</script>
```



```
<template>
	<div>
		<div>自定义组件</div>
		<div>--------------------------------</div>

		<count-down col="blue" @end="ending"></count-down>
	</div>
</template>


<script>
	import countdown from '@/components/countdown.vue'// 导入组件

	export default{
		data(){
			return{

			}
		},
		methods:{
			ending(){// 组件的回调
				alert("已经结束了")
			}
		},
		components:{
			'count-down':countdown// 声明组件标签
		}
	}
</script>
```

## vue操作DOM

```
<template>
	<div>
		<div>Vue中DOM操作</div>
		<div>--------------------------------------</div>

		<div ref="head" id="head"></div>
	</div>
</template>

<script>
	export default{
		data(){
			return{

			}
		},
		mounted(){

			// DOM已经生成，可以在这里进行JQ操作
			this.$refs.head.innerHTML = 'helloworld';
		}
	}
</script>
```

## 组件过度效果，动画     transition标签

```
<template>
	<div>
		<div>过渡效果</div>
		<div>--------------------------------------</div>

		<div id="demo">
		  	<button v-on:click="show = !show">
		    	Toggle
		  	</button>


		  	<transition name="fades">
		    	<p v-if="show">hello</p>
		  	</transition>

		  	<!-- 1、显示状态   opcity: 1   变到   0
		  	2、隐藏   
            leave   离开   leave --- leave-active  ----- leave-to


            1、隐藏  opcity: 0   变到   1
            2、显示
            enter   进入   enter 0 --- enter-active  ------  enter-to  1 -->

		</div>
	</div>
</template>


<script>
	export default{
		data(){
			return{
				show: true
			}
		}
	}
</script>

<style>
	.fades-enter-active, .fades-leave-active {
	  	transition: opacity .5s
	}
	.fades-enter, .fades-leave-to {
	  	opacity: 0
	}
	.fades-enter-to{
		opacity: 1
	}
</style>
```

## 路由的跳转

在main.js 导入路由，import router from './router'，并且注册

```
import router from './router'
/* eslint-disable no-new */
new Vue({
  el: '#app',
  router,
  //render: h => h(App)
  template: '<App/>',
  // 注册成组件
  components: { App }
})
```

在router/index.js下导入页面vue文件

```
import demo15 from '@/pages/demo15/index'
import demo16 from '@/pages/demo16/index'
import demo17 from '@/pages/demo17/index'

Vue.use(Router)

export default new Router({
  routes: [
    {
      path: '/',
      name: 'HelloWorld',
      component: HelloWorld
    },
    {
      path: '/demo1',
      name: 'demo1',
      component: demo1
      // component: require("@/pages/demo1/index")
    },
```



```
<template>
	<div>
		<div>vue-router</div>
		<div>-------------------------------------</div>

		<router-link to="/demo9">demo9</router-link>

		<div>-------------------------------------</div>

		<router-link :to="{ name: 'demo9', params: { userId: 456 }}">params</router-link>
		

		<div>-------------------------------------</div>
		<router-link :to="{ name: 'demo9', params: { userId: 456 }, query: { plan: 'private' }}">query</router-link>


		<div>-------------------------------------</div>

		<button @click="toURL">跳转页面</button>


	</div>
	<!-- https://router.vuejs.org/zh-cn/ -->

	<!-- http://localhost:8080/#/demo9/456? plan = private -->

	<!-- http://localhost:8080/#/demo9/456?plan=private -->

</template>

<script>
	export default{
		data(){
			return{

			}
		},
		methods:{
			toURL(){
				// this.$router.push({ path: '/demo8' })

				 //this.$router.push({ name: 'demo9', params: { userId: 123 }})
				 
				this.$router.push({ name: 'demo9', params: { userId: 123 }, query: { plan: 'private' } }) 
			}
		}
	}
</script>
```



## 状态管理 VueX，类似iOS的NSUserdefault

在main.js 导入路由，import store from './store'，并且注册

```
/* eslint-disable no-new */
new Vue({
  el: '#app',
  store,
  //render: h => h(App)
  template: '<App/>',
  // 注册成组件
  components: { App }
})

```

定义状态熟悉

actions是提供给页面调用修改数据层state，提供修改能力

mutations是定义逻辑封装，修改state

```
import Vue from 'vue'
import Vuex from 'vuex'

Vue.use(Vuex)

export default new Vuex.Store({
	state: {
	    count: 0,
	    num: 1
	},
	mutations: {
	    increment (state, num) {
	      state.count++
	      state.num = num;
	    }
	},
	actions: {
	    inc ({ commit }, obj) {
	      	commit('increment', obj)
	    }
	}
})
```



```
<template>
	<div>
		<div>状态管理vuex</div>
		<div>--------------------------------------</div>

		<div>{{msg}}</div>

		<button @click="change">change</button>

	</div>
	<!-- https://vuex.vuejs.org/zh -->
</template>

<script>
	export default{
		data(){
			return{
				msg: '123',
			}
		},
		methods:{
			change(){
				// vuex取数据
				// this.msg = this.$store.state.count;

				// 修改公共数据
				this.$store.dispatch('inc', 100000);
				this.msg = this.$store.state.num;
			}
		}
	}
</script>
```



## slot插槽，在组件中，抽象某些占位的布局，让使用控件的时候才定义。有点依赖反转的感觉，UI到底要显示成什么样子，不是依赖组件，而且依赖调用者

```
<template>
	<div>
		<h1>插槽测试</h1>
		<slot>
			
		</slot>

		<p>我是最底部</p>
		<slot name="bottom">
			
		</slot>
	</div>
</template>
```

```
<template>
	<div>
		<div>Slot插槽</div>
		<div>-------------------------------------</div>

		<slots>
			<div>123</div>
			<p>234</p>

			<p>666</p>

			<p slot="bottom">888</p>
		</slots>
	</div>
</template>

<script>

	import slots from '@/components/slot.vue';

	export default{
		data(){
			return{

			}
		},
		components:{
			slots
		}
	}
</script>
```



## vue-resource网络请求

main.js中配置

```
import vueResource from 'vue-resource'
Vue.use(vueResource);
```

```
<template>
	<div>
		<div>vue-resource请求</div>
		<div>-----------------------------------------</div>
		<!-- https://github.com/pagekit/vue-resource -->


	</div>
</template>


<script>
	export default{
		data(){
			return{

			}
		},
		mounted(){
			this.$http.get('/someUrl').then(response => {
		    	
		    	console.log(response.body);
		  	}, response => {
		    	// error callback
		  	});


		  	this.$http.post('/someUrl', {foo: 'bar'}).then(response => {
		    	console.log(response.body);
		  	}, response => {
		    	// error callback
		  	});


		  	// GET /someUrl?foo=bar
			this.$http.get('/someUrl', {params: {foo: 'bar'}, headers: {'X-Custom': '...'}}).then(response => {
			    // success callback
			}, response => {
			    // error callback
			});
		}
	}
</script>
```



## 第三方UI库mintUI

在main.js配置

```
import MintUi from 'mint-ui'
import 'mint-ui/lib/style.css'

Vue.use(MintUi);

```

```
<template>
  <div>
    <div>移动组件库Mint UI</div>
    <div>-------------------------------------</div>

    <mt-tabbar>
      <mt-tab-item id="外卖">
        <img slot="icon" src="../../assets/100x100.png"> 外卖
      </mt-tab-item>
      <mt-tab-item id="订单">
        <img slot="icon" src="../../assets/100x100.png"> 订单
      </mt-tab-item>
      <mt-tab-item id="发现">
        <img slot="icon" src="../../assets/100x100.png"> 发现
      </mt-tab-item>
      <mt-tab-item id="我的">
        <img slot="icon" src="../../assets/100x100.png"> 我的
      </mt-tab-item>
    </mt-tabbar>

  </div>
</template>
<script>
import Vue from 'vue'
import { Toast } from 'mint-ui';

import { MessageBox } from 'mint-ui';

import { Tabbar, TabItem } from 'mint-ui';

Vue.component(Tabbar.name, Tabbar);
Vue.component(TabItem.name, TabItem);

export default {
  data() {
    return {

    }
  },
  mounted() {
    //Toast('提示信息');

    //MessageBox('提示', '操作成功');
  }
}

</script>

```

