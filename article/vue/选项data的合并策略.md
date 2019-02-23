## 选项data的合并策略
在vue.js源码中, 选项 el,data,watch,props 等都有合并策略

在这个一节中, 只分析 data 的合并策略
```javascript
   /*对parent, child的key字段进行和合并,采取了不同的策略*/
     function mergeField(key) {
         var strat = strats[key] || defaultStrat;
         /*把 strat(parent[key], child[key], vm, key)函数的返回值给对应的options[key]*/
         options[key] = strat(parent[key], child[key], vm, key);
     }
```
在上一节中, strat(parent[key], child[key], vm, key) 函数的返回值, 赋值给 options[key]

可以先记住一个结论:
 - 无论是 vm 根实例, 还是子组件中 选项里的 data 最终都是函数.
 - 那么 data 是一个函数, 何时被的调用:加入到响应式系统或者数据初始化的时候,会被调用

接下来,分析选项 data 的合并过程, 下面是 strats.data 的自定义策略
```javascript
    var strats = config.optionMergeStrategies;
    strats.data = function ( parentVal, childVal, vm ) {
            /*检测vm,区分vm根实例,还是子类(组件)*/
            if (!vm) {
                /*处理组件的信息,vm的子类,选项的data就必须是function*/
                if (childVal && typeof childVal !== 'function') {
                    warn( /*发出报错的信息*/
                        'The "data" option should be a function ' +
                        'that returns a per-instance value in component ' +
                        'definitions.',
                        vm
                    );
                    return parentVal
                }
                /*运行到这里:传递vm, 说明是子类,组件*/
                return mergeDataOrFn(parentVal, childVal)
            }
            /*运行到这里:传递vm, 说明是实例*/
            return mergeDataOrFn(parentVal, childVal, vm)
        };
```
在整个Vue源码选项合并过程中, 区分子类, 还是根实例, 通过判断参数是否传递了 vm.
- 在子类中,子类 data 不是一个函数, 发生报错的信息,返回父组件的 parentVal, 如果是一个函数
直接调用 mergeDataOrFn 函数, 进行数据合并,不传入 vm
- vm 是根实例, 调用 mergeDataOrFn 函数, 并传入 vm

mergeDataOrFn函数的源码如下
```javascript
function mergeDataOrFn( parentVal, childVal,vm) {
        if (!vm) { /*子类*/
            // in a Vue.extend merge, both should be functions
            /* 通过 Vue.extend的子类,  父子的data都应该是一个函数*/
            if (!childVal) {
                return parentVal
            }
            if (!parentVal) {
                return childVal
            }
            /*childVal, parentVal都没有传递, 运行到此结束 */
            // when parentVal & childVal are both present,
            // we need to return a function that returns the
            // merged result of both functions... no need to
            // check if parentVal is a function here because
            // it has to be a function to pass previous merges.
            /*能够运行到这里, childVal,parentVal都会有值,才进行合并*/
            return function mergedDataFn() {
                /*将childVal,parentVal通过call来的调用, 把返回值纯对象当做参数传递给mergeData()函数*/
                return mergeData(
                    typeof childVal === 'function' ? childVal.call(this, this) : childVal,
                    typeof parentVal === 'function' ? parentVal.call(this, this) : parentVal
                )
            }
        } else { /*vm根实例*/
            return function mergedInstanceDataFn() {
                // instance merge
                var instanceData = typeof childVal === 'function'
                    ? childVal.call(vm, vm)
                    : childVal;
                var defaultData = typeof parentVal === 'function'
                    ? parentVal.call(vm, vm)
                    : parentVal;
                if (instanceData) {
                    return mergeData(instanceData, defaultData)
                } else {
                    return defaultData
                }
            }
        }
    }

```
在 mergeDataOrFn 代码中, 有这样的一段代码
```javascript
if (!childVal) {
    return parentVal
}
if (!parentVal) {
    return childVal
}
```
- 如果 childVal 没有值, 那么就直接返回 parentVal.
- 如果 childVal 有值, 而 parentVal 没有值, 就返回 childVal
- 两者都没有值, 下面的代码根本就不会执行. 两者都有值的情况下才进行下面的代码

在 childVal,parentVal 都有值的的情况下, 最终把 mergedDataFn() 函数返回, 没有进行合并

##### 无论子组件还是根实例, 最终都调用mergeData进行终极合并, 瞬间产生了两个问题
1.  为什么会返回一个 mergeData 函数?

    通过函数返回一个对象, 对象属于引用类型, 在内存不是同一份, 保证了每个组件的数据的独立性,
    可以避免组件之间的相互影响


2.  mergeData 为什么不在初始化的时候就合并好, 而是在调用的时候进行合并?

    inject 和 props 这两个选项的初始化是先于 data 选项的，就保证了能够使用 props 初始化 data 中的数据.

接下来, 在看看最终的mergeData终极合并的函数
```javascript
/* 此时的to, form是纯对象*/
function mergeData(to, from) {
    /* form 没有值, 直接返回to*/
    if (!from) {
        return to
    }
    var key, toVal, fromVal;
    /*获取from的key的集合*/
    var keys = hasSymbol
        ? Reflect.ownKeys(from)
        : Object.keys(from);
    /*对form的keys进行遍历*/
    for (var i = 0; i < keys.length; i++) {
        key = keys[i];
        // in case the object is already observed...
        /*将入到响应式系统的对象都将有__ob__属性*/
        if (key === '__ob__') {
            continue
        }
        toVal = to[key];
        fromVal = from[key];
        /*from 对象中的 key 不存在 to 对象中，则使用set函数为to对象设置 key 及相应的值*/
        if (!hasOwn(to, key)) {
            set(to, key, fromVal);
         /* from对象中的key也在to对象中，且这两个属性的值都是纯对象则递归进行深度合并*/
        } else if (toVal !== fromVal && isPlainObject(toVal) &&
                    isPlainObject(fromVal)) {
            mergeData(toVal, fromVal);
        }
    }
    return to
}
```
终结合并只处理了两种情况
- from 对象中的 key 不存在 to 对象中，则使用 set 函数为 to 对象设置 对应的 key 及对应的的值
- from对象中的 key 也在 to 对象中，且这两个属性的值都是纯对象则递归进行深度合并







