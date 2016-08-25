title:  Web 快速入门
---

## 文档概览

实时数据库是野狗后端云最核心的功能，通过 SDK 的形式提供数据存储功能，无需编写后端 代码也无需租用服务器。同时支持 web，ios，android等平台，在此只描述 Web 平台。

希望了解更多有关于实时数据库的功能特性，可以进入[实时数据库介绍](#)了解详情

如果您是第一次使用，您可以通过[快速起步](#)迅速了解实时数据库如何使用。完成快速起步之后您将会知道如何实现野狗实时数据库基本的连接、保存数据、读取数据以及实时同步功能。

如果您已经了解了以上内容，想使用更多更强大的功能，您可以直接前往[开发者指南](#)。

野狗实时数据库为您的数据提供了强大的安全保障，您可以通过我们提供的[规则表达式](#)配置强大灵活的安全策略。

我们还提供了很多应用场景，您已经对我们的有了一定认识，也知道基本的操作。那么您可以查看我们提供的 Demo，试着跟着Demo 教程，制作一些有意思的应用。

[打飞机对战游戏](#)
[实时排行榜](#)
[创建一个共享画板](#)


## 快速起步

我们通过编写一个简单的天气应用例子来了解实时数据库是如何使用的。
通过这个例子，你将会学到：
1. 向数据库中保存、修改数据。
2. 读取并且实时监听数据。

![](http://7u2r36.com1.z0.glb.clouddn.com/16-8-18/29982723.jpg)

> 快速起步之前，需要先创建您的应用，如果您还不知道如何创建应用，请先阅读[控制台指南—快速起步](#)
>

## 1. 引入 SDK
首先应该在页面中引入我们的 Wilddog SDK
很简单，只需要在你的页面中加入一行 javascript 标签。

```javascript
<script src = "https://cdn.wilddog.com/js/client/current/wilddog.js" ></script>
```

Node.js 用户(收起△)
如果您需要在 Node.js 环境中使用 Wilddog SDK，可以通过npm安装的方式。

```javascript
$npm install wilddog --save
```

安装完成之后 require Wilddog 实例

```javascript
var Wilddog = require("wilddog");
```

Typescript 用户(收起△)
您可以使用我们提供的 .d.ts 文件。
下载地址：https://github.com/WildDogTeam/wilddog-typescript

## 2. 创建Wilddog 引用
引入 Wilddog SDK 之后我们需要创建 Wilddog引用。有了 Wilddog 引用我们才能对数据进行操作。
让我们来创建一个 Wilddog引用对象。

```javascript
var ref = new Wilddog("https://<appId>.wilddogio.com/");
```

这样就创建完成了。
创建对象的时候，需要传入数据库的数据路径。上面的代码定位在数据库的根节点，
您也可以传入更具体的数据路径，Url与数据节点的关系如下图所示：
![](http://7u2r36.com1.z0.glb.clouddn.com/16-8-18/2316950.jpg)
比如我要在成都的天气下传入数据，那么我可以将输入的 Url定位为成都的 weather 节点下

```javascript
var ref = new Wilddog("https://<appId>.wilddogio.com/成都/weather");
```



## 3. 数据操作

保存数据
创建了 Wilddog 对象之后，我们就能利用它对数据进行操作了。让我们先从写入数据开始。
我们可以通过 Wilddog 提供的 set() 方法，写入 JSON 数据。
假设我们要在数据库中的根节点下存入 成都的天气 信息：

```javascript
ref.set({
"city" : "成都",
"weather" : "晴天"
});
```


信息就直接存入数据库了，非常方便。您可以查看下面的在线示例
[保存数据示例](#)

删除数据和更新数据，你可以查看开发者指南中的删除数据



## 4. 读取与监听数据
我们上一步已经把 成都的天气 的信息存入了数据库，那么我们就可以使用 on() 函数来读取存入的信息。
例如我们要知道成都的天气信息，我们通过 on() 来读取 weather 字段：

```javascript
ref.on("value", function(snapshot) {
console.log(snapshot.val());
}
```


这样就能读出根节点下的所有数据了,snapshot.val()函数返回的就是取出的 json数据。

如果你想读取某个节点的数据，比如只想看成都的天气，那么我们只需要在引用对象与 on() 之间后面加上 child(节点名称),就能够返回该节点的所有数据。

```javascript
ref.child("成都").on("value", function(snapshot) {
console.log(snapshot.val());
}
```


这样就能够从数据库中取出 weather 的数据了，

读取数据是通过绑定回调函数来实现的，我们使用 on()函数读取数据的时候使用了一个回调函数，回调函数的参数是一个 snapshot 对象类型，调用它的 val() 函数能够读取到返回的数据。
您可以点击下面链接的查看示例
[读取与监听数据]()

实时同步
上边这个例子中，使用 on()函数读取数据，value 这个事件会在初次读取到数据的时候被触发一次，并且此后每当数据发生改变的时候都会被触发。如果您在数据库中修改了数据，其他平台的数据会同步更新。

![](http://7u2r36.com1.z0.glb.clouddn.com/AQujQROxAxUc3Bxp.gif%21thumbnail.gif)

这是一个非常棒的特性，您可以用它实现很多很棒的功能，例如 xxx，xxx。
如果您只想读取一次，以后每次数据发生变化的时候将不再同步，那么您可以使用once()函数替代on()函数。

## 

引导向开发者指南

PS：数据安全以及规则表达式



## 快速起步中可能会遇到的问题
1. json序列化







