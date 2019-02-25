### initState-开启响应式系统入口

在 vm._init 中 调用调了  initState
```javascript
initState(vm);
```
 initState(vm) 源码实现具体如下:
 ```javascript
function initState(vm) {
    vm._watchers = [];
    /* 获取实例的options */
    var opts = vm.$options;
    /*如果设置了props, initProps初始化*/
    if (opts.props) {
        initProps(vm, opts.props);
    }
     /*如果设置了methods, initMethods初始化*/
    if (opts.methods) {
        initMethods(vm, opts.methods);
    }
    /*如果设置了data, initData初始化*/
    if (opts.data) {
        initData(vm);
    } else {
        /*响应式的入口*/
        observe(vm._data = {}, true /* asRootData */);
    }
     /*如果设置了computed, initComputed初始化*/
    if (opts.computed) {
        initComputed(vm, opts.computed);
    }
    if (opts.watch && opts.watch !== nativeWatch) {
        initWatch(vm, opts.watch);
    }
}
 ```
从上面的源码可以知道: 进行了一些初始化的操作, 在这里只分析 initData(vm), initState 是最重要的,
initProps, initMethods等大致的实现原理都是一样的

```javascript
/*如果设置了data, initData初始化*/
if (opts.data) {
    initData(vm);
} else {
    /*响应式的入口*/
    observe(vm._data = {}, true /* asRootData */);
}
```
接下来看下initData(vm)实现:
```javascript
function initData(vm) {
    /*获取到data,此时的data已经是一个函数*/
    var data = vm.$options.data;
    /*检测data是否是一个函数.如果是通过getData获取到data的返回值.*/
    data = vm._data = typeof data === 'function'
        ? getData(data, vm)
         /如果不是一个函数,就返回data或者{}空对象*/
        : data || {};
        /*data 此时已经获取到, 是对象  getData的会返回值 */
    if (!isPlainObject(data)) {
        data = {};
        warn(
            'data functions should return an object:\n' +
            'https://vuejs.org/v2/guide/components.html#data-Must-Be-a-Function',
            vm
        );
    }
    // proxy data on instance
    var keys = Object.keys(data);
    var props = vm.$options.props;
    var methods = vm.$options.methods;
    var i = keys.length;
    while (i--) {
        var key = keys[i];
        {
            if (methods && hasOwn(methods, key)) {
                warn(
                    ("Method \"" + key + "\" has already been defined as a data property."),
                    vm
                );
            }
        }
        if (props && hasOwn(props, key)) {
            warn(
                "The data property \"" + key + "\" is already declared as a prop. " +
                "Use prop default value instead.",
                vm
            );
        } else if (!isReserved(key)) {
            proxy(vm, "_data", key);
        }
    }
    // observe data
    observe(data, true /* asRootData */);
}
```
initData() 函数整理流程:
- 此时data 的是一个函数. 获取到 data 的返回值.
- vm 代理 data 的属性
- 最终加入到响应式系统, 由此开启了响应式入口

接下里每一步分析 initData() 实现原理

###### 获取data函数的返回值
```javascript
 /*获取到data,此时的data已经是一个函数*/
var data = vm.$options.data;
 /*检测data是否是一个函数.如果是通过getData获取到data的返回值.*/
data = vm._data = typeof data === 'function'
    ? getData(data, vm)
    : data || {};
/* 运行到此处: data应该是一个对象, 如果不是对象, 发生报错处理*/
if (!isPlainObject(data)) {
    data = {};
    warn(
        'data functions should return an object:\n' +
        'https://vuejs.org/v2/guide/components.html#data-Must-Be-a-Function',
        vm
    );
}
```
通过 vm.$options.data 获取到 data,  检测 data 是否是一个函数,
- data是一个函数,调用getData()函数
- 不是一个函数,返回 data 或者 {} 对象.

将最终的结果 赋值给 vm._data 变量

getData(data, vm)函数的实现:
```javascript
function getData(data, vm) {
    // #7573 disable dep collection when invoking data getters
    pushTarget();
    try {
        /*核心代码,  此时data是一个函数, 通过 call 调用, 获取返回值*/
        return data.call(vm, vm)
    } catch (e) {
        handleError(e, vm, "data()");
        return {}
    } finally {
        popTarget();
    }
}
```
getData()函数, 最终通过 data.call(), 将返回值返回去出去.  返回值是一个对象.

###### vm 代理 data 的里的属性
```javascript
// proxy data on instance
/* 获取到data的key的数组 */
var keys = Object.keys(data);
/*获取到props*/
var props = vm.$options.props;
/*获取到methods*/
var methods = vm.$options.methods;
var i = keys.length;
while (i--) {
    /*获取到每个key的名字*/
    var key = keys[i];
    {
        /*检测methods里面不能有与data相同的名字的key*/
        if (methods && hasOwn(methods, key)) {
            warn(
                ("Method \"" + key + "\" has already been defined as a data property."),
                vm
            );
        }
    }
     /*检测props里面不能有与data相同的名字的key*/
    if (props && hasOwn(props, key)) {
        warn(
            "The data property \"" + key + "\" is already declared as a prop. " +
            "Use prop default value instead.",
            vm
        );
      /*检测key的名字不能以_或者$开头, Vue中有很多以_或者$开头的属性,避免造成混乱*/
    } else if (!isReserved(key)) {
        proxy(vm, "_data", key);
    }
}
```
在获取到data, props, methods, 进行对每项属相的的名字检测.
props, methods里面每个属性的名字不能与data的key有相同的名字 原因是:
- 因为vm实例 都对象他们进行了代理

proxy(vm, "_data", key) 的实现:
```javascript
/* 定义一个公共的实现, 在prosps, 和methods都会用到. */
var sharedPropertyDefinition = {
    enumerable: true,  /*可以进行枚举*/
    configurable: true,  /*可以进行删除属性*/
    get: noop,  /*noop 是一个空函数, 常用套路*/
    set: noop
};
/* 如果代理vm代理 data , target是vm, sourceKey是data, key属性*/
function proxy(target, sourceKey, key) {
    /*对key进行get操作,设置sourceKey[key]*/
    sharedPropertyDefinition.get = function proxyGetter() {
        return this[sourceKey][key]
    };
    /*对key进行set操作, 返回sourceKey[key]的值*/
    sharedPropertyDefinition.set = function proxySetter(val) {
        this[sourceKey][key] = val;
    };
    /*使用Object.defineProperty对数据进行相关拦截的操作*/
    Object.defineProperty(target, key, sharedPropertyDefinition);
}
```
如果此时vm 要代理 data, proxy函数的三个参数:
- target 是vm
- sourceKey 是data
- key 是外部通过vm.xxx

获取vm.xxx 最终通过Object.defineProperty, 会触发 get钩子函数, 返回data属性里对应的key的值

设置vm.xxx 最终通过Object.defineProperty, 会触发 set钩子函数, 设置data属性里对应的key的值

vm 代理 props 属性 和 methods 的属性也是一样的原理, 在此不再进行分析.

###### 响应式系统入口
在initData最后一行的代码:  真正的开启响应式系统

```javascript
  observe(data, true /* asRootData */);
```
更多精彩, 将关注下一节

