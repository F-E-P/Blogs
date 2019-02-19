##  vm.$mount 深入理解Vue的渲染流程

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
    以上两种方式会触发$mount的调用
```
以下是Vue.$mount函数独立构建时的渲染的流程图, 将会分析每一步的代码实现过程

![](/images/vue/$mount.jpg)

以代码的形式来来展示

模板字符串 => render函数 => 虚拟dom => 真实的dom节点

![](/images/vue/$mount1.jpg)

开始分析 $mount, 以下是$mount的源码, 省略了部分代码
```javascript
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
    if (!options.render) {
        var template = options.template;
        ...
        if (template) {
            ...
            /* 把template编译成渲染函数 */
            var ref = compileToFunctions(template, {
                shouldDecodeNewlines: shouldDecodeNewlines,
                shouldDecodeNewlinesForHref: shouldDecodeNewlinesForHref,
                delimiters: options.delimiters,
                comments: options.comments
            }, this);
            ...
            }
        }
    }
    return mount.call(this, el, hydrating)
};
```
### $el查找过程
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
    /*  没有配置render函数  */
    if (!options.render) {
        var template = options.template;
        if (template) {  /*template模板是否存在*/
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
            } else if (template.nodeType) {  /*直接配置了dom节点对象*/
                template = template.innerHTML;
            } else {
                {
                    warn('invalid template option:' + template, this);
                }
                return this
            }
        } else if (el) {  /*没有设置 template, 就通过el元素来获取*/
            template = getOuterHTML(el);
        }
        ...
    }
    /*idToTemplate 函数的实现, 通过id来获取el的innerHTML*/
    var idToTemplate = cached(function (id) {
        /*根据id获取dom*/
        var el = query(id)
        return el && el.innerHTML
    });
    //
    function getOuterHTML(el) {
        if (el.outerHTML) {
            return el.outerHTML
        } else {   /*如果没有就创建一个,svg就没有outerHTML*/
           /* 创建一个div标签把自己包裹起来 */
            var container = document.createElement('div');
            container.appendChild(el.cloneNode(true)); //cloneNode 传入true, 把子节点也拷贝了
            return container.innerHTML
        }
    }
```
首先会检测vm.$options.render函数是否进行配置
- 有配置render函数, 直接调用去render
- 没有配置render函数, 会把el或者template转换成render函数, 通过 compileToFunctions函数

可以看出最终Vue组件的渲染都需要使用render函数来渲染.
