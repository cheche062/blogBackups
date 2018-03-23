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



<!-- more -->


### 2.Promise执行顺序
```js
setTimeout(() => {
    console.log('a');
}, 0);
let p = new Promise((resolve, reject) => {
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
let p1 = new Promise(function (resolve, reject) { 
    resolve('p1');
});
let p2 = new Promise(function (resolve, reject) {
    setTimeout(() => {
        resolve('p2')
    }, 100);
});
let p3 = new Promise(function (resolve, reject) {
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

### 3.Promise之then实例方法
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
    console.log(a);
}).then(function (value) {              
    console.log("第四个" + value);
}, (err) => {
    console.log("第四个error", err);
})

// 第一个a
// 第二个ab
// 第三个undefined
// 第四个error ReferenceError: a is not defined
```
`then`方法是Promise的实例方法，调用`then`后的返回值依然是一个promise对象，注意它是**全新的promise对象**，一般可以看到`then`的链式调用，这里需要注意区别于jQuery的链式调用。jQuery是返回调用对象本身。当链式调用时要注意不能被它绕晕了，要抓住一个重点，我们只是在调用`then`方法而已，给它传参只是定义函数，并没有执行！什么时候执行？是根据你的异步操作后的promise状态如何更新以及何时更新而确定。
传给`then`的回调函数中的返回值影响着最终返回出的promise对象，参数的返回值一般有三种情况。

* 一个普通的同步值，或者没写返回值默认就是`undefined`，当然它也属于普通同步值。则`then`最终返回的是状态是`resolve`成功的Promise对象，如上段代码的第三个输出，它的前一个then方法内部没有返回值则默认`undefined`，接下来就直接走进第三个`then`方法，且值`value`就是`undefined`。
* 返回新的Promise对象，`then`方法将根据这个Promise的状态和值创建一个新的Promise对象返回。如第二个输出，会等待上个then方法返回的新Promise对象状态的更新来确定，且会等待它的更新以及将最后的值传过来，这种情况也是当有多级异步操作所使用的方式。
* `throw`一个同步异常，`then`方法将返回一个`rejected`状态的Promise, 值是该异常。如第四个输出！




### 4.Promise之catch实例方法
> Promise.prototype.catch方法是then(null, rejection)的别名，用于指定发生错误时的回调函数。

```js
let p = new Promise((resolve, reject) => {
    //
});
p.then((val) => console.log('fulfilled:', val))
  .catch((err) => console.log('rejected', err));

// 等同于
p.then((val) => console.log('fulfilled:', val))
  .then(null, (err) => console.log("rejected:", err));
```

catch方法，它首先是捕捉处理错误，不论是promise调用了reject方法还是直接抛出错误，都会走到catch方法内进行处理。接下来就和then方法一样，返回的也是一个全新的Promise对象，错误处理的回调函数返回值同样有三种情况，具体看上个then方法。

```js
let p = new Promise((resolve, reject) => {
    reject('失败')
});
p.then((val) => console.log('1then: success', val))
 .then((val) => console.log('2then: success', val))
 .catch((val) => console.log('3catch: error', val))
 .catch((val) => console.log('4catch: error', val))
 .then((val) => console.log('5then: success', val))

// 控制台输出

// 3catch: error 失败
// 5then: success undefined
```

Promise 对象的错误具有“冒泡”性质，会一直向后传递，直到被捕获为止。也就是说，错误总是会被下一个catch语句捕获。
上段代码首先`p`这个Promise对象（状态是resolved）遇到第一个`then`会忽略掉它定义的成功回调，注意此时调用完第一个`then`方法后的返回值是**全新的Promise对象**！且状态同样是`resolved`，为何会这样？因为它把`p`的状态进行了一层包装也就作为了自己的状态，且值也和它一样！所以说`Promise`的状态具有传递性。

因为这个错误目前并没有被捕获处理，所以继续向后传递。同样遇到第二个`then`时我们可以当做跳过它，但发生的细节和第一个`then`同理,直到`3catch`将这个错误捕获，所以输出`3catch: error 失败`。上面也提到`catch`也就是`then`的一个别名而已，本质其实差不多。故此时`catch`调用后的返回值再次是一个全新的promise对象，那状态呢？因为这边给`catch`传递的参数并没有定义返回值，所以默认就是一个同步值`undefined`，则`catch`返回的promise对象的状态就是`resolved`。那么它调用最后一个`then`输出`5then: success undefined`，也就不难理解了。


### 5.Promise之resolve、reject静态方法
```js
let p1 = Promise.resolve('p1');
p1.then(val => console.log('success', val), val => console.log('error', val))

let p2 = Promise.reject('p2');
p2.then(val => console.log('success', val), val => console.log('error', val))
```
当传入参数是一般同步值时则返回一个状态为resolve或reject的Promise对象，值也就是传入的参数，相应的会调用成功或失败的回调。

```js
let p1 = Promise.resolve(1);
let p2 = Promise.resolve(p1);
let p3 = new Promise(function (resolve, reject) {
    resolve(p1);
});

console.log(p1 === p2)
console.log(p1 === p3)

p1.then((value) => { console.log('p1=' + value)})
p2.then((value) => { console.log('p2=' + value)})
p3.then((value) => { console.log('p3=' + value)})

// 控制台输出：
// true
// false
// p1=1
// p2=1
// p3=1
```
当传入一个Promise对象时，则`resolve`就直接返回该Promise对象，故`p1 === p2`为`true`，p3则为全新的Promise对象，但是它状态立刻变为`resolve`且值为p1，它会获取p1的状态和值作为自己的值。故`p3=1`。

### 6.Promise之all、race静态方法
```js
function timeout(who) {
    return new Promise(function (resolve, reject) {
        let wait = Math.ceil(Math.random() * 3) * 1000;
        setTimeout(function () {
            if (Math.random() > 0.5) {
                resolve(who + ' inner success');
            }
            else {
                reject(who + ' inner error');
            }
        }, wait);
        console.log(who, 'wait:', wait);
    });
}

let p1 = timeout('p1');
let p2 = timeout('p2');

p1.then((success) => { console.log(success) }).catch((error) => { console.log(error) })
p2.then((success) => { console.log(success) }).catch((error) => { console.log(error) })

// race只要有一个状态改变那就立即触发且决定整体状态失败还是成功.
// all只要有一个失败那就立即触发整体失败了，两个都成功整体才成功.
Promise.all([p1, p2])
    .then((...args) => {
        console.log('all success', args)
    })
    .catch((...args) => {
        console.log('someone error', args)
    })

// 控制台输出(情况1)
// p1 wait: 3000
// p2 wait: 1000p2 inner error
// someone error [ 'p2 inner error' ]
// p1 inner success

// 控制台输出(情况2)
// p1 wait: 2000
// p2 wait: 2000
// p1 inner success
// p2 inner success
// all success [ [ 'p1 inner success', 'p2 inner success' ] ]

```
all、race方法接受数组作为参数，且数组每个成员都为Promise对象。如果不是的话就调用Promise.resolve方法，将其转为 Promise 实例，再进一步处理。使用表示要包装的多个promise异步操作来确定。具体可以看代码理解，要多动手自己试验！


##### 参考文献：
[阮一峰ECMAScript 6 入门 Promise](http://es6.ruanyifeng.com/#docs/promise)
[八段代码彻底掌握 Promise](https://juejin.im/post/597724c26fb9a06bb75260e8)