title:  Web 开发者指南
---
阅读本文档前请确保对 Wilddog 数据库产品形态有大致了解，并且已能够简单的读写数据。[快速入门](/docs/quickstart/web.html) 能帮助你达到这一目的。

## 连接管理
对 Wilddog 云端数据库任何操作都要确保与 Wilddog 云端长连接建立成功。那么如何检测连接状态？Wilddog 对离线事件的支持又是怎样的？

#### 检查连接状态
在许多应用场景下，客户端需要知道自己是否在线。Wilddog客户端提供了一个特殊的数据地址：/.info/connected。每当客户端的连接状态发生改变时，这个地址的数据都会被更新。
``` js
var connectedRef = new Wilddog("https://samplechat.wilddogio.com/.info/connected");
connectedRef.on("value", function(snap) {
  if (snap.val() === true) {
    alert("connected");
  } else {
    alert("not connected");
  }
});
```
/.info/connected的值是boolean类型的，它不会和云端进行同步。

#### 离线事件

如果你想在监听到客户端断线后自动触发一些事件。例如，当一个用户的网络连接中断时，希望标记这个用户“离线”状态。Wilddog提供的离线事件功能实现这一需求。

离线事件能在云端检测到客户端连接断开时，将指定的数据写入云数据库中。不论是客户端主动断开，还是意外的网络中断，甚至是客户端应用崩溃，这些数据写入动作都将会被执行。因此我们可以依靠这个功能，在用户离线的时候，做一些数据清理工作。Wilddog 支持的所有数据写入动作，包括 set, update，remove，都可以设置在离线事件中执行。

下面是一个例子，使用`onDisconnect()`方法，在离线的时候写入数据：

```js
var presenceRef = new Wilddog('https://samplechat.wilddogio.com/disconnectmessage');
// 当客户端连接断开时，写入一个字符串
presenceRef.onDisconnect().set("I disconnected!");
```

**离线事件是如何工作的**

当进行了一个`onDisconnect()`调用之后，这个事件将会被记录在云端。云端会监控每一个客户端的连接。如果发生了超时，或者客户端主动断开连接，云端就触发记录的离线事件。

客户端可以通过回调方法，确保离线事件被云端正确记录了：

```js
presenceRef.onDisconnect().remove( function(err) {
  if(err) {
    console.error('could not establish onDisconnect event', err);
  }
});
```

要取消一个离线事件，可以使用`cancel()`方法：

```js
var onDisconnectRef = presenceRef.onDisconnect();
onDisconnectRef.set('I disconnected');
// 要取消离线事件
onDisconnectRef.cancel();
```
<br>

#### 手动建立或断开连接
Wilddog 也提供了手动建立或者断开连接的方法。示例如下：

```js
var ref = new Wilddog("https://samplechat.wilddogio.com/users");
Wilddog.goOffline(); // All local Wilddog instances are disconnected
Wilddog.goOnline(); // All local Wildodg instances automatically reconnect
```

## 保存数据

以下四种方法可用于将数据写入野狗云端：
* `set()`：将数据写入到指定的路径，如果指定路径已存在数据，那么数据将会被覆盖。
* `push()`：添加数据到列表。向指定路径下添加数据，由野狗自动生成唯一key。例如向 /posts 路径下 push 数据，数据会写入到/posts/<unique-post-id>下。
* `update()`：更新指定路径下的部分key的值，而不替换所有数据。
* `transaction()`：更新可能因并发更新而损坏的复杂数据。

#### 用 set() 写入数据

`set()` 是最基本的写数据操作，它会将数据写入当前引用指向的节点。该节点如果已有数据，任何原有数据都将被删除和覆盖，包括其子节点的数据。
`set()` 可以传入几种数据类型 `string`, `number`, `boolean`, `object` 做为参数。
例如，社交博客应用可以使用 `set()` 添加用户信息，如下所示：

```js
var ref = new Wilddog("https://<appId>.wilddogio.com/users");
ref.child(userId).set({
    username: name,
    email: email
});
```

野狗采用的是一个“数据同步”的架构。本地拥有数据副本。对数据的写入操作，首先写入本地副本，然后SDK去将数据与云端进行同步。
也就是说，当 `set()` 方法返回的时候，数据可能还没有同步到云端。
若要确保同步到云端完成，需要使用 `set()` 方法的第二个参数，该参数是一个回调函数，代码示例如下：

```js
var ref = new Wilddog("https://<appId>.wilddogio.com/users");
ref.child(userId).set({
    username: name,
    email: email
}, function(error) {
    if (error == null){
        // 数据同步到野狗云端成功完成
    }
});
```

#### 使用 update() 更新数据

如果要同时更新多个子节点，而不覆盖其它的子节点，可以使用 update() 方法:

```js
//原数据如下
{
    "Grace": {
        "nickname": "Nice Grace",
        "birthday": "19930118",
        "fullname": "Grace Lee"
    }
}

// 只更新 Grace 的 nickname
var hopperRef = usersRef.child("Grace");
hopperRef.update({
  "nickname": "Amazing Grace"
});

```

这样会更新 Grace 的 nickname 字段。如果我们用 set() 而不是 update()，那么 birthday 和 fullname 都会被删除。


#### 使用 push() 追加到数据列表
当多个用户同时试图在一个节点下新增一个子节点的时候，这时，数据就会被重写，覆盖。
为了解决这个问题，Wilddog push()采用了生成唯一ID 作为key的方式。通过这种方式，多个用户同时在一个节点下面push 数据，他们的key一定是不同的。这个key是通过一个基于时间戳和随机的算法生成的，即使在一毫秒内也不会相同，并且表明了时间的先后，wilddog采用了足够多的位数保证唯一性。

用户可以用push向博客app中写新内容：


```js
  var postsRef = ref.child("posts");

  postsRef.push({
    author: "gracehop",
    title: "Announcing COBOL, a New Programming Language"
  });

  postsRef.push({
    author: "alanisawesome",
    title: "The Turing Machine"
  });

```

产生的数据都有一个唯一ID:
```json
{

  "posts": {
    "-JRHTHaIs-jNPLXO": {
      "author": "gracehop",
      "title": "Announcing COBOL, a New Programming Language"
    },

    "-JRHTHaKuITFIhnj": {
      "author": "alanisawesome",
      "title": "The Turing Machine"
    }
  }
}
```

**获取唯一ID**
调用push会返回一个引用，这个引用指向新增数据所在的节点。你可以通过调用 `key()` 来获取这个唯一ID

```js
 // 通过push()来获得一个新的数据库地址
var newPostRef = postsRef.push({
	author: "gracehop",
	title: "Announcing COBOL, a New Programming Language"
});

// 获取push()生成的唯一ID
var postID = newPostRef.key();

```
#### 删除数据
删除数据最简单的方法是在引用上对这些数据所处的位置调用 `remove()`。

此外，还可以通过将 null 指定为另一个写入操作（例如，`set()` 或 `update()`）的值来删除数据。 您可以结合使用此方法与 update()，在单一 API 调用中删除多个子节点。

注意：Wilddog 不会保存空路径，即如果 /a/b/c 节点下的值被设为 null，这条路径下又没其他的含有非空值的子节点存在，那么云端就会把这条路径删除。

#### 事务操作
处理可能因并发修改而损坏的数据（例如，增量计数器）时，可以使用事务处理操作。您可以为此操作提供更新函数和可选完成回调。

更新函数会获取当前值作为参数，当你的数据提交到服务端时，会判断你调用的更新函数传递的当前值是否与实际当前值相等，如果相等，则更新数据为你提交的数据，如果不相等，则返回新的当前值，更新函数使用新的当前值和你提交的数据重复尝试更新，直到成功为止。

举例说明，如果我们想在一个的博文上计算点赞的数量，可以这样写一个事务：
```js
var upvotesRef = new Wilddog('https://docs-examples.wilddogio.com/saving-data/wildblog/posts/-JRHTHaIs-jNPLXOQivY/upvotes');

upvotesRef.transaction(function (currentValue) {
  return (currentValue || 0) + 1;
});
```
我们使用currentValue || 0来判断计数器是否为空或者是自增加。 如果上面的代码没有使用事务, 当两个客户端在同时试图累加，那结果可能是为数字 1 而非数字 2。

注意：transaction()可能被多次被调用，必须处理 currentData 变量为 null 的情况。 当执行事务时，云端有数据存在，但是本地可能没有缓存，此时 currentData 为 null。


## 读取和监听数据
本部分将介绍如何读取数据以及如何对 Wilddog 的数据进行排序和过滤（条件查询）。
另外，需要提到的一点是，Wilddog SDK的数据读取都是建立在添加监听基础上的。

#### 监听的事件类型
使用 on() 方法添加一个监听事件。监听在初始化时会触发一次，然后满足事件的特定类型时会再次触发。一共有以下几种事件类型：

事件     | 描述
-------- | ---
value | 当有数据请求或有任何数据发生变化时触发
child_added    | 当有新增子节点时触发
child_changed     | 当某个子节点发生变化时触发
child_removed	    | 当有子节点被删除时触发
child_moved     | 当有子节排序发生变化时触发

将 child_added、child_changed 和 child_removed 配合使用，即可监控到对子节点列表做出各种的更改。

**value事件**
使用 value 事件来读取当前节点下的数据的静态快照。
此方法在初始化时会触发一次，此后每当有数据变化都会被再次触发。
数据（包括子节点）的快照会以事件回调形式返回。如果没有任何数据，则会返回 null。

注意：每当指定的数据库路径下的数据（包括子节点数据）有改变时，都会触发 value 事件。所以，为了聚焦你只关心的数据并降低快照的大小，你应该把要监听的节点路径设置的更加精确。
例如，如果不是必要，尽量不要在根路径设置 value 监听。

下面的例子演示了在一个博客应用中获取 gracehop 的个人信息。

```js
var ref = new Wilddog('https://docs-examples.wilddogio.com/web/saving-data/wildblog/users/gracehop');
ref.on('value', function(snapshot) {
	console.log(JSON.stringify(snapshot.val()));
});
```
回调的 snapshot 会包含指定路径下的数据。使用 .val() 方法来获取 snapshot 中的数据。

**子节点事件**

#### 移除监听

#### 只读取一次数据

#### 排序和过滤数据

**排序**
**过滤**
**排序规则**

## 用户认证

此部分目前先不写，要结合新版 Auth 写个总的。

## 完整 API 文档

用现有的就行，不过要将 API 分类，并且丰富代码片段。


## 错误码说明

## 参考资源

#### 示例列表

[在此](https://www.wilddog.com/download/download-js-source)

#### 示例教程

[官方--弹幕](https://github.com/WildDogTeam/demo-js-danmu)

[第三方--游戏排行榜](http://www.jianshu.com/p/8be5331d92d9)


