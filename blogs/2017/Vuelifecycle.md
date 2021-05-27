---
title: vue 生命周期
date: '2021-05-12 12:54:14'
sidebar: false
categories:
 - vue
tags:
 - 生命周期
publish: true
---


![2018_10_29_EzDRiuSYNd.png](../images/2018_10_29_EzDRiuSYNd.png)

*Vuejs 2生命周期挂钩*

- **beforeCreate（）**：这个方法在Vue实例初始化之后，在数据观察和事件/观察器设置之前被同步调用。
- **created（）**：创建Vue实例后，同步调用此方法。数据观察，计算属性，方法和事件回调已经在这个阶段建立起来了，但是尚未开始。
- **beforeMount（）**：这个方法在组件被挂载之前调用。所以在`render`方法执行之前调用它。
- **mounted（）**：这个方法在组件被装载后调用。
- **beforeUpdate（）**：在虚拟DOM被重新渲染和修补之前，当数据改变时，这个方法被调用。
- **updated（）**：在数据更改导致虚拟DOM被重新渲染和修补后调用此方法。
- **activated（）**：**激活**保持活动的组件时调用此方法。
- **deactivated（）**：当一个保持活动的组件被去激活时，这个方法被调用。
- **beforeDestroy（）**：在Vue实例或组件被销毁之前调用此方法。在这个阶段，这个实例仍然是完整的。
- **destroy（）**：在Vue实例或组件被销毁后调用此方法。当这个钩子被调用时，Vue实例的所有指令都被解除绑定，所有的事件监听器都被移除，所有的子Vue实例也被销毁

