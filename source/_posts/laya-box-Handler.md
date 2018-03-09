---
title: LayaBox源码解析-Handler
date: 2018-03-09 11:30:09
categories:
- web前端
tags:
- layabox
- actionscript
- javascript

---
(本文是as和js混合，其实两者差不多，一个妈生的。^_^!!)
### 概述：
> 推荐使用 `Handler.create()` 方法从对象池创建，减少对象创建消耗。
> 创建的 Handler 对象不再使用后，可以使用 `Handler.recover()`将其回收到对象池

**注意：** 由于鼠标事件也用本对象池，不正确的回收及调用，可能会影响鼠标事件的执行。


<!-- more -->

### 作用：
* 绑定作用域
* 携带参数
* 是否执行一次以及回收

我用普通函数加上`call`、`apply`修正`this`指向外加标志符之类的也能现实为什么非要用它呢？
当应用程序变得越来越庞大和复杂时，自然而然就需要有一个统一管理的处理器。比如在A处定义好函数、作用域和参数但是此时不想立刻执行，待到B处再执行，则作用就体现出来，以及参数可以两处进行累加、对函数的及时回收将变得异常方便。把这些细节处理隐藏起来，暴露在外可以做到简洁，且对新手使用更加友好！

### 属性：
* `static` _pool:Array = [];  //对象池用于缓存节约资源
* `static` _gid:int = 1;  	 //用于计数id
* caller:*;  		//执行域也叫执行上下文对象
* method:Function;  //处理方法（主角）
* args:Array;  		//携带的参数
* once:Boolean = false;  //是否只执行一次
* _id:int = 0;  	//携带的id


### 方法：
**1. create:**
```actionscript
public static function create(caller:*, method:Function, args:Array = null, once:Boolean = true):Handler {
	if (_pool.length) return _pool.pop().setTo(caller, method, args, once);
	return new Handler(caller, method, args, once);
}
```
新建handler实例的推荐方式是`Handler.create`,原因在于方法第一步的判断，如果缓存对象池内有缓存则直接从缓存取出从而节约资源。接下来不论是`setTo`还是`new Handler`作用是一样的，后者也是在调用前者。

**2. setTo:**
```actionscript
public function setTo(caller:*, method:Function, args:Array, once:Boolean):Handler {
	_id = _gid++;
	this.caller = caller;
	this.method = method;
	this.args = args;
	this.once = once;
	return this;
}
```
首先静态属性`_gid`累计自增作为实例属性`_id`的值，表示每个实例有一个自己独一无二的`_id`，以及给各自实例属性赋值。

**3. run:**
```actionscript
public function run():* {
	if (method == null) return null;
	var id:int = _id;
	var result:* = method.apply(caller, args);
	_id === id && once && recover();
	return result;
}
```
`method`就是最初传进来的处理函数，通过`method.apply(caller, args)`传进`this`指向以及携带的参数列表`args`。且该处理函数的返回值同时也作为`run`的返回值返回。`_id === id && once && recover();`保证id不变的情况下如果是一次性处理函数则及时`recover`回收。

**4. runWith:**
```actionscript
public function runWith(data:*):* {
    if (method == null) return null;
    var id:int = _id;
    if (data == null)
        var result:* = method.apply(caller, args);
    else if (args) result = method.apply(caller, args.concat(data));
    else result = method.apply(caller, data);
    _id === id && once && recover();
    return result;
}
```
`runWith`与`run`的区别就是它可以继续传入参数与之前的参数通过`args.concat(data)`累加作为`method.apply`执行的第二个参数。
**注意：**`concat`方法的参数可以是数组也可以是非数组。数组的话就只是展开一维数组，非数组的话就直接当做`length`为1的一维数组合并。

**5. recover:**
```actionscript
public function recover():void {
    if (_id > 0) {
        _id = 0;
        _pool.push(clear());
    }
}
```
但凡`_id`大于0的则需要回收，因为已经回收的`_id`随即赋值为0了，并且回收扔进该类的静态属性`_pool`数组对象池中，以待下次利用。

**6. clear:**
```actionscript
public function clear():Handler {
    this.caller = null;
    this.method = null;
    this.args = null;
    return this;
}

```
清除自身但是`id`并不在这边清0，那是交给回收时清除，这也很好理解只是清除自身，可能没用了但是他还在这，并没有回收它，故`id`目前依然保留。


### 使用场景：
1. Laya.loader.load(url, Handler.create(this, function(){}));
2. node.mouseHandler = Handler.create(this, function(){}, null, false);
3. list.selectHandler = Handler.create(this, function(){}, null, false);

**常见误区：**
当需要多次执行时的场景时，`Handler.create()`方法的第四个参数是否是一次性，当不传该参数默认是`true`,事件则只会相应一次。常出现在给元素添加事件和加载资源时的进度条更新的回调函数。