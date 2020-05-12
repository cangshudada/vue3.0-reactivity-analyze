# Vue3.0 源码数据绑定简要解读



## 1.前言

本人一直认为vue的数据响应式机制是它的灵魂，这也是本人更喜欢vue的原因之一。在2019.10.5日尤大发布了`Vue3.0`预览版源码，其中的响应式机制也被ES6中的新语法重写了，本人整理出了Vue3.0数据绑定的实现原理供大家参考。

 

## 2.Vue2.x的数据绑定机制

读过`Vue2.x`源码的小伙伴肯定知道之前的数据绑定机制的原理，也就是利用`Object.defineProperty`来进行`拦截对象`，给对象的属性增加`set` 和 `get`方法，在`get`方法中收集依赖，在`set`方法中通知依赖更新视图，但是这种机制存在一定的缺陷：

- 需要深度递归遍历对象，浪费内存
- `Object.defineProperty`无法监听数组的变化，所以需要手动封装数组方法劫持
- 对象中越过`get/set`方法，直接增加键值对无法对新增的键值对进行数据绑定

以下简要说一下`vue2.x`的数据绑定机制： 



> ### 对象拦截

```javascript
function observer(target){
    // 如果不是对象数据类型直接返回即可
    if(!isObject(target)){
        return target
    }
    // 重新定义key
    for(let key in target){
        defineReactive(target,key,target[key])
    }
}

function isObject(target){
    return typeof target === "object" && target !== null;
}

function defineReactive(obj,key,value){
    if(isObject(value)){
        observer(value); //值为对象类型需要深层递归劫持
    }
    
    Object.defineProperty(obj,key,{
        get(){
            // 在get 方法中收集依赖
            return value
        },
        set(newVal){
            if(newVal !== value){
                // 为对象类型需要继续劫持
                if(isObject(value)){
                    observer(value);
                }
                update(); // 在set方法中触发更新
            }
        }
    })
}

function update(){
    console.log('update view')
}

let obj = {name:'youxuan'}
observer(obj);
obj.name = 'webyouxuan';
```



> ### 数组方法劫持

```javascript
const oldProtoMehtods = Array.prototype;
const proto = Object.create(oldProtoMehtods);
['push','pop','shift','unshift',...].forEach(method=>{
    Object.defineProperty(proto,method,{
        get(){
            update();
            oldProtoMehtods[method].call(this,...arguments)
        }
    })
})

function observer(target){
    // 如果不是对象数据类型直接返回即可
    if(typeof target !== 'object'){
        return target
    }
    
    // 如果为数组为数组添加自定义数据劫持方法
    if(Array.isArray(target)){
        Object.setPrototypeOf(target,proto);
        // 给数组中的每一项进行observr
        for(let i = 0 ; i < target.length;i++){
            observer(target[i])
        }
        return
    };
    // 重新定义key
    for(let key in target){
        defineReactive(target,key,target[key])
    }
}
```



此处对`Vue2.x`的数据绑定原理就不过多阐述了，接下来我们把焦点放在`Vue3.0`上来。



## 3.Vue3.0源码目录剖析

```base
├── packages
│   ├── compiler-core # 所有平台的编译器
│   ├── compiler-dom # 针对浏览器而写的编译器
│   ├── reactivity # 数据响应式系统
│   ├── runtime-core # 虚拟 DOM 渲染器 ，Vue 组件和 Vue 的各种API
│   ├── runtime-dom # 针对浏览器的 runtime。其功能包括处理原生 DOM API、DOM 事件和 DOM 属性等。
│   ├── runtime-test # 专门为测试写的runtime
│   ├── server-renderer # 用于SSR
│   ├── shared # 帮助方法
│   ├── template-explorer
│   └── vue # 构建vue runtime + compiler
```



**compiler**
`compiler-core`主要功能是暴露编译相关的`API`以及`baseCompile`方法
`compiler-dom`基于`compiler-core`封装针对浏览器的`compiler` (对浏览器标签进行处理)

**runtime**
`runtime-core` 虚拟 DOM 渲染器、Vue 组件和 Vue 的各种API
`runtime-test`将`DOM`结构格式化成对象，方便测试
`runtime-dom` 基于`runtime-core`编写的浏览器的`runtime` (增加了节点的增删改查，样式处理等)，返回`render`、`createApp`方法

**reactivity**
单独的数据响应式系统，核心方法`reactive`、`effect`、 `ref`、`computed`

**vue**
整合 `compiler` + `runtime`



## 4.Vue3.0初体验

由上面`Vue3.0`的目录结构来看整个项目还是非常清晰的，对于目前想要体验`vue3.0`的同学来说，官方脚手架也已经发布了支持vue-next版本的脚手架  [`vue-cli-plugin-vue-next`](https://github.com/vuejs/vue-cli-plugin-vue-next) ，具体体验方法如下：

```bash
# in an existing Vue CLI project
vue add vue-next
```

首先使用`vue-cli`初始化一个项目，然后在项目根目录下命令行中输入 `vue add vue-next`即可。

> 请注意：`vue-cli`版本必须更新到`v4.3.1`，并且`vue-router`以及`vuex`目前`vue3.0`并未支持。            — 20200511



以下是简单的`vue3.0`代码演示：

```vue
<template>
  <div id="app">
    <div>鼠标X坐标---{{x}}</div>
    <div>鼠标Y坐标---{{y}}</div>
  </div>
</template>

<script>
// 类似react hooks的方式，特有钩子直接从vue中导入
import { ref, onMounted, onUnmounted } from "vue";
    
// 特有逻辑函数剥离 vue3.0的好处    
function usePosition() {
  // 实时获取鼠标位置
  const x = ref(0);
  const y = ref(0);
  function update(e) {
    x.value = e.pageX;
    y.value = e.pageY;
  }
  onMounted(() => {
    window.addEventListener("mousemove", update);
  });
  onUnmounted(() => {
    window.removeEventListener("mousemove", update);
  });

  return {
    x,
    y
  };
}
    
    
export default {
  setup() {
    const { x, y } = usePosition(); // 使用公共逻辑
    return {
      x,
      y
    };
  }
};
</script>
```



## 5.Vue3.0数据绑定解析

在学习`Vue3.0`之前，必须要先熟练掌握ES6中的 [Proxy](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Proxy)、[Reflect](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/reflect) 及 [Map](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/map)、[Set](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/set)两种数据结构，如果不熟悉的同学建议先熟练了解这些知识。



我们首先看`Vue3.0`中是如何实现数据绑定的

```javascript
const person = Vue.reactive({name:'cangshudada'}); //person对象已经成为响应式数据
Vue.effect(()=>{ // effect方法会立即触发一次
    console.log(person.name);
})

person.name = '仓鼠大大';; // 当属性修改后会再次触发effect方法

```

> 源码是采用`ts`编写，由于可能有不熟悉ts的同学，这里我们采用js来从0编写实现原理，之后再看源码就会比较轻松啦！



### 5.1 reactive实现 

```javascript
/**
 *
 * @description 生成响应式对象
 * @param {any} target
 * @returns
 */
function reactive(target) {
    // 创建响应式对象
    return createReactiveObject(target);
}

/**
 *
 * @description 判断是不是object
 * @param {any} target
 * @returns {boolean}
 */
function isObject(target) {
    return typeof target === "object" && target !== null;
}

/**
 *
 * @description 创造响应式对象
 * @param {any} target
 * @returns
 */
function createReactiveObject(target){
    // 判断target是不是对象,不是对象直接返回
    if(!isObject(target)){
        return target;
    }
    
    // get set delete ...对象方法
    const handlers = {
        get(target,key,receiver){ // 取值
            let res = Reflect.get(target,key,receiver);
            return res;
        },
        set(target,key,value,receiver){ // 更改/新增属性
            let result = Reflect.set(target,key,value,receiver);
            return result;
        },
        deleteProperty(target,key){ // 删除属性
            const result = Reflect.deleteProperty(target,key);
            return result;
        }
    }
    // 开始代理
    observed = new Proxy(target,handlers);
    return observed;
}
let p = reactive({name:'cangshudada'});
console.log(p.name); // 取值
p.name = '仓鼠大大'; // 设置
delete p.name; // 删除
```

但是可能存在这样的对象

```javascript
const person ={
    name: 'cangshudada',
    age: 24,
    pets: {
        dog: {
            name: 'guagua',
            age: 1
        },
        cat: {
            name: 'gugu',
            age: 2
        }
    }
}
```

所以我们得继续实现**多层对象嵌套**情况下的代理：

```javascript
get(target, key, receiver) {
    // 取值
    const res = Reflect.get(target, key, receiver);
    return isObject(res) ? reactive(res) : res; // 懒代理，只有当取值时再次做代理，vue2.0中一上来就会全部递归增加getter,setter
}
```

我们继续考虑**数组**的情况

`Proxy`默认是可以支持数组，所以我们不需要像`Vue2.x`中一样对数组封装自己的方法并在其中来劫持监听数据改变，但是我们改变数组的时候仍然能够发现问题，那就是数组的改变会触发两次set，分别是**数组的长度变化**以及**索引值的变化**，接下来我们就需要屏蔽掉多次触发的问题。

```javascript
set(target, key, value, receiver) {
    const oldValue = target[key];
    const hadKey = target.hasOwnProperty(key);
    const result = Reflect.set(target, key, value, receiver);
    // 判断是否是新增还是修改的情况
    if (!hadKey) { //无key的情况则是新增
        trigger(target, 'add', key)
    } else if (oldValue !== value) { //防止数组重复操作修改索引或者length的时候多次触发set
        trigger(target, 'set', key)
    }
    return result;
}
```

此时数组的问题也解决了，最后就是对**同一对象重复代理的兼容**，这里我们利用[WeakMap ](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/WeakMap)来解决，这样完整的`reactive` 实现如下：

```javascript
const toProxy = new WeakMap(); // 存放被代理过的对象
const toRaw = new WeakMap(); // 存放已经代理过的对象

/**
 *
 * @description 生成响应式对象
 * @param {any} target
 * @returns
 */
function reactive(target) {
    // 创建响应式对象
    return createReactiveObject(target);
}


/**
 *
 * @description 判断是不是object
 * @param {any} target
 * @returns {boolean}
 */
function isObject(target) {
    return typeof target === "object" && target !== null;
}


/**
 *
 * @description 判断对象中是否有该键
 * @param {object} target
 * @param {object} key
 * @returns {boolean}
 */
function hasOwn(target, key) {
    return target.hasOwnProperty(key);
}

/**
 *
 * @description 创造响应式对象
 * @param {any} target
 * @returns
 */
function createReactiveObject(target) {

    // 是否是对象
    if (!isObject(target)) {
        return target;
    }

    // 判断取到被代理的对象
    let observed = toProxy.get(target);

    if (observed) { // 判断是否被代理过
        return observed;
    }
    if (toRaw.has(target)) { // 判断重复代理的情况,如果重复代理
        return target;
    }

    const handlers = {
        get(target, key, receiver) {
            // 取值
            const res = Reflect.get(target, key, receiver);
            track(target, 'get', key)；//收集依赖
            return isObject(res) ? reactive(res) : res; // 懒代理，只有当取值时再次做代理，vue2.0中一上来就会全部递归增加getter,setter
        },
        set(target, key, value, receiver) {
            const oldValue = target[key];
            const hadKey = hasOwn(target, key);
            const result = Reflect.set(target, key, value, receiver);
            // 判断是否是新增还是修改的情况
            if (!hadKey) { //无key的情况则是新增
                trigger(target, 'add', key) // 触发依赖更新 - 增加
            } else if (oldValue !== value) { //防止数组重复操作修改索引或者length的时候多次触发set
                trigger(target, 'set', key) // 触发依赖更新 - 修改
            }
            return result;
        },
        deleteProperty(target, key) {
            trigger(target, 'delete', key)；// 触发依赖更新 - 删除 
            const result = Reflect.deleteProperty(target, key);
            return result;
        }
    };

    // 开始代理
    observed = new Proxy(target, handlers);
    toProxy.set(target, observed);
    toRaw.set(observed, target); // 做映射表
    return observed;
}

// 对象的情况
const person = reactive({ name: 'cangshudada' });
console.log('person.name >>', person.name); // 获取
person.name = '仓鼠大大'; // 设置
delete person.name; // 删除
person.age = 12;//能够代理到直接在对象增加的键
person.age = 24

// 能够直接代理数组以及重复代理的情况
const ary = reactive([1, 2, 3, 4]);
ary.push(5)
const ary1 = reactive(ary); //此时重复代理会直接返回之前代理过的对象
```

> 到这里`reactive`方法已经基本实现完毕，接下来就是与`Vue2.x`中的逻辑一样进行依赖收集和触发依赖更新了，其中`track`的作用是对依赖进行收集，收集的主要是`effect`，`trigger`方法则是通知`effect`更新



### 5.2 effect实现

`effect`也就是副作用的意思，这个方法默认会在调用的时候率先执行一次，之后如果数据有变化后则会再次触发此回调函数。

```javascript
const person = Vue.reactive({name:'cangshudada'}); //person对象已经成为响应式数据
Vue.effect(()=>{ // effect方法会立即触发一次
    console.log(person.name);
})

person.name = '仓鼠大大';; // 当属性修改后会再次触发effect方法
```

我们先来实现`effect`函数

```javascript

/**
 *
 * @description effect函数
 * @param {function} fn 回调函数
 * @returns
 */
function effect(fn) {
    const effect = createReactiveEffect(fn); // 创建响应式的effect
    effect(); // 首先执行一次
    return effect;
}

// 存放响应式effect
const activeReactiveEffectStack = []; 

/**
 *
 *
 * @param {function} fn 回调函数
 * @returns
 */
function createReactiveEffect(fn) {
    const effect = function () {
        // 响应式的effect
        return run(effect, fn);
    };
    return effect;
}


/**
 *
 * @param {function} effect 响应式的effect
 * @param {function} fn 回调函数
 * @returns
 */
function run(effect, fn) {
    try {
        activeReactiveEffectStack.push(effect);
        return fn(); // 先让fn执行,执行时会触发get方法，可以将effect存入对应的key属性
    } finally {
        activeReactiveEffectStack.pop(effect);
    }
}
```

当调用`fn()`时可能会触发`get`方法，此时会触发上面**get**中调用的`track`函数

```javascript
const targetMap = new WeakMap();

function track(target,type,key){
    // 查看是否有effect
    const effect = activeReactiveEffectStack[activeReactiveEffectStack.length-1];
    if(effect){
        const depsMap = targetMap.get(target);
        if(!depsMap){ // 如果不存在依赖数组对象则添加Map对象
            targetMap.set(target,depsMap = new Map());
        }
        const deps = depsMap.get(target); 
        if(!deps){ //如果deps不存在则增加Set数组
            depsMap.set(key,(deps = new Set()));
        }
        if(!deps.has(effect)){ //如果deps中没有这个effect就将effect添加到依赖数组中
            deps.add(effect); 
        }
    }
}
```

当更新属性时会触发`trigger`执行，并根据`key`值找到对应的存储集合中的`effect`依次执行

```javascript
function trigger(target,type,key){
    const depsMap = targetMap.get(target);
    if(!depsMap){
        return
    }
    const deps = depsMap.get(key);
    if(deps){
        deps.forEach(effect=>{
            effect();
        })
    }
}
```

这个时候其实还存在`length`的问题，比如我们在`effect`中监听数组的`length`，这个时候因为我们上面在**set**函数中设置了`length`改变不触发`trigger`函数的机制，所以还需要在`trigger`中增加判断来兼容这种情况

```javascript
function trigger(target, type, key) {
  const depsMap = targetMap.get(target);
  if (!depsMap) {
    return;
  }
  const deps = depsMap.get(key);
  if (deps) {
    deps.forEach(effect => {
      deps();
    });
  }
  // 兼容处理当前更新类型是增加时，如果用到数组的length的effect应该也会被执行
  if (type === "add") {
    const lengthDeps = depsMap.get("length");
    if (lengthDeps) {
      lengthDeps.forEach(effect => {
        effect();
      });
    }
  }
}
```



### 5.3 ref实现

ref可以将原始数据类型同样转换成响应式数据，这个时候需要通过`.value`属性获取值

```javascript
/*
*
* @description 不同类型的数据响应式处理 如果是对象通过reactive函数进行数据绑定否则直接返回
*/
function convert(target) {
  return isObject(target) ? reactive(target) : target;
}

function ref(raw) {
  raw = convert(raw);
  const v = {
    _isRef:true, // 标识是ref类型
    get value() {
      track(v, "get", "");
      return raw;
    },
    set value(newVal) {
      raw = newVal;
      trigger(v,'set','');
    }
  };
  return v;
}
```

这个时候问题又来了，假如出现如下情况，则每次调用都得多加一个`.value`就会非常麻烦，所以我们也得对这种情况做个兼容

```javascript
const name = ref('cangshudada');
const person = reactive({
    c_Name: name
});
console.log(person.c_Name.value); // 每次调用c.a都得加上.value 比较麻烦
```

这个时候需要在`get`函数中兼容

```javascript
get(target, key, receiver) {
    // 取值
    const res = Reflect.get(target, key, receiver);
    // 兼容ref的value情况 因为前面的判断所以ref不可能为对象 可以直接返回
    if(res._isRef){
        return res.value
    }
    track(target, 'get', key)；//收集依赖
    return isObject(res) ? reactive(res) : res; // 懒代理
}
```



### 5.4 computed实现

之前版本的`computed`函数会缓存监听变量的值，只有当监听的变量值发生变化函数才会触发，在实际项目中用处非常大，如今`vue3.0`响应式数据机制重写，也导致了`computed`的重写，我们来看看在`vue3.0` `computed`是如何实现的，首先我们来看看用法

```javascript
const person = reactive({name:'cangshudada'});
const _computed = computed(()=>{
  console.log('computed执行了')  
  return `${person.name} --- xixi`;
})
// 不取_computed.value值则回调函数不执行，除非监听对象改变则取n次只执行一次
console.log(_computed.value);// computed执行了 cangshudada --- xixi
console.log(_computed.value);// cangshudada --- xixi
person.name = '仓鼠大大';
console.log(_computed.value);// computed执行了 仓鼠大大 --- xixi
```

computed实现

```javascript
function computed(fn){
  let dirty = true; // 第一次取值会触发
  const runner = effect(fn,{ // 标识这个effect是懒执行
    lazy:true, // 懒执行
    scheduler:()=>{ // 当依赖的属性变化了，调用此方法，而不是重新执行effect 依赖不更新则不更新dirty，进而不会触发runner()，缓存机制
      dirty = true;
    }
  });
  let value;
  return {
    _isRef:true,
    get value(){
      if(dirty){
        value = runner(); // 执行runner会继续收集依赖
        dirty = false;
      } 
      return value; // value没变化不会执行computed回调
    }
  }
}
```

修改`effect`函数 此处建议结合[effect实现](###5.2 effect实现)查看

```javascript
function effect(fn,options) {
  let effect = createReactiveEffect(fn,options);
  if(!options.lazy){ // 如果是lazy 则不立即执行
    effect();
  }
  return effect;
}

function createReactiveEffect(fn,options) {
  const effect = function() {
    return run(effect, fn);
  };
  effect.scheduler = options.scheduler;

  return effect;
}
```

在`trigger`时判断

```javascript
deps.forEach(effect => {
  if(effect.scheduler){ // 如果有scheduler 说明不需要执行effect
    effect.scheduler(); // 将dirty设置为true,下次获取值时变可以重新执行runner方法
  }else{
    effect(); // 否则正常执行effect即可
  }
});
```

```javascript
const person = reactive({name:'cangshudada'});
const _computed = computed(()=>{
  console.log('computed执行了')  
  return `${person.name} --- xixi`;
})
// 不取_computed.value值则回调函数不执行，除非监听对象改变则取n次只执行一次
console.log(_computed.value);
person.name = '仓鼠大大'; // 更改值 不会触发重新计算,但是会将dirty变成true
console.log(_computed.value); // 此时触发get函数进而调用runner()重新调用计算方法
```



## 6.总结

至此我们就将`Vue3.0`源码中的 `reactivity` 部分解析完毕了！了解了vue的数据绑定机制对于之后不管是面试还是后期的应用都有着很大的帮助，当然本篇文章只是对这部分进行了简要地解析，清楚了数据绑定这部分的逻辑与思想后再来读源码这部分相信各位会有更多的收获，目前`vue3.0`依旧在持续的更新，包括其周边生态，后期本人也会对其他模块进行解析，对本文章有相应建议的也欢迎提交Issues😆



 *by cangshudada*🐹