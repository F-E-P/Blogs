# vm.$mount 深入理解Vue的渲染流程

$mount的实现跟构建方式(webpack的vue-loader)有关系,也和平台有关系(weex),
下面的代码只分析了纯前端浏览器环境下的$mount的实现

```javascript
    new Vue({
        el:"#app",
        data:{}
    })
    //手动实现挂载
    new Vue({
        data:{}
    }).$mount("#app")
    以上两种方式会触发$mount函数
```
以下是Vue.$mount函数独立构建时的渲染的流程图:

![](/images/vue/$mount.jpg)

以代码的形式来来展示:

模板字符串 => render函数 => 虚拟dom => 真实的dom节点

![](/images/vue/$mount1.jpg)

开始分析 $mount, 以下是$mount的源码, 省略了部分代码:
```javascript
/*缓存不带编译功能的$mount函数, 在下面$mount最后调用*/
/*进行缓存Vue.prototype.$mount*/
var mount = Vue.prototype.$mount;
Vue.prototype.$mount = function (el, hydrating) {
    /* $el元素的查找过程 */
    el = el && query(el);
    if (el === document.body || el === document.documentElement) {
        warn("Do not mount Vue to <html> or <body> - mount to normal elements instead.");
        return this
    }
    /*  template的确定过程 */
    var options = this.$options;
    // resolve template/el and convert to render function
    /*检测vm.$options 是否有render函数, 如果有优先选用render函数, 不在进行template模板的编译*/
    /*没有配置render函数,在vm.$options确定是否配置tempalte模板*/
    if (!options.render) {
        var template = options.template;
            if (template) {
                if (typeof template === 'string') {
                   ...
                } else if (template.nodeType) {
                    template = template.innerHTML;
                } else {
                    { warn('invalid template option:' + template, this);}
                    return this
                }
            } else if (el) {
                template = getOuterHTML(el);
            }
            if (template) {
               ...
                var ref = compileToFunctions(template, {
                    outputSourceRange: "development" !== 'production',
                    shouldDecodeNewlines: shouldDecodeNewlines,
                    shouldDecodeNewlinesForHref: shouldDecodeNewlinesForHref,
                    delimiters: options.delimiters,
                    comments: options.comments
                }, this);
                var render = ref.render;
                var staticRenderFns = ref.staticRenderFns;
                options.render = render;
                options.staticRenderFns = staticRenderFns;

                /* istanbul ignore if */
                if (config.performance && mark) {
                    mark('compile end');
                    measure(("vue " + (this._name) + " compile"), 'compile', 'compile end');
                }
            }
        }
        return mount.call(this, el, hydrating)
    };
```
### 确定$el元素, 寻找挂载点
```javascript
    el = el && query(el);
    if (el === document.body || el === document.documentElement) {
        warn("Do not mount Vue to <html> or <body> - mount to normal elements instead.");
        return this
    }
```
调用了query(el)
```javascript
function query(el) {
    if (typeof el === 'string') {
        var selected = document.querySelector(el);
        if (!selected) {
            warn('Cannot find element: ' + el);
            return document.createElement('div')
        }
        return selected
    } else {
        return el
    }
}
```
query函数的主要作用是:
- 使用原生的querySelector(查找第一个匹配的dom元素)来查找dom，如果没有找到则新建一个div返回；
- 若el自身就是一个dom元素，则直接返回。

可以看出对el进行了很多的限制, 不能挂在到html上和body上.

### 检测template 是否配置
```javascript
var mount = Vue.prototype.$mount;
Vue.prototype.$mount = function (el, hydrating) {
    ...
    var options = this.$options;
    /*  没有配置vm.$options.render函数  */
    if (!options.render) {
        var template = options.template;
         /*template模板是否存在*/
        if (template) {
            if (typeof template === 'string') {
                /* 在template里面也可以这样配置 */
                /* template:"#app"  如果这样配置, 判断第一个字符是#, 过选择器查询dom元素*/
                if (template.charAt(0) === '#') {
                    template = idToTemplate(template);
                    if (!template) {
                        warn(
                            ("Template element not found or is empty: " + (options.template)),
                            this
                        );
                    }
                }
            /*直接配置了dom节点对象*/
            } else if (template.nodeType) {
                template = template.innerHTML;
            } else {
                {
                    warn('invalid template option:' + template, this);
                }
                return this
            }
         /* 没有设置vm.$options.template, 就通过el元素来获取 */
        } else if (el) {
            template = getOuterHTML(el);
        }
        ...
    }



    /* idToTemplate 函数的实现, 通过id来获取el的innerHTML*/
    var idToTemplate = cached(function (id) {
        /*根据id获取dom*/
        var el = query(id)
        return el && el.innerHTML
    });
    /* getOuterHTML函数 */
    function getOuterHTML(el) {
        if (el.outerHTML) {
            return el.outerHTML
        } else {
            /*一般情况下,svg就没有outerHTML*/
            /* 创建一个div标签把自己包裹起来 */
            var container = document.createElement('div');
            container.appendChild(el.cloneNode(true)); //cloneNode 传入true, 把子节点也拷贝了
            return container.innerHTML
        }
    }
```
首先会检测vm.$options.render函数是否进行配置
- 有配置render函数, 直接调用去render(从Vue.js 2.0开始优选使用render函数)
- 没有配置render函数, 会把el或者template转换成render函数, 通过 compileToFunctions函数

可以看出最终Vue组件的渲染都需要使用render函数来渲染,
如果有render函数, 就不在进行template模板的编译.

## 运行时 + 编译器 vs. 只包含运行时
上面的分析, 可以看懂官网上的 "运行时 + 编译器 vs. 只包含运行时"的区别
![](/images/vue/Vue.运行时.jpg)
