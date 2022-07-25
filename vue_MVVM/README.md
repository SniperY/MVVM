# MVVM  

杨昭彤 南京大学 软件工程 qq号：1515479726

###  双向数据绑定 Model View View Model  

1. Angular1.x当中的双向数据绑定是通过 监听的方式来出来的，核心思想为脏值检查,angular 通过$watch()去监听值然后调用  $apply()/ $digest()方法来实现的。

2. Vue 数据劫持 + 发布订阅模式（不兼容低版本）+ 数据代理；



我们通常写 Vue 的时候，都会这样写：

```
<body>
  <div id="app">
      {{kecheng}}
  </div>
</body>

<script>
    let mvvm = new Mvvm({
        el: '#app',
        data:{ kecheng:'百度'}
    })

</script>

```


用这个 Object.defineProperty 这个属性来实现数据劫持（Observer）。

数据劫持：观察对象，通过递归给每一个对象增加 Object.definePropery，在 set 方法中触发 observe 方法，就能监听到数据的变化，如果数据类型是 `{a:{b:1}}`多层的，那么就要用到递归去实现。

```
function observe (data) {
    return new Observe(data)
}

function Observe (data) {
    if(!data || typeof data !== 'object')  return
    //把data属性 通过Object.definePropert  来定义属性
    for (let key in data) {   
        let value = data[key]  
        //递归方式绑定所有属性    数据是 {a:{b:1}}
        observe(value)
        Object.defineProperty(data, key, {
            enumerable:true,
            get() {
                return value
            },
            set(newValue) {
                //如果值没有发生改变的话  
                if(newValue  == value) return
                //重新赋值
                value = newValue
                observe(value)
            },
        })
    }
}


```

### Vue 中的数据代理   


遇到一些比较复杂的数据结构，例如 `data:{ kecheng:'百度', msg:{vx:*******,creator: 'yzt' }}`。

若使用 observe 方法，当需要获取 creator 字段的话，需要通过 `mvvm._data.msg.creator ..... ` 的形式来获取值。遇到再复杂的数据结构就会更乱。然而若想要通过 `mvvm.msg` 方式来获取数据（去掉_data）。去掉复杂的查询方式，所以用到了数据代理的方式来处理以上问题，其中 this 代表的是整个数据。


```
   //数据代理方式
   for(let key in data ) {
     Object.defineProperty(this, key ,{
        enumerable:true,
        get() {
            return this._data[key];
        },
        set(newValue){
            this._data[key] = newValue
        }
     })
   }

```
Vue特点，不能新增不存在的属性 ，因为不存在的属性没有 get、set 方法。而vue当中的深度响应，会给每一个新对象增加数据劫持，从而去监控新对象的变化。


### 模板编译 Compile

```
function Compile(el,vm) {
  vm.$el = document.querySelector(el);
  var Fragment = document.createDocumentFragment();
  //把模板放入内存当中
  while (child = vm.$el.firstChild) {
    Fragment.appendChild(child)
  }
  replace(Fragment,vm)
  vm.$el.appendChild(Fragment)
}

function replace (Fragment,vm){
    //类数组转化成数组
  Array.from(Fragment.childNodes).forEach(function(node){
    var text = node.textContent ;
    var reg = /\{\{(.*)\}\}/;
    if(node.nodeType == '3' && reg.test(text)) {
        //console.log(RegExp.$1)
        let ary = RegExp.$1.split('.')
        //console.log(ary)  [msg, vx]   
        let val = vm  
        ary.forEach(function(key){  //取this.msg  /this.gongzhonghao
            val = val[key]
        });
        node.textContent = text.replace(reg,val)
    }
    if(node.childNodes){
        replace(node,vm)
    }
  })
}

```

### 发布订阅模式

```
observe(value)
Object.defineProperty(data, key, {
    enumerable:true,
    get() {
        console.log(Dep.target)  
        Dep.target && dep.addSub(Dep.target)  //[增加watcher]
        return value
    },
    set(newValue) {
        //如果值没有发生改变的话
        if(newValue  == value) return
        //重新赋值
        value = newValue
        observe(value)
        dep.notify()  //让所有的watcher的update 执行
    },
})

```

##### **增加订阅**  


```
  //绑定的方法都有一个update属性
  function Dep (){
    this.subs = []  //订阅器
  }
  //增加订阅
  Dep.prototype.addSub = function(watcher) {
     this.subs.push(watcher)
  }

```
##### **发布订阅**

```
//通知
Dep.prototype.notify = function() {
   this.subs.forEach(watcher => watcher.update())
}


```
##### **观察者**

```
function Watcher(vm,exp,fn){
   this.vm = vm
   this.exp = exp
   this.fn = fn         //添加到订阅中
   Dep.target = this
   var val  = vm
   var arr = exp.split('.');
   arr.forEach(function(key){
       val = val[key]
   })     //在这里调用objectDefineProperty中get方法
   Dep.target = null
}
Watcher.prototype.update = function() {
  //获取新值
  var val  = this.vm
  var arr = this.exp.split('.');
  arr.forEach(function(key){
      val = val[key]
  })
  this.fn(val);
}

```

#### 发布之后，修改模板



```
  发布订阅模式开启
  new Watcher(vm, RegExp.$1, function(newValue){
    node.textContent = text.replace(reg,newValue)
  })

```

以上4步完事之后，就可以实现了 Vue 当中的发布订阅模式。

### 如何实现 input 标签当中的v-model 


```

 <input type="text" v-model='kecheng'>

```

在编译的时候我们要判断节点，当 nodeType == 1 的时候，获取 DOM 的属性来判断 type 类型，如果是想要的 v-model 的话就去监听当前元素，并开启发布订阅模式去监听变化。

```
  //当前为标签的时候
    if(node.nodeType == '1'){
      var nodeAttrs = node.attributes //获取当前节点DOM属性
      //console.log(nodeAttrs)  //NamedNodeMap {0: type, 1: v-model, type: type, v-model: v-model, length: 2}
      Array.from(nodeAttrs).forEach((attr) => {
         var name = attr.name
         var key = attr.value  //v-model = 'value'
         if (name.indexOf('v-') == 0) {
           node.value = vm[key]
         }
         //发布订阅模式开启
         new Watcher(vm, key, function(newValue){
           node.value = newValue
         })

         node.addEventListener('input',function(e){
            var newValue = e.target.value
            vm[key] = newValue
         })
      })

```

### 如何使用 computed 计算时进行缓存处理

首先要是去获取当前 computed 值。

```

function computed() { //具有缓存功能
    let computed = this.$options.computed;  // {key:value}
    let self = this
    Object.keys(computed).forEach(function(key){   //拿到key 值
      Object.defineProperty(self, key,{
        get: typeof computed[key] === 'function' ?  computed[key] : computed[key].get,
        set(){},
      })
    })
}

```
### 实现一个 MVVM 

MVVM 作为数据绑定的入口，整合 Observer、Compile 和 Watcher 三者，通过 Observer 来监听自己的 Model 数据变化，通过 Compile 来解析编译模板指令，最终利用 Watcher 搭起 Observer 和 Compile 之间的通信桥梁，达到数据变化 -> 视图更新，视图交互变化（input） -> 数据 Model 变更的双向绑定效果。 

```
    let mvvm = new Mvvm({
        el: '#app',
        data:{ kecheng:'百度', msg:{vx:********,creator: 'yzt' }},
        //computed 可以缓存 只是把数据挂在mvvm上
        computed:{
          say(){
            return this.kecheng  + '--------' + this.msg.creator
          }
        }
    })

```

```
function Mvvm(options = {}) {
    //将所有的数据绑定在$options
   this.$options = options

   var data = this._data = this.$options.data;
   //增加发布订阅
   observe(data)

   //数据代理方式
   proxyData(data,this)
   //计算
   computed.call(this)
   //模板编译
   compile(options.el,this)
}

```
Vue 中的 MVVM，这里主要还是利用了 Object.defineProperty() 这个方法来劫持了 vm 实例对象的属性的读写权，使读写 vm 实例的属性转成读写了 this._data 的属性值。

### 总结 

本文主要围绕着实现 Observer、Compile、computed 、proxyData 几个方式展开。

