---
title:  js 数组排序
date: "2018-11-12 15:06:37"
sidebar: false
categories:
 - 技术
tags:
 - js
publish: true
---



### 冒泡排序

```js
let arr = [99, 1, 2, 3, 4, 6, 8, 2, 3, 42, 4, 5, 23, 54, 99]

function bubble(arr) {
    // 外层循环控制比较次数
    for (let i = 0; i < arr.length - 1; i++) {
        // 内层循环作比较
        for (let j = 0; j < arr.length -i -1; j++) {
            const value = arr[j];
            if (value > arr[j+1]) {
                const tmp = arr[j]
                arr[j] = arr[j+1]
                arr[j+1] = tmp
            }
            
        }
    }

    return arr
}

console.log(bubble(arr));

```

### 累加求1~100被3并且5整除的和

方法一

```js
let sum = 0
for (let i = 0; i < 100; i++) {
    if (i % 3 === 0 && i % 5 === 0) {
        sum +=i
    }
}
console.log(sum);

```

方法二

```js 
function he(num) {
    if (num > 10000) {
        return 0
    }

    if (num % 3 === 0 && num % 5 === 0) {
        return num + he(num + 1)
    }

    return he(num + 1)
}

console.log(he(1));

```

### 求10以内偶数的积

```js
function ji(num) {
    if (num < 1) {
        return 1
    }

    if (num % 2 === 0 ) {
        return num * ji(num-1)
    }

    return ji(num-1)
}

console.log(ji(10));

```

### 快速排序

```js
const arr = [1, 3, 99, 43, 66, 2, 345, 67]
var quickSort = function (arr) {

    if (arr.length <= 1) { return arr; }

    var pivotIndex = Math.floor(arr.length / 2);

    // 关键吧，当时错在这里了
    var pivot = arr.splice(pivotIndex, 1)[0];

    var left = [];

    var right = [];

    for (var i = 0; i < arr.length; i++) {

        if (arr[i] < pivot) {

            left.push(arr[i]);

        } else {

            right.push(arr[i]);

        }

    }

    return quickSort(left).concat([pivot], quickSort(right));
};

console.log(quickSort(arr));
```

