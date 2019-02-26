#  Vue.js  new Vue()内部运行机制

![](/images/vue/vue_base.jpg)
 来自官网的一张图,整体流程:
 - new Vue() 时, init 初始化过程
 - compile()  模板编译到 render 渲染函数
 - 响应式系统


Vue.js的 init 过程,是比较复杂的.
经常书写下面的代码:
```JavaScript
  new Vue({
      el:"#app",
      data:{
          test:"这是一个测试"
      }
  })
```
接下来看看Vue的构造函数
```JavaScript
    /*Vue的构造函数,options是我们在new Vue({}) 传递进来的一个对象*/
    function Vue (options) {
        /*进行了Vue函数调用的安全监测*/
        /*Vue的构造函数只能通过new Vue()来调用*/
        /*不能通过 Vue()调用, 这样调用 this指向window*/
        if (!(this instanceof Vue)) {
            /*封装了warn函数,  用于报错提示信息等*/
            warn('Vue is a constructor and should be called with the `new` keyword');
        }
        /* 调用原型上的_init方法, 进行初始化  */
        this._init(options);
    }
```
从上面的代码可以看出:
- 对Vue构造函数的进行限制, 只有通过 new Vue() 才可调用,
而直接调用 Vue(),里面的 this 会指向 window 会有报错提醒
- 调用了自身原型上的 _init 函数

下面是Vue原型上的 _init 函数的代码
```JavaScript
var uid$3 = 0; /* 用于统计Vue构造函数被new多少次*/
/*initMixin主要是对对options选项的合并和规范*/
function initMixin (Vue) {
    Vue.prototype._init = function (options) {
        var vm = this;
        /* 统计 Vue被new了多少次*/
        vm._uid = uid$3++;

        var startTag, endTag;
        /* istanbul ignore if */
        if (config.performance && mark) {
            startTag = "vue-perf-start:" + (vm._uid);
            endTag = "vue-perf-end:" + (vm._uid);
            mark(startTag);
        }


        /*设置了一个标识, 避免被vm实例加入响应式系统 */
        vm._isVue = true;
        /*合并选项*/
        if (options && options._isComponent) {
            // optimize internal component instantiation
            // since dynamic options merging is pretty slow, and none of the
            // internal component options needs special treatment.
            initInternalComponent(vm, options);
        } else {
            /*选项合并的入口代码*/
            vm.$options = mergeOptions(
                resolveConstructorOptions(vm.constructor),
                options || {},
                vm
            );
        }
        /* istanbul ignore else */
        {
            initProxy(vm); /*初始化Proxy, 检测ES6的Proxy函数是否支持等*/
        }
        // expose real self
        vm._self = vm;
        initLifecycle(vm);  /* 初始时生命周期*/
        initEvents(vm); /* 初始化事件*/
        initRender(vm);  /* 渲染页面 */
        callHook(vm, 'beforeCreate');   /*生命周期钩子函数beforeCreate被的调用*/
        initInjections(vm); // resolve injections before data/props
        initState(vm);    /*初始化状态 props data computed watch methods*/
        initProvide(vm); // resolve provide after data/props
        callHook(vm, 'created');   /*生命周期钩子函数created被的调用*/

        /* istanbul ignore if */
        if (config.performance && mark) {
            vm._name = formatComponentName(vm, false);
            mark(endTag);
            measure(("vue " + (vm._name) + " init"), startTag, endTag);
        }
         /*如果配置了el选项, 去挂载*/
        if (vm.$options.el) {
            vm.$mount(vm.$options.el);
        }
    };
}
```
_init 函数的流程:
- 最重要的事情 mergeOptions() 函数, 进行选项的合并和规范化
- 初始化页面, 初始化事件, 渲染页面, 初始化 data、props、computed、watcher 等等
