---
title: Rxjs乱语
date: 2018-04-08 15:41:36
categories:
- web前端
tags:
- Rxjs
- JavaScript
---

### 概述：

> RxJS 是使用 Observables 基于可观测数据流在异步编程应用中的库，它使编写异步或基于回调的代码更容易。

解决异步编程我们已经有了es6的Promise，但Rx是更好的方案。

优势：
* 延迟执行
* 可传送多个值
* 可中途取消以及重试
* 可保留值待后续发送
* 函数式编程体验

<!-- more -->


### 内容：

##### observable（可被观察对象）、producer (生产者)、observer（观察者）、operator（操作符）、subscription(可退订者)

```js
import { Observable } from 'rxjs/Rx';

// 创建可观察的被观察者
let source$ = Observable.create(observer => {
    setTimeout(() => {
        // 生产者
        let producer = {
            val: 0,
            nextValue: function(){
                this.val = this.val + 1
                return this.val
            }
        }
        observer.next(producer.nextValue())
        observer.next(producer.nextValue())
    }, 100)
})
// 操作符
.map(val => val * 2)
//观察者1
let observer1 = {
    next: val => console.log('A:', val)
}
//观察者2
let observer2 = {
    next: val => console.log('B:' , val)
}
//观察者3
let observer2 = {
    next: val => console.log('B:' , val)
}
//绑定订阅
source$.subscribe(observer1)
source$.subscribe(observer2)
// 订阅者
let subscription = source$.subscribe(observer3)
subscription.unsubscribe();
// 输出
// A: 1
// A: 2
// B: 1
// B: 2
```
上面这个例子来个初体验，通过`Observable.create`来创建一个可被观察的对象，可理解为一个数据流`stream`的开端，注意创建时给它传入的回调函数并未立刻执行，这一点需区别于Promise的立刻执行性。

`.map()`是一个操作符，可将它的意义理解为对数据流进行相应的操作，并将数据流继续向下流。**注意：**`map`操作符的返回值是一个新的`observable`也就是这里的`source$`最终的可观测对象！
这样实现的目的同样也具有函数式的体验，**不破坏原始数据流**，你可以监听之前的数据流而不受本次操作符所影响。

接下来的两个观察者`observer`当然就是监听最终数据，它是一个对象且拥有`next`方法，该方法往往就是更新视图`view`了。在对被观察者进行订阅时`subscribe`传入观察者`observer`，此时此刻传入`Observable.create`方法的回调函数才会执行，并开启一个延时100毫秒的定时器。

`observer`形参可以理解为就是传入的观察者，当然它并不完全是，实际它是由Rx内部重新包装过的标准观察者，目的：一方面是为了兼容使用者的简写变得更简单，另一方面它需要功能扩展的其它全方法规范应用。调用观察者的`next`方法并给他传值，可以是异步请求的结果值送出，而这边的值也是经过了Rx包装过的通过一个生产者`producer`处理或者存储再送出，可以发现这里能向外送出多个值。数据会经过`map`处理将每个值*2，这里也体现了函数式的优势，map操作不参与内部值的生产，只参加数据的操作。

`subscription`可以取消观察者的订阅！可订可退出！优于`promise`的无法中途退出的弊端



### 实践：

##### 需求
* 用户向输入框输入内容则发送请求数据
* 限制用户的请求频率，停止输入后200毫秒后发送
* 过滤用户的空内容
* 服务端请求结果响应时间是不固定的，所以在下一个请求发出前要忽略本次请求的结果，就是后一个请求要覆盖干掉前者
* 将结果进行长度排序
* 清空输入框的内容同时展示数据
* 给渲染出来的数据子项添加鼠标移入移出的背景变色事件以及点击事件则删除该子项

**具体代码：**

```js
import $ from 'jquery';
import Rx, { Observable } from 'rxjs/Rx';
import { setupRxDevtools } from 'rx-devtools/rx-devtools';
import 'rx-devtools/add/operator/debug';

setupRxDevtools();

const SEARCH_REPOS = 'https://api.github.com/search/repositories?sort=stars&order=desc&q=';

// 输入关键字进行异步查找并排序
let input = document.getElementById('input')
let input$ = Observable.fromEvent(input, 'input')

let app$ = input$
    .debounceTime(200)
    .debug('查询')
    .map(e => e.target.value)
    .filter((text) => !!text)
    .switchMap(key => {
        return Observable.create((observer) => {
            $.ajax({
                url: SEARCH_REPOS + key,
                success: (data) => {
                    let result = data.items.map((item) => item.name)
                    observer.next(result)
                }
            })
        })
        // outerVal就是输入框的内容值，
        // innerVal就是请求后的结果数据
    }, (outerVal, innerVal) => {
        return innerVal.filter(item => item.includes(outerVal))
    })
    // 长度排序
    .map(data => data.sort((a, b) => a.length - b.length))
    // 清空内容
    .do(() => $('#list-group').html(''))
    // 数据转流
    .mergeMap(data => Observable.from(data))
    // 转成jq对象
    .map(item => $(`<li>${item}</li>`))
    // 添加到列表里
    .do($dom => $('#list-group').append($dom))
    // 背景颜色切换
    .mergeMap(toggleBgColor)

// 背景颜色切换
function toggleBgColor($dom) {
    // 创建鼠标移入数据流
    let mouseover$ = Observable.fromEvent($dom, 'mouseover')
    // 创建鼠标移出数据流
    let mouseout$ = Observable.fromEvent($dom, 'mouseout')
    // 创建鼠标点击数据流
    let click$ = Observable.fromEvent($dom, 'click')
    click$ = click$.do((e) => {
        $(e.target).remove()
    })
    mouseover$ = mouseover$.do((e) => {
        $(e.target).css('backgroundColor', 'green')
    })
    mouseout$ = mouseout$.do((e) => {
        $(e.target).css('backgroundColor', 'white')
    })

    // 合并数据流转为其它数据
    return mouseover$.merge(mouseout$, click$)
        .map($dom => $dom.target.innerHTML)
}

app$.subscribe(console.log);
```


##### 调试：
其中使用了用于Rx的调试工具，需要安装两个东西
1. npm install rx-devtools --save-dev
2. 谷歌的扩展插件[RxJS developer tools](https://chrome.google.com/webstore/detail/rxjs-developer-tools/dedeglckjaldaochjmnochcfamanokie)

使用方式如上述代码所示，使用需要注意几个要点
* 在顶部引入并执行`setupRxDevtools`方法
* `.debug('查询')`方法在需要调试的流中插入即可，`查询`名字可随意取
* 在页面刷新前需要打开`Rx Devtools`页签，如图：

![rx1](/images/debug/rx1.png)
* 选中"查询"当有数据流产生时会同时映射到右图，点击相应的操作符上面的玻璃球，就能看到流进该操作的实时数据便于调试。

![rx2](/images/debug/rx2.png)

##### 分析：
* 这种编程体验很优雅，通过一个个操作符将应用需求组装起来，易扩展和维护，代码看起来很简洁干净。
* 操作符永远不破坏原始数据流仅仅返回新的数据流，所以仍然可以监听之前的数据流，如：`input$`和`app$`，map的操作并妨碍影响对`input$`的数据监听。

```js
let input = document.getElementById('input')
let input$ = Observable.fromEvent(input, 'input')
let app$ = input$.map(e => e.target.value)

input$.subscribe(val => console.log('A', val))
app$.subscribe(val => console.log('B', val))
```


### 相关文档：

[RxJS 5 基本原理](https://rxjs-cn.github.io/rxjs5-ultimate-cn/)
[RxJs中文文档](http://cn.rx.js.org/)
