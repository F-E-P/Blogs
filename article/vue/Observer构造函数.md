## Observer 构造函数 - 响应对象的变化并深度观测

```javascript
/**
 * Observer class that is attached to each observed
 * object. Once attached, the observer converts the target
 * object's property keys into getter/setters that
 * collect dependencies and dispatch updates.
 */
var Observer = function Observer(value) {
    /* 获取到传入过来的value值 */
    this.value = value;
    /* 依赖收集 */
    this.dep = new Dep();
    /*统计被观察的数据的个数*/
    this.vmCount = 0;
    /* 给value添加 __ob__的数据, 目的是让__ob__变成不可枚举的属性 */
    def(value, '__ob__', this);
    /*value不仅是数组,还可以是对象*/
    if (Array.isArray(value)) {
        if (hasProto) {
            protoAugment(value, arrayMethods);
        } else {
            copyAugment(value, arrayMethods, arrayKeys);
        }
        this.observeArray(value);
    } else {
        this.walk(value);
    }
};
/**
 * Walk through all properties and convert them into
 * getter/setters. This method should only be called when
 * value type is Object.
 */
 /*将对象类型的属性转为getter/setters*/
Observer.prototype.walk = function walk(obj) {
   var keys = Object.keys(obj);
   for (var i = 0; i < keys.length; i++) {
       defineReactive$$1(obj, keys[i]);
   }
};
/**
 * Observe a list of Array items.
 */
Observer.prototype.observeArray = function observeArray(items) {
     for (var i = 0, l = items.length; i < l; i++) {
        observe(items[i]);
     }
};
```
根据 Observer 构造函数的注释, Observer 构造函数的主要作用是把每个要被观察的数据添加到响应式系统, 一旦添加
该数据, 将该数据的属性转为getter/setters, 收集依赖和派发更新

Observer 构造函数的代码结构, 很简单
- value 属性       要将入响应式系统的数据
- dep 属性         依赖收集
- vmCount 属性     统计响应式数据
- walk            原型上的方法
- observeArray    原型上的方法

######  被观察的数据的 \_\_ob\_\_ 属性
在 Observer 构造函数里面, 调用了 def(value, '\_\_ob\_\_', this) 函数
```javascript
/* def(value, '__ob__', this) */
function def(obj, key, val, enumerable) {
    Object.defineProperty(obj, key, {
        value: val,
        enumerable: !!enumerable,
        writable: true,
        configurable: true
    });
}
```
def 函数主要是配置 Object.defineProperty 里的 enumerable.
在 Observer 里面调用了 def,没传递 enumerable, 那么就是undefined, !!enumerable两次取反最终为false.
通过 for in 操作不可枚举出 \_\_ob\_\_ 属性

###### 对象加入到响应式系统
经过 if...else...的判断,  如果是对象, 将调用
```javascript
   this.walk(value);
```
通过实例调用, walk() 是定义在原型上的
```javascript
Observer.prototype.walk = function walk(obj) {
    /* 获取到key的集合 */
   var keys = Object.keys(obj);
   /* 进行遍历 */
   for (var i = 0; i < keys.length; i++) {
        /*defineReactive$$1函数调用多少次, 取决于数据对象的key的个数*/
        /*将数据对象obj, 对应的key当做参数传入, 只传递了两个参数*/
       defineReactive$$1(obj, keys[i]);
   }
};
```
walk 函数通过 Object.keys 收集 obj 的 key,对 keys 里面的每个key进行遍历, 调用了 defineReactive$$1() 函数.

```javascript
function defineReactive$$1( obj, key, val, customSetter, shallow ) {
        /* dep是回调列表, 用来收集依赖 */
        var dep = new Dep();
        /* 获取属性描述对象, 获取用户是否已经添加了描述对象 */
        var property = Object.getOwnPropertyDescriptor(obj, key);
        /* 该属性的描述对象如果存在, 并且是不可以配置, 直接返回了 */
        if (property && property.configurable === false) {
            return
        }

        // cater for pre-defined getter/setters
        /*获取 get 钩子函数*/
        var getter = property && property.get;
        /*获取 set 钩子函数*/
        var setter = property && property.set;
        if ((!getter || setter) && arguments.length === 2) {
            val = obj[key];
        }
        /* 深度的观测 */
        var childOb = !shallow && observe(val);
        Object.defineProperty(obj, key, {
            enumerable: true,
            configurable: true,
            /* 进行依赖的收集 */
            get: function reactiveGetter() {
                ...
                return value
            },
            /* 调用依赖项 */
            set: function reactiveSetter(newVal) {
               ...
            }
        });
    }
```
defineReactive$$1 函数是整个响应式系统的核心, 将数据对象的属性转换为访问器属性, 给属性添加getter/setter


接下里分析 defineReactive$$ 1函数

```javascript
var property = Object.getOwnPropertyDescriptor(obj, key);
if (property && property.configurable === false) {
    return
}
```
Object.getOwnPropertyDescriptor 获取该属性的描述对象, 判断是否有 property, 并且判断该对象是否可以进行配置.
如果不能进行配置, 也就不能为改属性的定义

继续分析
```javascript
// cater for pre-defined getter/setters
/*获取 get 钩子函数*/
var getter = property && property.get;
/*获取 set 钩子函数*/
var setter = property && property.set;
if ((!getter || setter) && arguments.length === 2) {
   val = obj[key];
}
/* 深度的观测  */
var childOb = !shallow && observe(val);
```
用了 && 逻辑运算符,  如果 property 存在, property.get 也存在, getter 是 get 的钩子函数. 反之是 false. setter 也是同样的原理

可以这样想: 一个属性, 已经存在了 get 或者 set 钩子函数. if 的判断保证了响应系统数据的一致性
```javascript
var childOb = !shallow && observe(val);
```
在调用 defineReactive$$1 的时候, 没有传递 shallow 形参, 是undefined, 为false . 进行取反.
此时的 val = obj[key] 的属性的值, 可能还是一个对象, 调用 observe 函数,进行深度观测 进行数据过滤和加入响应式系统

get 钩子函数如下:
```javascript
get: function reactiveGetter() {
    var value = getter ? getter.call(obj) : val;
    if (Dep.target) {
        dep.depend();
        if (childOb) {
            childOb.dep.depend();
            if (Array.isArray(value)) {
                dependArray(value);
            }
        }
    }
    return value
}
```
检测 getter 的钩子函数是否存在, 如果存在就直接调用,  获取返回值.  最终返回
到此开始进行依赖的收集, 进行 if 判断, Dep.target 存在, 将调用 dep.depend() 函数收集依赖.
如果 childOb 存在, 进行深度的依赖收集. 如果是一个数组 在下一节会讲到

set钩子函数
```javascript
 set: function reactiveSetter(newVal) {
        var value = getter ? getter.call(obj) : val;
        /* eslint-disable no-self-compare */
        if (newVal === value || (newVal !== newVal && value !== value)) {
            return
        }
        /* eslint-enable no-self-compare */
        if (customSetter) {
            customSetter();
        }
        // #7981: for accessor properties without setter
        if (getter && !setter) {
            return
        }
        if (setter) {
            setter.call(obj, newVal);
        } else {
            val = newVal;
        }
        childOb = !shallow && observe(newVal);
        dep.notify();
    }
});
```
set 钩子函数在数据发生变化, 进行了依赖的触发.
```javascript
 if (newVal === value || (newVal !== newVal && value !== value)) {
    return
}
```
if 判断进行了性能的优化, 如果改变的值 newVal 和 value 相等, 没有必要进行依赖的触发.
newVal !== newVal 自身与自身不相等, 只有NaN
```javascript
 childOb = !shallow && observe(newVal);
```
如果给当前的依赖进行设置newVal, 是一个引用类型, 继续进行 observe, 进行数据监测, 加入到响应式系统

```javascript
 dep.notify();
```
进行依赖的触发



