---
title: per-vue-project
tags:
  - Vue
date: 2019-10-18 17:00:22
---
# Vue项目优化实践

## 工具函数导入方式优化
例如公共的格式化类工具函数，要使用它们首先要引入，然后还要把它们一个个挂到组件实例上才能被模版使用，非常不方便，例如：

```jsx
// formatter.js
function formatDate() {}
// 此处省略大量...

// sth.vue
<script>
import { formatDate, ...大量函数 } from 'formatter'

export default {
    methods: {
        formatDate,
        ...大量函数
    }
}
</script>
```

- 通过 ```mixin```解决：将所有的```format```函数放到```mixin```里，使用时直接```mixin```混入。  
优点：避免了繁琐的引入  
缺点：方法使用时来路不明，大量冗余方法挂载到实例上  
不可取。

- 借鉴```vuex```的```mapGetter```方式：

```js
// utils/tools
export function pick(obj, keys = []) {
  return keys.reduce((o, key) => {
    if (!obj[key]) {
      console.error(`pick error: 不存在${key}～`)
    }

    o[key] = obj[key]

    return o
  }, {})
}

// index.js
import * as config from './formatters'
import { pick } from 'utils/tools'

export * from './formatters'

const obj = Object.assign({}, config)

// 由于ElementUI表格上的formatter函数参数顺序问题，这里调整参数顺序
Object.entries(config).forEach(([key, fn]) => {
  obj[`_${key}`] = (row, column, value, index) => {
    return fn(value, undefined, index, row, column)
  }
})

export function mapFormatters(keys = []) {
  return pick(obj, keys)
}
```

此时组件内部就可以这样使用：

```jsx
// sth.vue
<script>
import { mapformatters } from 'formatter'

export default {
    methods: {
        ...mapformatters([
            'formatDate',
            ...大量函数名称
        ])
    }
}
</script>
```

## 接口调用方面

### Promise or async/await
- 只有一个请求使用```promise```即可，多个请求嵌套时使用```async/await```

```js
/* 嵌套 */
// promise
a(() => {
  b(() => {
    c()
  })
})

// async/await
await a()
await b()
await c()
```

- 但是如果有两个并发的异步请求，在都完成后do something：

```js
//错误
await a()
await b()
//这样变成了 a().then(() => b() )
// a 好了才会执行 b
done()

//正确
await Promise.all([a(), b()])
done()
```

## 命名方面
参考Vue官方[风格指南](https://cn.vuejs.org/v2/style-guide/index.html)
- 单文件组件的文件名应该要么始终是单词大写开头 (PascalCase)，要么始终是横线连接 (kebab-case)。
- 组件名为多个单词必要
组件名应该始终是多个单词的：  
根组件 ```App``` 以及 ```<transition>、<component>``` 之类的 ```Vue``` 内置组件除外。  
这样做可以避免跟现有的以及未来的 HTML 元素相冲突，因为所有的 HTML 元素名称都是单个单词的。


## Watch immediate
当 watch 一个变量的时候，初始化时并不会执行，如下面的例子，你需要在created的时候手动调用一次。

```js
// bad
{
    created() {
        this.checkedMap = {...this.defaultFilters}
    },
    watch: {
        defaultFilters () {
            this.checkedMap = {...this.defaultFilters}
        }
    }
}

// good
{
    watch: {
        defaultFilters: {
            handler: () => {
                this.checkedMap = {...this.defaultFilters}
            },
            immediate: true
        }
    },
}
```
::: tip  
ps: watch 还有一个容易被大家忽略的属性deep。当设置为true时，它会进行深度监听。简而言之就是你有一个 const obj={a:1,b:2}，里面任意一个 key 的 value 发生变化的时候都会触发watch。应用场景：比如我有一个列表，它有一堆query筛选项，这时候你就能deep watch它，只有任何一个筛序项改变的时候，就自动请求新的数据。或者你可以deep watch一个 form 表单，当任何一个字段内容发生变化的时候，你就帮它做自动保存等等。  
:::

## 表单参数处理

### Computed 的 get 和 set
假如表单填写的值需要特殊处理后再传给后端：  
当然我们可以直接在访问接口时处理参数，但是参数很多，很多都需要处理会导致访问接口都函数变得过于臃肿。这时可以使用 ```Computed 的 get 和 set```

::: warning
下面只是一个示例！
:::

```html
<template>
    <el-form :model="formModel">
        <el-form-item prop="pwd">
            <el-input v-model="pwd"></el-input>
        </el-form-item>
    </el-form>
</template>

<script>
const crypt = {
    encrypt(word) {},   // 加密
    decrypt(word) {}    // 解密
}

export default {
    data() {
        return {
            formModel: {}
        }
    },
    computed: {
        pwd: {
            get() {
                return crypt.decrypt(this.formModel.pwd)
            },
            set(val) {
                this.formModel.pwd = crypt.encrypt(val)
            }
        }
    }
}
</script>
```

## Object.freeze

```this.item = Object.freeze(Object.assign({}, this.item))```对于接口返回都大量数据，可以使用它来优化性能，因为当你把一个普通的 JavaScript 对象传给 Vue 实例的 data 选项，Vue 将遍历此对象所有的属性，并使用 Object.defineProperty 把这些属性全部转为 getter/setter，它们让 Vue 能进行追踪依赖，在属性被访问和修改时通知变化。

## 组件通讯方面

### props和事件
- 没有Typescript为了支持类型检查props一定要写成对象形式：

```js
{
    // props: ['val'], // 不建议这样使用
    props: {
        val: {
            type: String,
            required: true,
            default: '',
            validator: value => new Set(['success', 'warning', 'danger']).has(value)
        }
    }
}
```

- props是父组件传给自组件的，子组件永远不要尝试直接去修改它，应该使用事件的方式:

```jsx
// 父组件
<the-component @event-name="eventHandler">

// 子组件
this.$emit('event-name'/*, some arguments of eventHandler*/)
```

- attrs: 获取子传父中未在 props 定义的值  
    - 场景1：如果子组件需要大量来自父组件的值，那么就需要在自组件定义大量props，此时可以考虑使用attrs。  
    *但是为了降低子组件对父的耦合还是不建议这样使用。*
    - 场景2: 当我们要封装第三方组件时，第三方组件可能拥有很多自定义的属性和事件，此时我们添加自己的属性和事件的同时还要保证被封装组件的属性和事件被暴露出去，例如：

```html
<template>
    <div>
        <third-part-component v-bind="$attrs" v-on="$listeners" />
    </div>
    <footer></footer>
</template>

<script>
    export default {
        inheritAttrs: false, // 下面会说
        props: {
            // ...
        },
        methods: {
            handler() {
                // 自定义事件
                this.$emit('event-name', ...)
            }
        },
        // sth
    }
</script>
```

- inheritAttrs [官方文档](https://cn.vuejs.org/v2/api/#inheritAttrs)  
默认为```true```，此时父组件传递所有的属性都会被应用到子组件的根节点上。改为```false```可去掉这一默认行为。

###  $root
整棵组件树的根实例

###  .sync
实现双向绑定的语法糖，[官方文档](https://cn.vuejs.org/v2/guide/components-custom-events.html#sync-%E4%BF%AE%E9%A5%B0%E7%AC%A6)

### v-slot
2.6.0 新增 1.slot,slot-cope,scope 在 2.6.0 中都被废弃,但未被移除 2.作用就是将父组件的 template 传入子组件 3.插槽分类: A.匿名插槽(也叫默认插槽): 没有命名,有且只有一个;[官方文档](https://cn.vuejs.org/v2/api/#v-slot)

### 路由传参

方案一 ```path/:id```:

```js
// 路由定义
{
  path: '/describe/:id',
  name: 'Describe',
  component: Describe
}
// 页面传参
this.$router.push({
  path: `/describe/${id}`,
})
// 页面获取
this.$route.params.id
```

方案一 ```params```:

```js
// 路由定义
{
  path: '/describe',
  name: 'Describe',
  omponent: Describe
}
// 页面传参
this.$router.push({
  name: 'Describe',
  params: {
    id: id
  }
})
// 页面获取
this.$route.params.id
```

方案三 ```query```:

```js
// 路由定义
{
  path: '/describe',
  name: 'Describe',
  component: Describe
}
// 页面传参
this.$router.push({
  path: '/describe',
    query: {
      id: id
  }
)
// /describe?id=123
```

三种方案对比 方案二参数不会拼接在路由后面,页面刷新参数会丢失 方案一和三参数拼接在后面,丑,而且暴露了信息

### Vue.observable
2.6.0 新增
用法:让一个对象可响应。Vue 内部会用它来处理 data 函数返回的对象;  
返回的对象可以直接用于渲染函数和计算属性内，并且会在发生改变时触发相应的更新;  
也可以作为最小化的跨组件状态存储器，用于简单的场景。  
通讯原理实质上是利用Vue.observable实现一个简易的 vuex:

```js
// store.js
import Vue from 'vue'

export const store = Vue.observable({ count: 0 })
export const mutations = {
  setCount (count) {
    store.count = count
  }
}
```

```html
<template>
    <div>
        <label for="bookNum">数 量</label>
            <button @click="setCount(count+1)">+</button>
            <span>{{count}}</span>
            <button @click="setCount(count-1)">-</button>
    </div>
</template>

<script>
import { store, mutations } from 'store.js' // Vue2.6新增API Observable

export default {
  name: 'Add',
  computed: {
    count () {
      return store.count
    }
  },
  methods: {
    setCount: mutations.setCount
  }
}
</script>
```

## render 函数
场景:有些代码在 template 里面写会重复很多,所以这个时候 render 函数就有作用啦[官方文档](https://cn.vuejs.org/v2/guide/render-function.html)

## 异步组件
场景:项目过大就会导致加载缓慢,所以异步组件实现按需加载就是必须要做的事啦 1.异步注册组件 3种方法

```js
// 工厂函数执行 resolve 回调
Vue.component('async-webpack-example', function (resolve) {
  // 这个特殊的 `require` 语法将会告诉 webpack
  // 自动将你的构建代码切割成多个包, 这些包
  // 会通过 Ajax 请求加载
  require(['./my-async-component'], resolve)
})

// 工厂函数返回 Promise
Vue.component(
  'async-webpack-example',
  // 这个 `import` 函数会返回一个 `Promise` 对象。
  () => import('./my-async-component')
)

// 工厂函数返回一个配置化组件对象
const AsyncComponent = () => ({
  // 需要加载的组件 (应该是一个 `Promise` 对象)
  component: import('./MyComponent.vue'),
  // 异步组件加载时使用的组件
  loading: LoadingComponent,
  // 加载失败时使用的组件
  error: ErrorComponent,
  // 展示加载时组件的延时时间。默认值是 200 (毫秒)
  delay: 200,
  // 如果提供了超时时间且组件加载也超时了，
  // 则使用加载失败时使用的组件。默认值是：`Infinity`
  timeout: 3000
})
```

### 路由按需加载

```js
{
  path:'/',
  name:'home',
  components:()=>import('@/components/home')
}
```
## 动态组件
场景:做一个 tab 切换时就会涉及到组件动态加载

```html
<component v-bind:is="currentTabComponent" />
<!--优化-->
<keep-alive>
  <component v-bind:is="currentTabComponent" />
</keep-alive>
```

## 递归组件
场景:如果开发一个 tree 组件,里面层级是根据后台数据决定的,这个时候就需要用到递归组件
递归组件必须设置name和递归终止条件，```v-if=终止条件```

## 函数式组件
无状态，无法实例化，只是一个函数，开销低，[官方文档](https://cn.vuejs.org/v2/guide/render-function.html#%E5%87%BD%E6%95%B0%E5%BC%8F%E7%BB%84%E4%BB%B6)

```html
<template functional>
  <div v-for="(item,index) in props.arr">{{item}}</div>
</template>
```

## Vue.extend

场景:vue 组件中有些需要将一些元素挂载到元素上,这个时候 extend 就起到作用了 是构造一个组件的语法器 写法:

```js
const Modal = Vue.extend({
    template: '<div>This is a modal</div>',
    props: {
        title: {
            type: String,
            default: '标题'
        }
    }
})

// 挂载
new Modal({
    propsData: {
        title: 'Modal'
    }
}).$mount('#modal')
```

## 插件

通过```Vue.use(Plugin)```安装注册插件，通过安装插件可以添加全局资源（例如：组件等），添加Vue实例方法[官方文档](https://cn.vuejs.org/v2/guide/plugins.html)

## Vue.nextTick
组件的```mounted```生命周期dom并未渲染完毕可以通过：
```js
{
    mounted() {
        this.$nextTick(() => 获取更新后的dom)
    }
}
```

## Vue.set()
场景:当你利用索引直接设置一个数组项时或你修改数组的长度时,由于 Object.defineprototype()方法限制,数据不响应式更新 不过vue.3.x 将利用 proxy 这个问题将得到解决 解决方案:

```js
// 利用 set
this.$set(arr,index,item)

// 利用数组 push(),splice()
```

## v-pre

场景:vue 是响应式系统,但是有些静态的标签不需要多次编译,这样可以节省性能

## v-cloak

场景:在网速慢的情况下,在使用vue绑定数据的时候，渲染页面时会出现变量闪烁
用法:这个指令保持在元素上直到关联实例结束编译。和 CSS 规则如 [v-cloak] { display: none } 一起用时，这个指令可以隐藏未编译的 Mustache 标签直到实例准备完毕

## v-once

v-once 和 v-pre 的区别: v-once只渲染一次；v-pre不编译,原样输出

## .vue组件的deep 属性

有的时候我们想要修改某组件内部的样式，但是由于```<style scoped>```加了```scoped```导致这里写的选择器上都加上了```[data-039c5b43]```类似的自定义属性，例如：  

```html
<style lang="less" scoped>
    .container .red { color: #f00 }
</style>

<!-- 编译后 -->
<style type="text/css">
    .container[data-039c5b43] .red[data-039c5b43] { color: #f00 }
</style>
```

然而组件内部对应的标签上并不存在```.red[data-039c5b43]```选择器，因此无法修改到组件内部的样式。  
因此官方提供了deep属性：

```html
<!-- 上面样式加一个 /deep/ -->
<style lang="less" scoped>
    .container /deep/ .red { color: #f00 }
</style>

<!-- 编译后 -->
<style type="text/css">
    .container[data-039c5b43] .red { color: #f00 }
</style>
```
