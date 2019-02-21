#  Vue. new Vue()发生了什么

## new Vue()发生了什么
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
- 对Vue构造函数的进行限制, 只有通过new Vue()才可调用,
而直接调用Vue(),里面的this会指向window会有报错提醒
- 调用了自身原生的的_init函数

下面是Vue原型上的_init函数的代码
```JavaScript
var uid$3 = 0; /* 用于统计Vue构造函数被new多少次*/
/*initMixin主要是对对options选型的规范和合并*/
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
            vm.$options = mergeOptions(
                resolveConstructorOptions(vm.constructor),
                options || {},
                vm
            );
        }
        /* istanbul ignore else */
        {
            initProxy(vm); /*初始化Proxy, 检测Proxy函数是否支持等*/
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

        if (vm.$options.el) {
            vm.$mount(vm.$options.el);
        }
    };
}
```
_init函数的流程:
- 最重要的事情mergeOptions()函数, 进行选项的合并
- 初始化页面, 初始化事件, 渲染页面, 初始化 data、props、computed、watcher 等等
