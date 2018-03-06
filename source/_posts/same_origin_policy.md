---
title: 浏览器的同源策略及解决方案
date: 2018-03-06 15:20:39
categories:
- web前端
tags:
- Cookie
- AJAX
- Iframe

---

### 一. 含义
最初是指一张网页不能读取另一张网页的cookie,除非两张网页是同源的。

* 协议相同
* 域名相同
* 端口号相同

<!-- more -->
**例子**
`http://www.example.com/dir/page.html`
协议是`http：//`, 域名是`www.example.com`,端口号是`80`（默认可以省略）


**目的**
同源政策的目的，是为了保证用户信息的安全，防止恶意的网站窃取数据。

**限制范围**
1. Cookie、LocalStorage 和 IndexDB 无法读取。
2. DOM无法获得。
3. AJAX无法请求

虽然这些限制是必要的，但是有时很不方便，合理的用途也受到影响。下面详细介绍，如何规避上面三种限制。

### 二. Cookie
Cookie 是服务器写入浏览器的一小段信息，只有同源的网页才能共享。但是，两个网页一级域名相同，只是二级域名不同，浏览器允许通过设置`document.domain`共享 Cookie。
```
document.domain = 'example.com';
```
注意，这种方法只适用于 `Cookie `和 `iframe` 窗口，`LocalStorage` 和 `IndexDB` 无法通过这种方法，规避同源政策，而要使用下文介绍的`PostMessage API`。

另外，服务器也可以在设置`Cookie`的时候，指定`Cookie`的所属域名为一级域名，比如`.example.com`
```
Set-Cookie: key=value; domain=.example.com; path=/
```
这样的话，二级域名和三级域名不用做任何设置，都可以读取这个Cookie。

### 三. iframe

如果两个网页不同源，就无法拿到对方的`DOM`。典型的例子是`iframe`窗口和`window.open`方法打开的窗口，它们与父窗口无法通信。

如果两个窗口一级域名相同，只是二级域名不同，那么设置上一节介绍的`document.domain`属性，就可以规避同源政策，拿到`DOM`。

对于完全不同源的网站，目前有三种方法，可以解决跨域窗口的通信问题。

1. 片段标识符
2. window.name
3. postmessage 垮文档通信api

##### 1.片段标识符
（fragment identifier）指的是，URL的#号后面的部分，比如`http://example.com/x.html#fragment`的`#fragment`。如果只是改变片段标识符，页面不会重新刷新。也称做哈希值

```
window.onhashchange = checkMessage;

function checkMessage() {
  var message = window.location.hash;
  // ...
}
```
##### 2.window.name
浏览器窗口有window.name属性。这个属性的最大特点是，无论是否同源，只要在同一个窗口里，前一个网页设置了这个属性，后一个网页可以读取它。

父窗口先打开一个子窗口，载入一个不同源的网页，该网页将信息写入window.name属性。
这种方法的优点是，window.name容量很大，可以放置非常长的字符串；缺点是必须监听子窗口window.name属性的变化，影响网页性能。

##### 3.window.postMessage
上面两种方法都属于破解，HTML5为了解决这个问题，引入了一个全新的API：跨文档通信 API（Cross-document messaging）。

这个API为window对象新增了一个window.postMessage方法，允许跨窗口通信，不论这两个窗口是否同源。
```js
var popup = window.open('http://bbb.com', 'title');
popup.postMessage('Hello World!', 'http://bbb.com');
```
`postMessage`方法的第一个参数是具体的信息内容，第二个参数是接收消息的窗口的源（origin），即` "协议 + 域名 + 端口" `。也可以设为*，表示不限制域名，向所有窗口发送。

```js
//子窗口向父窗口发送消息的写法类似。
window.opener.postMessage('Nice to see you', 'http://aaa.com');

//父窗口和子窗口都可以通过message事件，监听对方的消息。
window.addEventListener('message', function(e) {
  console.log(e.data);
},false);
```

message事件的事件对象event，提供以下三个属性。

* event.source：发送消息的窗口
* event.origin: 消息发向的网址
* event.data: 消息内容

`event.origin`属性可以过滤不是发给本窗口的消息。
```js
window.addEventListener('message', receiveMessage);
function receiveMessage(event) {
  if (event.origin !== 'http://aaa.com') return;
  if (event.data === 'Hello World') {
      event.source.postMessage('Hello', event.origin);
  } else {
    console.log(event.data);
  }
}
```

### 四.AJAX
同源政策规定，AJAX请求只能发给同源的网址，否则就报错。

除了架设服务器代理（浏览器请求同源服务器，再由后者请求外部服务），有三种方法规避这个限制。

1. JSONP
2. WebSocket
3. CORS

##### 1.JSONP
JSONP是服务器与客户端跨源通信的常用方法。最大特点就是简单适用，老式浏览器全部支持，服务器改造非常小。

它的基本思想是，网页通过添加一个`<script>`元素，向服务器请求JSON数据，这种做法不受同源政策限制；服务器收到请求后，将数据放在一个指定名字的回调函数里传回来。

首先，网页动态插入元素，由它向跨源网址发出请求。
```js
function addScriptTag(src) {
  var script = document.createElement('script');
  script.setAttribute("type","text/javascript");
  script.src = src;
  document.body.appendChild(script);
}

window.onload = function () {
  addScriptTag('http://example.com/ip?callback=foo');
}

function foo(data) {
  console.log('Your public IP address is: ' + data.ip);
};
```
上面代码通过动态添加`<script>`元素，向服务器example.com发出请求。注意，该请求的查询字符串有一个callback参数，用来指定回调函数的名字，这对于JSONP是必需的。

服务器收到这个请求以后，会将数据放在回调函数的参数位置返回。
```js
foo({
  "ip": "8.8.8.8"
});
```
由于`<script>`元素请求的脚本，直接作为代码运行。这时，只要浏览器定义了foo函数，该函数就会立即调用。作为参数的JSON数据被视为JavaScript对象，而不是字符串，因此避免了使用JSON.parse的步骤。

##### 2.WebSocket
WebSocket是一种通信协议，使用`ws://（非加密）`和`wss://（加密）`作为协议前缀。该协议不实行同源政策，只要服务器支持，就可以通过它进行跨源通信。


##### 3.CORS
CORS是跨源资源分享（Cross-Origin Resource Sharing）的缩写。它是W3C标准，是跨源AJAX请求的根本解决方法。相比JSONP只能发GET请求，CORS允许任何类型的请求。






