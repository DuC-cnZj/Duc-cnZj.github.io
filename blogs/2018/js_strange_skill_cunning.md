---
title:  js 小技巧
date: "2018-10-31 10:37:21"
sidebar: false
categories:
 - 技术
tags:
 - js
publish: true
---


计算数组值的和

```js
let a = [1,2,3,4]
let sum=0;

for (let i = 0; i < a.length; i++) {
    sum += a[i];
}

console.log(sum); // 10


let arr = [1,2,3,4]
let str = arr.join('+')
eval(str) // 10
```

双重遍历去重

```js
arr = [1, 2, 3, 3, 4, 4, 5, 2, 1, 4];

for (let i = 0; i < arr.length - 1; i++) {
    let cur = arr[i] // 取出当前项

    // 遍历当前项之后的每一项，如果和当前项重复，则删除
    for (let j = i + 1; j < arr.length; j++) {
        if (cur === arr[j]) {
            arr.splice(j, 1)
            // 数组减一之后，造成数组塌陷，减去一项之后后一项往前移了一位，但是 j 没变，j进入下一轮循环直接跳过了往前移的那一项
            j--
        }
    }
}

console.log(arr)
```

indexof 去重

```js
arr = [1, 2, 3, 3, 4, 4, 5, 2, 1, 4];

for (let i = 0; i < arr.length; i++) {
    console.log(arr);
    
    let cur = arr[i] // 取出当前项
    // 取出当前项后面的项
    let nextArr = arr.slice(i+1)
    // 判断后面的数组中是否存在当前项，若存在，则删除当前项
    if (nextArr.indexOf(cur) !== -1) {
        arr.splice(i, 1)
        i--
    }
    
    
}
console.log(arr);
```

对象键值对去重

```js
arr = [1, 2, 3, 3, 4, 4, 5, 2, 1, 4, 7];

let obj = {}
for (let i = 0; i < arr.length; i++) {
    
    let cur = arr[i] // 取出当前项
    
    if (typeof obj[cur] !== 'undefined') {
        // arr.splice(i,1) // splice性能消耗太大
        arr[i] = arr[arr.length-1]
        arr.length--
        i--
        
        continue
    }
    
    obj[cur] = cur
}
console.log(arr);
```

相邻去重

```js
arr = [1, 2, 3, 3, 4, 4, 5, 2, 1, 4, 7];
arr.sort()

for (let i = 0; i < arr.length -1; i++) {
    if (arr[i]  === arr[i+1]) {
        arr.splice(i,1)
        i--
    }
}
console.log(arr);
```

