## mergeOptions选项合并策略
mergeOptions的主要作用:
- 对 options 进行规范
- options 的合并, 默认策略和自定义策略

合并策略目的:围绕着组件和子类来进行限制的

```javascript
const vm= new Vue({
        el:"#app",
        data:{
            test:"这是一个测试"
        }
    })
```
当在控制台打印vm.$options可以看到多了几个属性

![](/images/vue/1.vue.jpg)

Vue.js在初始化的时候, 有些默认的配置, initGlobalAPI 函数为 Vue.options, 进行了一些初始化的默认配置
```javascript
    function initGlobalAPI(Vue) {
    ...
    Vue.options = Object.create(null);
    /*对资源assets的默认初始化*/
    ASSET_TYPES.forEach(function (type) {
        Vue.options[type + 's'] = Object.create(null);
    });

    // this is used to identify the "base" constructor to extend all plain-object
    // components with in Weex's multi-instance scenarios.
    Vue.options._base = Vue;

    extend(Vue.options.components, builtInComponents);
    ...
}
```

在上一节 vm._init 函数中,调用了 mergeOptions 函数, 进行选项的合并
```javascript
function initMixin(Vue) {
    Vue.prototype._init = function (options) {
        var vm = this;
        ...
        // merge options
        if (options && options._isComponent) {
            // optimize internal component instantiation
            // since dynamic options merging is pretty slow, and none of the
            // internal component options needs special treatment.
            initInternalComponent(vm, options);
        } else {
            /*
            * 在这个调用了mergeOptions函数
            * 获取resolveConstructorOptions(vm.constructor)返回值
            */
            vm.$options = mergeOptions(resolveConstructorOptions(vm.constructor), options || {}, vm)
        }
        ...
    };
}
```
接下来看 mergeOptions 函数的实现:
```javascript
/* 用于把parent,child进行合并 */
function mergeOptions(parent, child, vm) {
   /*检测options里,组件的名字命名规范*/
    {
        checkComponents(child);
    }
    /*child也可以是是一个函数*/
    if (typeof child === 'function') {
        child = child.options;
    }
    /*规范化Props*/
    normalizeProps(child, vm);
    /*规范化Inject*/
    normalizeInject(child, vm);
    /*规范化Directives*/
    normalizeDirectives(child);

    // Apply extends and mixins on the child options,
    // but only if it is a raw options object that isn't
    // the result of another mergeOptions call.
    // Only merged options has the _base property.
    if (!child._base) {
        if (child.extends) {
            /*递归调用把extends合并到parent上*/
            parent = mergeOptions(parent, child.extends, vm);
        }
        if (child.mixins) {
            /*递归调用把mixins合并到parent上*/
            for (var i = 0, l = child.mixins.length; i < l; i++) {
                parent = mergeOptions(parent, child.mixins[i], vm);
            }
        }
    }
    /*最终合并完成要返回的$options, vm.$options对象*/
    var options = {};
    var key;
    for (key in parent) {  /*先判断parent上是否存在key*/
        mergeField(key);
    }
    for (key in child) {
        if (!hasOwn(parent, key)) { /*判断parent以后,在判断child是上否有key*/
            mergeField(key);
        }
    }
    /* 合并字段 */
    function mergeField(key) {
        /*根据不同的 */
        var strat = strats[key] || defaultStrat;
        options[key] = strat(parent[key], child[key], vm, key);
    }
    /*返回最终合并完成的options, 会赋值给Vue.$options*/
    return options
}
```
通过分析 mergeOptions 函数, 主要做了一下几件事情:
- 检查组件的命名是否规范
- 规范化 Props,Inject,Directives
- Vue 选项的合并
mergeOptions的第三个参数 vm, 用于区分根实例还是子组件. 在上面的代码中, 传递了vm参数,
mergeOptions在另一个也被调用了,在 Vue.extend() 这个函数中, 没有传递 vm 参数


##### 检查组件的命名是否规范checkComponents(child)
```javascript
 function checkComponents(options) {
    for (var key in options.components) {
        validateComponentName(key);
    }
}
```
将 child 的传递进来, 进行遍历, 获取到每个 key. 将每个 key 作为参数传递给
validateComponentName(key);
```javascript
function validateComponentName(name) {
    if (!new RegExp(("^[a-zA-Z][\\-\\.0-9_" + unicodeLetters + "]*$")).test(name)) {
        warn(
            'Invalid component name: "' + name + '". Component names ' +
            'should conform to valid custom element name in html5 specification.'
        );
    }
    if (isBuiltInTag(name) || config.isReservedTag(name)) {
        warn(
            'Do not use built-in or reserved HTML elements as component ' +
            'id: ' + name
        );
    }
}
```
isBuiltInTag 函数, 不能是 slot,component Vue 内置组件的名字
```javascript
var isBuiltInTag = makeMap('slot,component', true);
```
isReservedTag 函数, 组件的名字不能为html标签的名字和svg标签的名字
```
var isReservedTag = function (tag) {
    return isHTMLTag(tag) || isSVG(tag)
};
```

从 validateComponentName 分析得出组件的命名规范应该满足一下的要求:
- /^[a-zA-Z][\-\.0-9_/.test(name) 为 true
- isBuiltInTag(name) 或者 config.isReservedTag(name) 为 false

##### Vue选项的合并
```javascript
/*最终合并完成要返回的$options, vm.$options对象*/
    var options = {};
    var key;
    /*对parent进行遍历*/
    for (key in parent) {
        mergeField(key);
    }
    /*对child进行遍历*/
    for (key in child) {
        /*hasOwn多了一层的判断, 如果child的key, 已经在上面parent进行了mergeField*/
        /*就不在进行遍历, 避免重复调用*/
        if (!hasOwn(parent, key)) {
            mergeField(key);
        }
    }
     /*对parent, child的key字段进行和合并,采取了不同的策略*/
     function mergeField(key) {
         var strat = strats[key] || defaultStrat;
         /*把 strat(parent[key], child[key], vm, key)函数的返回值给对应的options[key]*/
         options[key] = strat(parent[key], child[key], vm, key);
     }
     return options
}
```
上面的代码, 最终返回一个 options,  先对 parent 进行遍历, 在对 child 进行遍历, 在遍历 child 时,
多了一层的限制, 对于相同的key, 如果 parent 已经进 mergeField, child,就不在进行遍历.

mergeField 函数:
- 先去检测 strats[key], 对该 key 是否有自定义的合并策略, 如果有就直接使用,像 el, watch, data等都进行自定义的合并策略
- 如果没有自定义的策略, 就是用是默认的策略合并


对于 strats[key] 函数, 什么是 strats?
```javascript
/**
 * Option overwriting strategies are functions that handle
 * how to merge a parent option value and a child option
 * value into the final value.
 */
var strats = config.optionMergeStrategies;
```
config.optionMergeStrategies 是一个合并选项的策略对象，这个对象下包含很多函数，
这些函数就可以认为是合并特定选项的策略
el, data, watch等进行了合并的限制


在看看 defaultStrat 函数的实现, childVal === undefined 直接使用 parentVal
```javascript
var defaultStrat = function (parentVal, childVal) {
    return childVal === undefined? parentVal : childVal
};
```
