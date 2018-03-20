---
title: Vue数据绑定
date: 2018-03-19 14:11:38
categories:
- web前端
tags:
- Vue
- JavaScript

---

#### 关于vue数据绑定的极简原理

![vue logo](https://user-gold-cdn.xitu.io/2017/8/2/c00a07c463dd341d5c0e731a9ebdca52?imageView2/1/w/800/h/600/q/85/format/webp/interlace/1)

<!-- more -->
***dep.js***
```javascript
/**
 * 对订阅者进行收集、存储和通知
 *
 * @class      Dep (name)
 */
export default class Dep {
    constructor() {
        this.subs = [];
    }

    addSub(sub) {
        this.subs.push(sub);
    }

    notify() {
        // 通知所有的订阅者（Watcher），触发订阅者的相应逻辑处理
        this.subs.forEach(sub => sub.update());
    }
}
Dep.target = null;
```
* `Dep`类用来实例一个个的收集器，每个收集器用来存储对单个数据的订阅者，当该项数据发生变化时统一的去通知他们
* `Dep.target`用来暂存当前订阅者

***observer.js***
```javascript
import Dep from './dep';

function defineReactive(obj, key, val) {
    let dep = new Dep();

    // 给当前属性的值添加监听
    observer(val);
    Object.defineProperty(obj, key, {
        enumerable: true,
        configurable: true,
        get: () => {
            console.log('get value: ', val)
            if (Dep.target) {
                dep.addSub(Dep.target)
            }
            return val;
        },

        set: (newVal) => {
            if (val === newVal) {
                console.log('无需更新')
                return;
            }
            console.log('new value setted: ', newVal)
            val = newVal;
            dep.notify();
        }
    })
}

export function observer(value) {
    if (!value || typeof value !== 'object') {
        return
    }

    Object.keys(value).forEach(key => defineReactive(value, key, value[key]));
}

```
* 核心方法[Object.defineProperty](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty)定义对象的属性访问器，从而达到对数据改变的监听。
* `observer`首先判断数据是否是对象，`Object.keys`遍历对象的每个字段对其执行`defineReactive`添加监听
* `defineReactive`首先进来实例化`Dep`，单个数据监听的收集器。这里再次对值`observer`实际上是类似递归，数据对象深层嵌套时继续监听。
* `enumerable、configurable`可枚举、可配置。`get`获取属性时，会把监听者添加到`dep`中，这边实际上是个闭包。`dep`收集器会常驻内存保留着。
* 当数据`set`设置时首先判断新值`newVal`是否与与旧值`val`相等，相等则无需更新，这边的旧值`val`也就是形参`val`也属于闭包。新旧值不等时则接下来由`dep.notify`来通知所有的监听者

***watcher.js***
```javascript
import Dep from './dep';

export default class Watcher {
    constructor(vm, expOrFn, cb) {
        this.vm = vm; //vue数据主体
        this.expOrFn = expOrFn; // 监听的字段
        this.cb = cb; // 数据变化后的回调
        this.val = this.get(); // 初始化获取数据
    }

    // 订阅数据更新时调用
    update() {
        let val = this.get();
        this.val = val;
        this.cb.call(this.vm, this.val);
    }

    get() {
        // 当前订阅者(Watcher)读取被订阅数据的最新更新后的值时，通知订阅者管理员收集当前订阅者
        Dep.target = this;
        let val = this.vm._data[this.expOrFn];

        Dep.target = null;
        return val;
    }
}

```
* `get`获取此刻的数据，首先会将自身实例挂在到`Dep.target`，其目的是访问`this.vm._data[this.expOrFn]`数据时触发监听器`observer.js`中，然后再重置`Dep.target`;
* `update`方法，数据更新后会由`dep`通知触发。从而执行回调`this.cb`，且把新值`this.val`传进回调,该方法是我们的最终目的，更新视图`view`;


***vue.js***
```javascript
import Watcher from './watcher';
import { observer } from './observer';

export default class Vue {
    constructor(options = {}) {
        this.$options = options;
        this._data = this.$options.data;
        Object.keys(this._data).forEach(key => this._proxy(key));

        observer(this._data);
    }

    $watch(expOrFn, cb) {
        new Watcher(this, expOrFn, cb);
    }

    _proxy(key) {
        Object.defineProperty(this, key, {
            configurable: true,
            enumerable: true,
            get: () => this._data[key],
            set: (val) => {
                this._data[key] = val;
            }
        })
    }
}

```
* 为使用方便把数据的监听同样添加到`Vue`实例中，通过`$watch`方法来定义监听的字段以及回调

***main.js***
```javascript
import Vue from './vue';

let demo = new Vue({
    data: {
        name: 'cheche'
    }
})

demo.$watch("name", (value) => {
    console.log("【update view111】: ", value);
})

demo.name = "meihao";

```
* 最后的使用示例