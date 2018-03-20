---
title: Promise实用理解
date: 2018-03-20 15:47:28
categories:
- web前端
tags:
- Promise
- JavaScript
---

### 1.Promise含义
> Promise 是异步编程的一种解决方案优于传统的解决方案——回调函数和事件。
> 简单说就是一个容器，里面保存着某个未来才会结束的事件（通常是一个异步操作）的结果。
> Promise对象代表一个异步操作，有三种状态：`pending`（进行中）、`resolved`（已成功）和`rejected`（已失败）。


### 2.Promise执行顺序
```js
setTimeout(() => {
    console.log('a');
}, 0);
var p = new Promise((resolve, reject) => {
    console.log('b');
    resolve();
});
p.then(() => {
    console.log('d');
});
console.log('c');

// 控制台输出:
// 'b'
// 'c'
// 'd'
// 'a'

```
要理解该输出顺序首先应该了解js的执行任务队列优先级（由高到低）

* 主线程
* Micro-task队列 (微任务)
* Macro-tasks队列 (宏任务)

首先`setTimeout`属于宏任务扔进Macro-tasks队列，新建实例`Promise`时接受一个回调函数作为参数，注意此时该回调函数属于主线程会立刻执行，输出`'b'`紧接着执行`resolve`也就意味着该`promise`对象的状态将从`pending`更新为`resolved`，其挂载的回调函数也就是then里面的参数函数并不会立即执行，因为它属于微任务，所以丢进Micro-task队列。接下来输出`'c'`，到目前为止主线程任务已经结束，接着执行微任务输出`'d'`,最后执行宏任务输出`'a'`。


### 3.Promise状态更新
```js
var p1 = new Promise(function (resolve, reject) { 
    resolve('p1');
});
var p2 = new Promise(function (resolve, reject) {
    setTimeout(() => {
        resolve('p2')
    }, 100);
});
var p3 = new Promise(function (resolve, reject) {
    setTimeout(() => {
        reject('p3')
        resolve('p3')
    }, 100);
});

p1.then((value) => {
    console.log(value);
})
p2.then((value) => {
    console.log(value);
})
p3.then((value) => {
    console.log('success', value);
}, (value) => {
    console.log('error', value);
})
console.log('p1:', p1);
console.log('p2:', p2);
console.log('p3:', p3);

setTimeout(() => {
    console.log('p1:', p1);
    console.log('p2:', p2);
    console.log('p3:', p3);
}, 100);

// 控制台输出
// p1: Promise {[[resolved]]: "p1"}
// p2: Promise {[[pending]]}
// p3: Promise {[[pending]]}
// p1
// p2
// error p3
// p1: Promise {[[resolved]]: "p1"}
// p2: Promise {[[resolved]]: "p2"}
// p3: Promise {[[rejected]]: "p3"}
```
p1最新创建就调用了`resolve`则它的状态立刻变为`resolved`，值为p1，但此时p2和p3都为`pending`状态，100毫秒后p2输出值p2且状态转为`resolved`。
p3首先调用了`reject`则其状态转为`rejected`，值为p3，尽管下一行又调用了resolve但并没有任何作用忽略成功的回调，只有`error p3`。
这段实验也显示出Promise的一个特点

* 调用then方法传入回调可以从外部接受promise的异步返回数据value,当嵌套多级异步操作时这种优势更大。
* 状态的不可逆性，Promise的状态和值确定下来，后续再调用resolve或reject方法，不能改变它的状态和值。

### 3.Promise之then方法
```js
new Promise((resolve, reject) => {
    setTimeout(() => {
        resolve('a');
    }, 1000);
}).then(function (value) {               
    console.log("第一个" + value);
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            resolve(value + 'b');
        }, 1000);
    })
}).then(function (value) {              
    console.log("第二个" + value);
}).then(function (value) {              
    console.log("第三个" + value);
})

// 控制台输出
// 第一个a
// 第二个ab
// 第三个undefined
```
`then`方法是Promise的实例方法，调用`then`后的返回值依然是一个promise对象，注意它是**全新的promise对象**，一般可以看到`then`是链式调用，这里需要注意区别于jQuery的链式调用。当链式调用时要注意不能被它绕晕了，要抓住一个重点，我们只是在调用`then`方法而已，给它传参只是定义函数，并没有执行！什么时候执行？是根据你的异步操作后的promise状态如何更新而确定。
`then`的参数返回值影响着它返回后的全新promise