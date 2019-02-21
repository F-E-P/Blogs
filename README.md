
###  Vue2.5源码解析

 ![](/images/vue/vue_base.jpg)
 来自官网的一张图,整体流程:
 - new Vue()时, init初始化过程
 - $mount的挂载
 - compile()编译到render渲染函数
 - 最复杂的响应式系统的搭建

以下的每个小节会从new Vue()开始, 逐步分析每个过程
1.  [new Vue()发生了什么](/article/vue/newVue()发生了什么.md)
2.  [mergeOptions选项合并的策略](/article/vue/newVue()发生了什么.md)
3.  [vm.$mount 深入理解Vue的渲染流程](/article/vue/mount.md)
4.  [template模板是如何编译成render渲染函数](/article/vue/mount.md)

