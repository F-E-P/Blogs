## mergeOptions选项合并策略
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

Vue.js在初始化的时候, 有些默认的配置,initGlobalAPI函数为Vue.options,进行了一些初始化的默认配置
```javascript
    function initGlobalAPI(Vue) {
    ...
    Vue.options = Object.create(null);
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

在上一节vm._init函数中,调用了mergeOptions函数, 进行选项的合并
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
            * 在这个调用了mergeOptions
            * 获取resolveConstructorOptions(vm.constructor)返回值
            */
            vm.$options = mergeOptions(resolveConstructorOptions(vm.constructor), options || {}, vm)
        }
        ...
    };
}
```
接下来看mergeOptions函数的实现:
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
            /*递归调用把emixins合并到parent上*/
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
通过分析mergeOptions函数, 主要做了一下几件事情:
- 检查组件的命名是否规范
- 规范化Props,Inject,Directives
- Vue选项的合并

##### 检查组件的命名是否规范checkComponents(child)
```javascript
 function checkComponents(options) {
    for (var key in options.components) {
        validateComponentName(key);
    }
}
```
将child的传递进来, 进行遍历, 获取到每个key. 将每个key作为参数传递给
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
isBuiltInTag函数, 不能是slot,component Vue内置组件的名字
```javascript
var isBuiltInTag = makeMap('slot,component', true);
```
isReservedTag函数, 组件的名字不能为html标签的名字和svg标签的名字
```
var isReservedTag = function (tag) {
    return isHTMLTag(tag) || isSVG(tag)
};
```

从validateComponentName分析得出组件的命名规范应该满足一下的要求:
- /^[a-zA-Z][\-\.0-9_/.test(name) 为true
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
         options[key] = strat(parent[key], child[key], vm, key);
     }
     return options
}
```
上面的代码, 最终返回一个options,  先对parent进行遍历, 在对child进行遍历, 在遍历child时,
多了一层的限制, 对于相同的key, 如果parent已经进mergeField, child,就不在进行遍历.

mergeField函数:
- 先去检测strats[key], 对该key是否有自定义的合并策略, 如果有就直接使用,像el,watch,data等都进行自定义的合并策略
- 如果没有自定义的策略, 就是用是默认的策略合并


对于strats[key]函数, 什么是strats?
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


在看看defaultStrat的源码实现, childVal === undefined直接使用parentparentVal
```javascript
 var defaultStrat = function (parentVal, childVal) {
        return childVal === undefined? parentVal : childVal
    };
```
