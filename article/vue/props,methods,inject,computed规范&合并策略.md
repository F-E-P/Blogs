##  props directives 规范&props methods inject computed合并策略

#### props 规范化

在平时开发中, props 传递属性可以动态传递和静态传递
```javascript
 /*静态检测*/
 props:["message"]
 /*动态检测*/
 props:{
    message:{
        type:String
        default:"Vue"
    }
 }
```
初始化过程, 在 mergeOptions 函数里面, 可以看到对 props 的规范检测, 进行统一的规范化处理
```
normalizeProps(child, vm);
```
接下来分析得源码实现:
```javascript
/**
 * Ensure all props option syntax are normalized into the
 * Object-based format.
 */
function normalizeProps(options, vm) {
    var props = options.props;
    /*没有props直接返回*/
    if (!props) {
        return
    }
    var res = {};
    var i, val, name;
    if (Array.isArray(props)) {
        i = props.length;
        while (i--) {
            val = props[i];
            if (typeof val === 'string') {
                name = camelize(val);
                res[name] = {type: null};
            } else {
                warn('props must be strings when using array syntax.');
            }
        }
    } else if (isPlainObject(props)) {
        for (var key in props) {
            val = props[key];
            name = camelize(key);
            res[name] = isPlainObject(val)
                ? val
                : {type: val};
        }
    } else {
        warn(
            "Invalid value for option \"props\": expected an Array or an Object, " +
            "but got " + (toRawType(props)) + ".",
            vm
        );
    }
    options.props = res;
}

```
normalizeProps 这个函数的作用的: 将 props 规范到对象. 在这个函数中主要做了以下几件事
- 没有传递 props, 就直接返回
- 对 props 是数组处理
- props 是原生的对象处理
- 如果传递的既不是数据也不是对象 发错报错信息

1. props 是数组
```javascript
/*要最终处理完返回的对象*/
var res = {};
var i, val, name;
/* 检测是否是一个数组 */
if (Array.isArray(props)) {
    /*获取 props 长度*/
    i = props.length;
    while (i--) {
        val = props[i];
        /*在props是数组的情况下, 判断每个值是否是字符串 */
        if (typeof val === 'string') {
            /*将中横线 -  变为驼峰字符*/
            name = camelize(val);
            /* 将该项转换为对象, type为 null */
            res[name] = {type: null};
        } else {
            /*在数组的情况下只能是字符串*/
            warn('props must be strings when using array syntax.');
        }
    }
}
```
在 props 是数组的情况下,数组里面的每一项只能是字符串.
 最终将数据数组里面的每一项字符串转为对象.
 例如:
 ```javascript
 props:["message"]
 ```
 转换后:
 ```javascript
 props:{
    messsage:{
        type:null  /*可以看做是任意类型*/
    }
 }
 ```

2. props 是对象:

```javascript
if (isPlainObject(props)) {
    for (var key in props) {
        val = props[key]; /*获取到key对应的val*/
         /*将中横线 - 转为了驼峰*/
        name = camelize(key);
        /*检测是val是否是原生的对象, 是, 就直接把val赋值给res[name]*/
        res[name] = isPlainObject(val)
            ? val
            : {type: val};
    }
```
最终将 props 统一为对象

camelize 函数的作用把带有中横线 props 命名转为驼峰驼峰命名
```javascript
var camelizeRE = /-(\w)/g;
var camelize = cached(function (str) {
    return str.replace(camelizeRE, function (_, c) {
        return c ? c.toUpperCase() : '';
    })
});
```
看下官网的示例代码
```javascript
Vue.component('blog-post', {
    // 在 JavaScript 中是 camelCase 的
    props: ['postTitle'],
    template: '<h3>{{ postTitle }}</h3>'
})
```
```html
<!-- 在 HTML 中是 kebab-case 的 -->
<blog-post post-title="hello!"></blog-post>
```
在 HTML 中可以 post-title 可以带中横线(推荐这种写法, 符合 HTML 属性规范), 在props规范化中,
将中横线-替换掉, 变成驼峰的方式
####  directives 规范化
在我们定义指令的时候, 可以有不同的写法
```javascript
let  p = Vue.extend({
    directive:{
        test1:{  /*钩子函数*/
            bind:function () {
                console.log("v-test1")
            }
        },
        test2:function () {  /*简写*/
            console.log("v-test1")
        }
    }

})
```
在 mergeOptions 中,会调用 normalizeDirectives(child) 将不同的写法进行统一的规范化

接下来在在源码中是如何规范的
```javascript
/**
 * Normalize raw function directives into object format.
 */
function normalizeDirectives(options) {
    var dirs = options.directives;
    if (dirs) {
        for (var key in dirs) {
            var def$$1 = dirs[key];
            if (typeof def$$1 === 'function') {
                dirs[key] = {bind: def$$1, update: def$$1};
            }
        }
    }
}
```
从上面可以看出, 指令的简写方式, 最终会转为对象形式存在.作为 bind 和 update 的函数

## props methods inject computed合并策略

源码的实现:
```javascript
strats.props =
    strats.methods =
        strats.inject =
            strats.computed = function (
                parentVal,
                childVal,
                vm,
                key
            ) {
                if (childVal && "development" !== 'production') {
                    assertObjectType(key, childVal, vm);
                }
                /*检测parentVal是否存在, 不存在,直接把childVal返回, 不用合并*/
                if (!parentVal) {
                    return childVal
                }
                var ret = Object.create(null);
                /*把parentVal的属相混入ret中*/
                extend(ret, parentVal);
                /*如果childVal存在, 把childVal混入到ret中*/
                if (childVal) {
                    extend(ret, childVal);
                }
                return ret
            };
```
可以看出上面的代码:先把parentVal混入到ret中, 在把childVal混入到ret中.
如果键值一样, 就直接覆盖