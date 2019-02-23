## Vue技术栈系列(持续更新中...)
###  Vue.js 2.5源码解析
 ![](/images/vue/vue_base.jpg)
 来自官网的一张图,整体流程:
 - new Vue()时, init初始化过程
 - compile()编译到render渲染函数
 - 响应式系统

以下的每个小节会从new Vue()开始, 逐步分析每个过程
1.  [Vue.js初始化过程 - new Vue()内部运行机制](/article/vue/newVue()发生了什么.md)
2.  [Vue.js初始化过程 - mergeOptions选项合并策略](/article/vue/mergeOptions.md)
3.  [Vue.js初始化过程 - 选型data的合并策略](/article/vue/选项data的合并策略.md)
4.  [Vue.js模版编译 - vm.$mount 深入理解Vue的渲染流程](/article/vue/mount.md)
5.  [Vue.js模板编译 - template模板是如何编译成render渲染函数流程分析](/article/vue/compileToFunctions.md)

###  Vuex源码解析
###  Vue-router源码解析
