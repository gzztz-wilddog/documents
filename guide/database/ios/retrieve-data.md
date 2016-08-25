title:  读取和查询数据
---
本部分将介绍如何读取数据以及如何对数据进行排序和查询。
需要先提到的一点是，Wilddog SDK 的数据读取都是建立在添加监听的基础上，然后在监听的回调函数中完成对数据的读取。

## 监听的事件类型

使用 observeEventType 方法添加一个监听事件。一共有以下几种事件类型：

事件     | 描述
-------- | ---
WEventTypeValue | 当程序初始化时或有任何数据发生变化时触发
WEventTypeChildAdded    | 当程序初始化时或有新增子节点时触发
WEventTypeChildChanged     | 当某个子节点发生变化时触发
WEventTypeChildRemoved	    | 当有子节点被删除时触发
WEventTypeChildMoved     | 当有子节点排序发生变化时触发

将 WEventTypeChildAdded、WEventTypeChildChanged 和 WEventTypeChildRemoved 配合使用，即可监听到对子节点做出各种的更改。

#### value 事件 

使用 WEventTypeValue 事件来读取当前节点下的所有数据的静态快照。
此方法在初始化时会触发一次，此后每当有数据（包括任何子节点）变化都会被再次触发。初始化时，如果没有任何数据，则 snapshot 返回的 value 为 nil。
数据（包括子节点）的快照会以事件回调形式返回。

**注意**：每当指定路径下的数据（包括更深层节点数据）有改变时，都会触发 value 事件。所以，为了聚焦你只关心的数据并降低快照的大小，你应该把要监听的节点路径设置的更加精确。
例如，如果不是必要，尽量不要在根路径设置 value 监听。

以下示例演示了社交博客应用如何从数据库中检索博文详细信息：

Objective-C

```
_refHandle = [_postRef observeEventType:WEventTypeValue withBlock:^(WDataSnapshot * _Nonnull snapshot) {
    NSDictionary *postDict = snapshot.value;
    // ...
}];

```

Swift

```     
refHandle = postRef.observeEventType(WEventType.Value, withBlock: { (snapshot) in
  let postDict = snapshot.value as! [String : AnyObject]
  // ...
})

```

用 observeEventType 方法监听收到一个 WDataSnapshot，其中包含着 value 事件发生时，数据库中 postRef 指定节点下的所有数据。 如果该位置不存在任何数据，则 value 为 nil。

`WDataSnapshot` 里封装了一些常用的方法，帮助你更方便的处理数据，常用的列举如下：

方法     | 说明
-------- | ---
value | 返回当前快照的数据
children    | 获取当前快照中，所有子节点的迭代器，可用来遍历快照中每一个子节点
childrenCount    | 返回当前节点中子节点的个数
exists     | 如果 snapshot 对象包含数据返回 true，否则返回false
hasChildren     | 检查是否存在某个子节点

更多更详细的用法说明参见 [API](/docs/guide/database/ios/api.html#WDataSnapshot-Methods) 文档。

#### child 事件
当某个节点的子节点发生改变时（如通过 `childByAutoId` 方法添加子节点，或通过 `updateChildValues` 更新子节点），就会触发 `child 事件`。

`WEventTypeChildAdded` 事件常用来获取当前路径下的子节点列表。初始化时将针对每个现有的子节点触发一次(例如：列表拥有10个子节点，那么该方法就会触发10次)。之后每当增加子节点时就会再次触发，在回调中只获取新增的子节点数据。

每次子节点修改时，均会触发 `WEventTypeChildChanged` 事件。这包括对子节点的后代所做的任何修改。

删除直接子节点时，将会触发 `WEventTypeChildRemoved` 事件。传递给回调块的快照包含已删除的子节点的数据。

每当因更新（导致子节点重新排序）而触发 `WEventTypeChildChanged` 事件时，系统就会触发 `WEventTypeChildMoved` 事件。该事件用于通过 `orderByChild`、`orderByValue` 或 `orderByPriority` 中的任何一种进行排序的数据。


灵活组合使用这些事件对于监听数据库中某个特定节点将会非常有用。 例如，社交博客应用可以结合使用这些方法来监控博文评论中的活动，如下所示：

Objective-C

```
// Listen for new comments in the Wilddog database
[_commentsRef
 observeEventType:WEventTypeChildAdded
 withBlock:^(WDataSnapshot *snapshot) {
     [self.comments addObject:snapshot];
     [self.tableView insertRowsAtIndexPaths:@[[NSIndexPath indexPathForRow:[self.comments count] - 1 inSection:kSectionComments]] withRowAnimation:UITableViewRowAnimationAutomatic];
 }];
// Listen for deleted comments in the Wilddog database
[_commentsRef
 observeEventType:WEventTypeChildRemoved
 withBlock:^(WDataSnapshot *snapshot) {
     int index = [self indexOfMessage:snapshot];
     [self.comments removeObjectAtIndex:index];
     [self.tableView deleteRowsAtIndexPaths:@[[NSIndexPath indexPathForRow:index inSection:kSectionComments]]
                           withRowAnimation:UITableViewRowAnimationAutomatic];
 }];

```

Swift

```     
// Listen for new comments in the Wilddog database
commentsRef.observeEventType(.ChildAdded, withBlock: { (snapshot) -> Void in
      self.comments.append(snapshot)
      self.tableView.insertRowsAtIndexPaths([NSIndexPath(forRow: self.comments.count-1, inSection: kSectionComments)], withRowAnimation: UITableViewRowAnimation.Automatic)
})
// Listen for deleted comments in the Wilddog database
commentsRef.observeEventType(.ChildRemoved, withBlock: { (snapshot) -> Void in
      let index = self.indexOfMessage(snapshot)
      self.comments.removeAtIndex(index)
      self.tableView.deleteRowsAtIndexPaths([NSIndexPath(forRow: index, inSection: kSectionComments)], withRowAnimation: UITableViewRowAnimation.Automatic)
})

```

## 移除监听

当您退出 ViewController 时，观测程序不会自动停止同步数据。 如果未正确删除，观测程序会继续将数据同步到本地内存。 不再需要观测程序时，通过将关联的 `WilddogHandle` 传递给 `removeObserverWithHandle` 方法，即可将其删除。

将回调快添加到引用时，会返回 `WilddogHandle`。这些句柄可用于删除回调块的监听。

如果已将多个侦听器添加到数据库引用，则会在引发事件时调用每个侦听器。 要在该位置停止同步数据，必须通过调用 `removeAllObservers` 方法删除其中的所有观测程序 (`removeAllObservers`方法的作用是取消之前用 `observeEventType `方法在该节点注册的所有监听事件。)。

在父节点上调用 `removeObserverWithHandle` 或者 `removeAllObservers` 时不会删除在其子节点上注册的监听。您还必须跟踪这些引用或句柄才能将其删除。

## 一次性读取数据

在某些情况下，您可能希望只调用一次回调，如同调用一次 HTTP 请求一样。（例如，在初始化时，预期 UI 元素不会再发生更改时）。您可以使用 observeSingleEventOfType 方法简化这种情况：

添加的事件回调仅触发一次，以后不会再次触发。

对于只需加载一次且预计不会频繁变化，或是我们需要主动拉取的数据，这非常有用。 例如，上述示例中的博客应用使用此方法在用户开始撰写新博文时加载其个人资料：

Objective-C

```
NSString *userID = currentUser.uid;
[[[_ref childByAppendingPath:@"users"] childByAppendingPath:userID] observeSingleEventOfType:WEventTypeValue withBlock:^(WDataSnapshot * _Nonnull snapshot) {
    // Get user value
    User *user = [[User alloc] initWithUsername:snapshot.value[@"username"]];
    
    // ...
} withCancelBlock:^(NSError * _Nonnull error) {
    NSLog(@"%@", error.localizedDescription);
}];

```

Swift

```     
let userID = currentUser?.uid
ref.childByAppendingPath("users").childByAppendingPath(userID!).observeSingleEventOfType(.Value, withBlock: { (snapshot) in
    // Get user value
    let username = snapshot.value!["username"] as! String
    let user = User.init(username: username)
    
    // ...
}) { (error) in
    print(error.localizedDescription)
}

```

## 排序和查询数据

你可以使用 [Query](/docs/guide/database/ios/api.html#Query-Methods) 类 API 进行数据排序。Wilddog 支持按键、按值、按子节点的值或按优先级对数据进行排序。
只有在对数据排序之后，你才可以进行具体的查询操作，从而获取你想要的特定数据。

**注意**：排序和过滤的开销可能会很大，在客户端执行这些操作时尤其如此。 如果你的应用使用了查询，请定义 [.indexOn](api/.indexOn) 规则，以便在服务器上添加索引以提高查询性能。详细操作参见[添加索引](rule/indexOn)。

#### 数据排序

对数据排序前，要先指定按照`键`、`值`、`子节点的值`或按`优先级`这四种的哪一种排序。对应的方法如下：

方法 | 用法
----  | ----
queryOrderedByKey | 按子键对结果排序。
queryOrderedByValue | 按子值对结果排序。
queryOrderedByChild | 按指定子键的值对结果排序。
queryOrderedByPriority | 按节点的指定优先级对结果排序。

**注意**：每次只能使用一种排序方法。对同一查询调用多个排序方法会引发错误。作为一种变通的方法，你可以先按一种方式查询，然后自行在结果集中进行第二次查询。

以下示例演示了如何检索用户置顶博文（按星级数排序）的列表：
Objective-C

```
// My top posts by number of stars
WQuery *myTopPostsQuery = [[[self.ref childByAppendingPath:@"user-posts"]
                                      childByAppendingPath:[super getUid]]
                                     queryOrderedByChild:@"starCount"];

```

Swift

```     
// My top posts by number of stars
let myTopPostsQuery = (ref.childByAppendingPath("user-posts").childByAppendingPath(getUid())).queryOrderedByChild("starCount")

```

此查询根据用户 ID 从数据库中的路径检索用户博文，并按每篇博文获得的星级数排序。调用 `queryOrderedByChild` 方法可指定对结果排序所依据的子键。 在本例中，博文按各自 "starCount" 子节点的值进行排序。 如需了解有关如何对其他数据类型进行排序的详细信息，请参见[排序规则](#排序规则)。


#### 查询数据

只有对数据排序进行之后，才能查找数据，你可以结合使用以下方法来构造查找的条件。

方法 | 用法
---- | ----
queryLimitedToFirst | 设置从第一条开始，一共返回多少条数据（节点）。
queryLimitedToLast | 设置从最后一条开始，一共返回多少条（返回结果仍是升序，降序要自己处理）。
queryStartingAtValue | 返回大于或等于指定的键、值或优先级的数据，具体取决于所选的排序方法。
queryEndingAtValue | 返回小于或等于指定的键、值或优先级的数据，具体取决于所选的排序方法。
queryEqualToValue | 返回等于指定的键、值或优先级的数据，具体取决于所选的排序方法。可用于精确查询。

与排序依据方法不同，你可以结合使用这些过滤方法。例如，你可以结合使用 `queryStartingAtValue` 与 `queryEndingAtValue` 方法将结果限制在指定的范围内。

**限制结果数**

你可以使用 `queryLimitedToFirst` 和 `lqueryLimitedToLast` 方法为某个给定的事件设置要监听的子节点的最大数量。 例如，如果你使用 `queryLimitedToFirst` 将限制个数设置为 100，那么一开始最多只能收到 100 个 `WEventTypeChildAdded` 事件，即只返回前100条数据的快照。
当数据发生更改时，对于进入到前100的数据，你会接收到 `WEventTypeChildAdded` 回调，对于从前100中删除的数据，你才会接收到 `WEventTypeChildRemoved` 事件，也就是说只有这100条里的数据变化才会触发事件。

以下示例演示了示例博客应用如何按所有用户检索 100 篇最新博文的列表：

Objective-C

```
// Last 100 posts, these are automatically the 100 most recent
// due to sorting by push() keys
WQuery *recentPostsQuery = [[self.ref childByAppendingPath:@"posts"] queryLimitedToFirst:100];

```

Swift

```     
// Last 100 posts, these are automatically the 100 most recent
// due to sorting by push() keys
let recentPostsQuery = (ref?.childByAppendingPath("posts").queryLimitedToFirst(100))!

```

#### 数据排序
本小节介绍在使用各种排序方式时，数据究竟是如何排序的。

**queryOrderedByChild**

当使用`queryOrderedByChild:`时，按照子节点的公有属性 key 的 value 进行排序。仅当 value 为单一的数据类型时，排序有意义。如果 key 属性有多种数据类型时，排序不固定，此时不建议使用`queryOrderedByChild:`获取全量数据，例如，

```json
{
  "scores": {
    "no1" : {
        "name" : "tyrannosaurus",
        "score" : "120"
    },
    "no2" : {
        "name" : "bruhathkayosaurus",
        "score" : 55
    },
    "no3" : {
        "name" : "lambeosaurus",
        "score" : 21
    },
    "no4" : {
        "name" : "linhenykus",
        "score" : 80
    }, 
    "no5" : {
        "name" : "pterodactyl",
        "score" : 93
    }, 
    "no6" : {
        "name" : "stegosaurus",
        "score" : 5
    }, 
    "no7" : {
        "name" : "triceratops",
        "score" : 22
    }, 
    "no8" : {
        "name" : "brontosaurus",
        "score" : true
    }
  }
}

```

霸王龙的分数是`NString`类型，雷龙的分数是`BOOL`类型，而其他恐龙的分数是`NSNumber`类型，此时使用`queryOrderedByChild:`获得全量数据时，是一个看似固定的排序结果；但是配合使用`queryLimitedToFirst:`时，将获得不确定的结果。`NSObject`类型数据的 value 值为 null，不会出现在结果中。
当配合使用`queryStartingAtValue:`、`queryEndingAtValue:`和`queryEqualToValue:`时，如果子节点的公有属性 key 包含多种数据类型，将按照这些函数的参数的类型排序，即只能返回这个类型的有序数据。上面的数据如果使用 `[[[ref queryOrderedByChild:@"score"]queryStartingAtValue:@60]queryLimitedToFirst:4]` 将得到下面的结果：

```json
{
   "no4" : {
       "name" : "linhenykus",
       "score" : 80
   },
   "no5" : {
       "name" : "pterodactyl",
       "score" : 93
   }
}
  
```

<p style='color:red'><em>注意：如果 path 与 value 的总长度超过1000字节时，使用 `queryOrderedByChild:` 将搜索不到该数据。</em></p>

**queryOrderedByKey**

当使用`queryOrderedByKey`对数据进行排序时，数据将会按照下面的规则，以字段名升序排列返回。注意，节点名只能是字符串类型。

1, 节点名能转换为32-bit整数的子节点优先，按数值型升序排列。

2, 接下来是字符串类型的节点名，按字典序排列。

**queryOrderedByValue**

当使用`queryOrderedByValue`时，按照直接子节点的 value 进行排序。仅当 value 为单一的数据类型时，排序有意义。如果子节点包含多种数据类型时，排序不固定，此时不建议使用`queryOrderedByValue`获取全量数据，例如，

```json
{
  "scores": {
    "tyrannosaurus" : "120",
    "bruhathkayosaurus" : 55,
    "lambeosaurus" : 21,
    "linhenykus" : 80,
    "pterodactyl" : 93,
    "stegosaurus" : 5,
    "triceratops" : 22,
    "brontosaurus" : true
  }
}

```

霸王龙的分数是 `NSString`类型，雷龙的分数是 `BOOL` 类型，而其他恐龙的分数是 `NSNumber` 类型，此时使用 `queryOrderedByValue` 获得全量数据时，是一个看似固定的排序结果；但是配合使用`queryLimitedToFirst:`时，将获得不确定的结果。`NSObject`类型数据的 value 值为 null，不会出现在结果中。
当配合使用`queryStartingAtValue:`、`queryEndingAtValue:`和`queryEqualToValue:`时，如果子节点的 value 包含多种数据类型，将按照这些函数的参数的类型排序，即只能返回这个类型的有序数据。上面的数据如果使用`[[[ref queryOrderedByValue]queryStartingAtValue:@60]queryLimitedToFirst:4]`将得到下面的结果：
```json
{
    "linhenykus" : 80,
    "pterodactyl" : 93
}
```
<p style='color:red'><em>注意：如果 path 与 value 的总长度超过1000字节时，使用`queryOrderedByValue:`将搜索不到该数据。</em></p>

**queryOrderedByPriority**

当使用`queryOrderedByPriority`对数据进行排序时，子节点数据将按照优先级和字段名进行排序。注意，优先级的值只能是数值型或字符串。

1, 没有优先级的数据（默认）优先。

2, 接下来是优先级为数值型的子节点。它们按照优先级数值排序，由小到大。

3, 接下来是优先级为字符串的子节点。它们按照优先级的字典序排列。

4, 当多个子节点拥有相同的优先级时（包括没有优先级的情况），它们按照节点名排序。节点名可以转换为数值类型的子节点优先（数值排序），接下来是剩余的子节点（字典序排列）。



