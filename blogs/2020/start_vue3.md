---
title: vue3快速上手-起始和新特性解读
date: '2020-10-19 13:26:16'
sidebar: false
categories:
 - 技术
tags:
 - vue3
publish: true
---

>  转自 [村长学前端](https://mp.weixin.qq.com/s/T_VSqE_aJ04w5wdU8zbG_g)

本文是vue3快速入门系列文章第一篇，主要讲解如何快速体验vue3，并且了解vue3中的新特性使用，包括：

- Composition API
- Teleport
- Fragments
- Emits Component Option
- `createRenderer` API用于创建自定义渲染器



### composition api

composition api为vue应用提供更好的逻辑复用和代码组织。

```
<template>
  <div>
    <p>counter: {{ counter }}</p>
    <p>doubleCounter: {{ doubleCounter }}</p>
    <p ref="desc"></p>
  </div>
</template>

<script>
import {
  reactive,
  computed,
  watch,
  ref,
  toRefs,
  onMounted,
  onUnmounted,
} from "vue";

export default {
  setup() {
    const data = reactive({
      counter: 1,
      doubleCounter: computed(() => data.counter * 2),
    });

    let timer;

    onMounted(() => {
      timer = setInterval(() => {
        data.counter++;
      }, 1000);
    });

    onUnmounted(() => {
      clearInterval(timer);
    });

    const desc = ref(null);

    watch(
      () => data.counter,
      (val, oldVal) => {
        // console.log(`counter change from ${oldVal} to ${val}`);
        desc.value.textContent = `counter change from ${oldVal} to ${val}`;
      }
    );

    return { ...toRefs(data), desc };
  },
};
</script>
```



### Teleport

传送门组件提供一种简洁的方式可以指定它里面内容的父元素。

```
<template>
  <button @click="modalOpen = true">弹出一个全屏模态窗口</button>

  <teleport to="body">
    <div v-if="modalOpen" class="modal">
      <div>
        这是一个模态窗口! 我的父元素是"body"！
        <button @click="modalOpen = false">Close</button>
      </div>
    </div>
  </teleport>
</template>

<script>
export default {
  data() {
    return {
      modalOpen: true,
    };
  },
};
</script>

<style scoped>
.modal {
  position: absolute;
  top: 0;
  right: 0;
  bottom: 0;
  left: 0;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}

.modal div {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  background-color: white;
  width: 300px;
  height: 300px;
  padding: 5px;
}
</style>

```

###  

### Fragments

vue3中组件可以拥有多个根。

```
<template>
  <div @click="$emit('click')">
    <h3>自定义事件</h3>
  </div>
</template>

<script>
export default {
  emits: ["click"],
};
</script>

```



### Emits Component Option

vue3中组件发送的自定义事件需要定义在emits选项中：

- 原生事件会触发两次，比如`click`
- 更好的指示组件工作方式
- 对象形式事件校验

```
<template>
  <div @click="$emit('click')">
    <h3>自定义事件</h3>
  </div>
</template>

<script>
export default {
  emits: ["click"],
};
</script>

```



### 自定义渲染器 custom renderer

Vue3.0中支持 *自定义渲染器* (Renderer)：这个 API 可以用来自定义渲染逻辑。比如下面的案例我们可以把数据渲染到canvas上。



首先创建一个组件描述要渲染的数据，我们想要渲染一个叫做piechart的组件，我们不需要单独声明该组件，因为我们只是想把它携带的数据绘制到canvas上。创建CanvasApp.vue

```
<template>
  <piechart
    @click="handleClick"
    :data="state.data"
    :x="200"
    :y="200"
    :r="200"
  ></piechart>
</template>
<script>
import { reactive, ref } from "vue";
export default {
  setup() {
    const state = reactive({
      data: [
        { name: "大专", count: 200, color: "brown" },
        { name: "本科", count: 300, color: "yellow" },
        { name: "硕士", count: 100, color: "pink" },
        { name: "博士", count: 50, color: "skyblue" },
      ],
    });
    function handleClick() {
      state.data.push({ name: "其他", count: 30, color: "orange" });
    }
    return {
      state,
      handleClick,
    };
  },
};
</script>

```



下面我们创建自定义渲染器，main.js

```
<script>
import { createApp, createRenderer } from 'vue'
import CanvasApp from './CanvasApp.vue'

const nodeOps = {
 insert: (child, parent, anchor) => {
   // 我们重写了insert逻辑，因为在我们canvasApp中不存在实际dom插入操作
   // 这里面只需要将元素之间的父子关系保存一下即可
   child.parent = parent;

   if (!parent.childs) {
     parent.childs = [child]
  } else {
     parent.childs.push(child);
  }

   // 只有canvas有nodeType，这里就是开始绘制内容到canvas
   if (parent.nodeType == 1) {
     draw(child);
     // 如果子元素上附加了事件，我们给canvas添加监听器
     if (child.onClick) {
       ctx.canvas.addEventListener('click', () => {
         child.onClick();
         setTimeout(() => {
           draw(child)
        }, 0);
      })
    }
  }
},
 remove: child => {},
 createElement: (tag, isSVG, is) => {
   // 创建元素时由于没有需要创建的dom元素，只需返回当前元素数据对象
   return {tag}
},
 createText: text => {},
 createComment: text => {},
 setText: (node, text) => {},
 setElementText: (el, text) => {},
 parentNode: node => {},
 nextSibling: node => {},
 querySelector: selector => {},
 setScopeId(el, id) {},
 cloneNode(el) {},
 insertStaticContent(content, parent, anchor, isSVG) {},
 patchProp(el, key, prevValue, nextValue) {
   el[key] = nextValue;
},
};

// 创建一个渲染器
let renderer = createRenderer(nodeOps);

// 保存画布和其上下文
let ctx;
let canvas;

// 扩展mount，首先创建一个画布元素
function createCanvasApp(App) {
 const app = renderer.createApp(App);
 const mount = app.mount
 app.mount = mount(selector) {
   canvas = document.createElement('canvas');
   canvas.width = window.innerWidth;
   canvas.height = window.innerHeight;
   document.querySelector(selector).appendChild(canvas);
   ctx = canvas.getContext('2d');
   mount(canvas);
}
 return app
}

createCanvasApp(CanvasApp).mount('#demo')
</script>

```



编写绘制逻辑

```
const draw = (el,noClear) => {
 if (!noClear) {
   ctx.clearRect(0, 0, canvas.width, canvas.height)
}
 if (el.tag == 'piechart') {
   let { data, r, x, y } = el;
   let total = data.reduce((memo, current) => memo + current.count, 0);
   let start = 0,
       end = 0;
   data.forEach(item => {
     end += item.count / total * 360;
     drawPieChart(start, end, item.color, x, y, r);
     drawPieChartText(item.name, (start + end) / 2, x, y, r);
     start = end;
  });
}
 el.childs && el.childs.forEach(child => draw(child,true));
}

const d2a = (n) => {
 return n * Math.PI / 180;
}
const drawPieChart = (start, end, color, cx, cy, r) => {
 let x = cx + Math.cos(d2a(start)) * r;
 let y = cy + Math.sin(d2a(start)) * r;
 ctx.beginPath();
 ctx.moveTo(cx, cy);
 ctx.lineTo(x, y);
 ctx.arc(cx, cy, r, d2a(start), d2a(end), false);
 ctx.fillStyle = color;
 ctx.fill();
 ctx.stroke();
 ctx.closePath();
}
const drawPieChartText = (val, position, cx, cy, r) => {
 ctx.beginPath();
 let x = cx + Math.cos(d2a(position)) * r/1.25 - 20;
 let y = cy + Math.sin(d2a(position)) * r/1.25;
 ctx.fillStyle = '#000';
 ctx.font = '20px 微软雅黑';
 ctx.fillText(val,x,y);
 ctx.closePath();
}
```


