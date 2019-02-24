## initProxy-渲染函数的作用域代理

在 vm._init() 函数的初始化过程中调用了 initProxy 函数,以下是 initProxy 函数的源码
```javascript
initProxy = function initProxy(vm) {
/*hasProxy 判断当前环境是否支持es 提供的 Proxy*/
if (hasProxy) {
   // determine which proxy handler to use
   var options = vm.$options;
   /*不同条件返回不同的handlers, getHandler或者hasHandler */
   var handlers = options.render && options.render._withStripped
       ? getHandler
       : hasHandler;
   /* 代理vm实例 */
   vm._renderProxy = new Proxy(vm, handlers);
} else {
   vm._renderProxy = vm;
    }
};
```
initProxy函数的主要目的: 通过vm._renderProxy 代理 vm 实例, 根据不同的条件, 生成 Proxy 的 handlers 拦截行为
vm._renderProxy 在 render 函数被调用的时候, 当做参数被传递进入

```javascript
vnode = render.call(vm._renderProxy, vm.$createElement)
```
render使用call调用, 上下文为vm._renderProxy, 这部分会在后面的小节讲到

先看看 hasProxy 的实现:
```javascript
/*hasProxy的实现*/
var hasProxy = typeof Proxy !== 'undefined' && isNative(Proxy);

/* isNative的实现 */
function isNative(Ctor) {
    return typeof Ctor === 'function' && /native code/.test(Ctor.toString())
}
```
hasProxy 主要的作用就是检测 es6 的 Proxy 是否支持. 如果支持 Proxy , 通过 isNative 检测是否是
语言提供的 Proxy.  这样检测的目的是:我们自己写可以定义一个变量 Proxy, 从而绕开自己定义的

通过 isNative 函数,  可以学习到如何检测一个函数是自己定义的还是语言提供的

接下来分析后面的代码, 在语言支持es6支持 Proxy 的情况下:
```javascript
if (hasProxy) {
    // determine which proxy handler to use
    var options = vm.$options;
    var handlers = options.render && options.render._withStripped
       ? getHandler
       : hasHandler;
    vm._renderProxy = new Proxy(vm, handlers);
}
```
options.render && options.render._withStripped 从这两项是否配置, 来获取 Proxy 的拦截行为
handlers.
- options.render 配置渲染函数
- options.render._withStripped 是在测试环境下会配置此项,结合webpack才会出现

options.render && options.render._withStripped 都支持会返回 getHandler, 否则返回hasHandler

###### hasHandler

```javascript
var hasHandler = {
    /*target要代理的对象, key在外部操作时访问的属性*/
    has: function has(target, key) {
        /*key in target返回true或者false*/
        var has = key in target;
        /*在模板引擎里面,有一些属性vm没有进行代理, 但是也能使用, 像Number,Object等*/
        var isAllowed = allowedGlobals(key) ||
            (typeof key === 'string' && key.charAt(0) === '_' && !(key in target.$data));
        /*在上面的has和isAllowed为false的情况下*/
        if (!has && !isAllowed) {
            if (key in target.$data) {
                warnReservedPrefix(target, key);
            }
            /*warnNonPresent函数, 当访问属性,没有存在vm实例上, 会报错提示*/
            else {
                warnNonPresent(target, key);
            }
        }
        /*has或者isAllowed*/
        return has || !isAllowed
    }
};
```
hasHandler 只配置了 has 钩子 ,当进行propKey in proxy  in 操作符 或者 with() 操作时, 会触发 has钩子函数

hasHandler在查找key时,从三个方向进行查找
-  代理的 target 对象  通过 in 操作符
-  全局对象API      allowedGlobals 函数
-  查找是否是渲染函数的内置方法  第一个字符以_开始  typeof key === 'string' && key.charAt(0) === '_'


hasHandler, 首先去检测 vm 实例上是否有该属性,  下面的代码是vm实例上可以查看到test
```javascript
new Vue({
   el:"#app",
   template:"<div>{{test}}</div>",
   data:{
       test
   }
})
```
如果在 vm 实例上没有找到, 然后再去判断下是否是一些全局的对象, 例如 Number 等, Number是语言所提供的
在模板中也可以使用
```javascript
new Vue({
   el:"#app",
   /*Number属于语言提供的全局API*/
   template:"<div> {{ Number(test) +1 }}</div>",
   data:{
       test
   }
})
```
在模板里使用一些全局的API, 而全局API是 allowedGlobals(key) 函数帮我们收集的
```javascript
var allowedGlobals = makeMap(
    'Infinity,undefined,NaN,isFinite,isNaN,' +
    'parseFloat,parseInt,decodeURI,decodeURIComponent,encodeURI,encodeURIComponent,' +
    'Math,Number,Date,Array,Object,Boolean,String,RegExp,Map,Set,JSON,Intl,' +
    'require' // for Webpack/Browserify
);

/*makeMap函数, str参数是接受的字符串, expectsLowerCase参数是否需要小写*/
 function makeMap(str, expectsLowerCase ) {
        /* 创建一个对象 */
        var map = Object.create(null);
        /*将字符串分割成数组*/
        var list = str.split(',');
        /*对数组进行遍历*/
        for (var i = 0; i < list.length; i++) {
            /*将每个key对应的值设置为true*/
            map[list[i]] = true;
        }
        /*最终返回, 根据参数设置是否是需要转换大小写*/
        return expectsLowerCase
            ? function (val) {
                return map[val.toLowerCase()];
            }
            : function (val) {
                return map[val];
            }
    }
```
makeMap 函数的只要作用把这些全局的API转成以下的形式,
```javascript
{
    Infinity:true,
    undefined:true
}
```
###### getHandler
options.render && options.render._withStripped 都支持会返回 getHandler

当key在target时, 会触发get钩子函数
```javascript
var getHandler = {
    get: function get(target, key) {
        /*key是字符串, 并且key 不在target上*/
        if (typeof key === 'string' && !(key in target)) {
            if (key in target.$data) {
                warnReservedPrefix(target, key);
            }
            else {
                /*key没有在实例上定义*/
                warnNonPresent(target, key);
            }
        }
        /* 直接返回 key 对应的 value */
        return target[key]
    }
};
```
warnNonPresent函数:
```javascript
const warnNonPresent = (target, key) => {
    warn(
        `Property or method "${key}" is not defined on the instance but ` +
        'referenced during render. Make sure that this property is reactive, ' +
        'either in the data option, or for class-based components, by ' +
        'initializing the property. ' +
        'See: https://vuejs.org/v2/guide/reactivity.html#Declaring-Reactive-Properties.',
        target
    )
}
```
warnNonPresent会发出警告信息, 当使用的key 没有在vm实例没有定义.
```javascript
new Vue({
   el:"#app",
   template:"<div> {{ a }}</div>",
   data:{
       message:"hello vue"
   }
})
```
使用了 a  在data没有定义,会触发 warnNonPresent 函数, 打印报错信息