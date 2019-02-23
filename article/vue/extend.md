## Vue.extend的实现原理

Vue.extend(): 使用基础 Vue 构造器，创建一个“子类”。
来自官网的一段示例代码
```html
<div id="mount-point"></div>
```
```javascript
// 创建构造器
var Profile = Vue.extend({
  template: '<p>{{firstName}} {{lastName}} aka {{alias}}</p>',
  data: function () {
    return {
      firstName: 'Walter',
      lastName: 'White',
      alias: 'Heisenberg'
    }
  }
})
/* 创建 Profile 实例，并挂载到一个元素上 */
new Profile().$mount('#mount-point')
```
结果
```html
<p>Walter White aka Heisenberg</p>
```
通过下面的图,展示Vue.extend()创建的子类的关系图
![](/images/vue/2.vue.jpg)

接下来更加查看Vue.extend()源码的实现原理:
```javascript
function initExtend(Vue) {
    ...
    Vue.extend = function (extendOptions) {
        extendOptions = extendOptions || {};
        var Super = this; /*Super代表vm的实例*/
        var SuperId = Super.cid;
        var cachedCtors = extendOptions._Ctor || (extendOptions._Ctor = {});
        if (cachedCtors[SuperId]) {
            return cachedCtors[SuperId]
        }

        var name = extendOptions.name || Super.options.name;
        if (name) {
            validateComponentName(name);
        }
        /*Sub 定义一个子类的构造函数*/
        var Sub = function VueComponent(options) {
            /*调用vm原型的上的this._init函数*/
            this._init(options);
        };
        /*把子类的原型对象Sub.prototype执行Super.prototype对象 */
        Sub.prototype = Object.create(Super.prototype);
        /*修改constructor的指向*/
        Sub.prototype.constructor = Sub;
        Sub.cid = cid++;
        Sub.options = mergeOptions(
            Super.options,
            extendOptions
        );
        Sub['super'] = Super;

        // For props and computed properties, we define the proxy getters on
        // the Vue instances at extension time, on the extended prototype. This
        // avoids Object.defineProperty calls for each instance created.
        if (Sub.options.props) {
            initProps$1(Sub);
        }
        if (Sub.options.computed) {
            initComputed$1(Sub);
        }

        // allow further extension/mixin/plugin usage
        Sub.extend = Super.extend;
        Sub.mixin = Super.mixin;
        Sub.use = Super.use;

        // create asset registers, so extended classes
        // can have their private assets too.
        ASSET_TYPES.forEach(function (type) {
            Sub[type] = Super[type];
        });
        // enable recursive self-lookup
        if (name) {
            Sub.options.components[name] = Sub;
        }

        // keep a reference to the super options at extension time.
        // later at instantiation we can check if Super's options have
        // been updated.
        Sub.superOptions = Super.options;
        Sub.extendOptions = extendOptions;
        Sub.sealedOptions = extend({}, Sub.options);

        // cache constructor
        cachedCtors[SuperId] = Sub;

        return Sub
    };
}
```