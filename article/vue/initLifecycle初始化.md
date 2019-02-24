## initLifecycle初始化

先看个简单的小例子
```javascript
const compA = {
    template: "<div>我是compA</div>"
}
const vm = new Vue({
    el: "#app",
    components: {
        "comp-a": compA
    }
})
console.log(vm)
```
在控制打印 vm 时, 可以看到这两个属性:$parent, $chilren

![](/images/vue/6.vue.jpg)
在官网看到这个两个的属性的解释
- $parent   父实例，如果当前实例有的话。
- $chilren  当前实例的直接子组件

在对 initLifecycle 函数深入理解之前, 介绍一下抽象组件

![](/images/vue/7.vue.jpg)

根据官网的解释, 他自身不会渲染到Dom元素, 也不会出现在父组件链中,只是起着一层包裹的作用

在 Vue.js 中抽象组件有:
```javascript
/* KeepAlive抽象的组件 */
var KeepAlive = {
    name: 'keep-alive',
    abstract: true,
    props: {
        ...
    },
    created: function created() {
       ...
    },
    ...
    return vnode || (slot && slot[0])
}

/* Transition抽象组件 */
var Transition = {
    name: 'transition',
    props: transitionProps,
    abstract: true,
    render: function render(h) {
       ...
    }
    ...
    return rawChild
}
```
从两个抽象组件中, 可以看出,抽象组件有一个共同的标识

```javascript

abstract: true
```

接下来, 开始了解 initLifecycle 函数的功能:
- 将当前的实例添加到父实例的$children中, 并设置自身的$parent指向父实例
- 初始化一些属性和状态

在 _init 函数过程, 调用了 initLifecycle(vm),
```javascript

vm._self = vm;
initLifecycle(vm);
```
initLifecycle() 函数的具体代码如下:

```javascript

function initLifecycle(vm) {
    /*获取到options, options已经在mergeOptions中最终处理完毕*/
    var options = vm.$options;

    // locate first non-abstract parent
    /*获取当前实例的parent*/
    var parent = options.parent;
    /*parent存在, 并且不是非抽象组件*/
    if (parent && !options.abstract) {
        /*循环向上查找, 知道找到是第一个非抽象的组件的父级组件*/
        while (parent.$options.abstract && parent.$parent) {
            parent = parent.$parent;
        }
        /*将当前的组件加入到父组件的$children里面.  此时parent是非抽象组件 */
        parent.$children.push(vm);
    }
    /*设置当前的组件$parent指向父级组件*/
    vm.$parent = parent;
    vm.$root = parent ? parent.$root : vm;

    /*设置vm的一些属性*/
    vm.$children = [];
    vm.$refs = {};

    vm._watcher = null;
    vm._inactive = null;
    vm._directInactive = false;
    vm._isMounted = false;
    vm._isDestroyed = false;
    vm._isBeingDestroyed = false;
}
```
从上面的 if 开始, 成立的条件是: 当前组件有 parent 属性, 并且是非抽象组件. 才进入 if 语句.
然后通过 while 循环.向上继续查到 第一个非抽象组件. 然后做了两件事:
- 将当前的 vm 添加到查找到的第一个非抽象父级组件 $children 中

```javascript

 parent.$children.push(vm);
```
- 将当前的组件的$parent,指向查找到的第一个非抽象组件

```javascript

vm.$parent = parent;
```

之后的代码给vm设置了一些属性



