title: 规则表达式
---

## 简介

Wilddog非常重视数据的安全问题。在过去的开发过程中，数据的安全问题，一直令开发者头疼，不仅功能不全而且配置复杂。

Wilddog提供了一种类 javascript语法的规则表达式来解决你的安全问题。

规则表达式符合 JSON规范，你可以通过编写少量的规则表达式实现强大的安全功能，包括设置读写权限，用户授权，数据校验和数据索引。

规则是声明式的，即独立于主产品逻辑之外。这样是有好处的：安全性不完全依赖于客户端，无需中间服务器即可保护数据安全，真正解决无后端应用中的安全问题。

你可以在控制台-实时数据库-读写权限中配置规则表达式。

![img](https://dn-shimo-image.qbox.me/dyDkf9I24SM0anKa.png!thumbnail)

规则表达式有四种类型，读、写以及校验，我们能够通过灵活配置这几个个规则，实现访问权限控制和数据校验还有建立索引。

| 规则类型      | 描述                             |
| --------- | ------------------------------ |
| .read     | 定义了数据是否可以被用户读取                 |
| .write    | 定义了数据是否可以被用户写入                 |
| .validate | 定义了什么样的数据是正确的格式，是否有某些子属性，数据类型等 |
| .indexOn  | 为节点数据建立索引，提高查询效率               |



规则类型描述.read定义了数据是否可以被用户读取.write定义了数据是否可以被用户写入.validate定义了什么样的数据是正确的格式，是否有某些子属性，数据类型等.indexon为节点数据建立索引，提高查询效率

通过配置规则表达式，可以很轻松的实现各种安全功能：

### 设置读写权限

.read .write 你可以设置你数据中任意节点的可读和可写性。



```javascript
{
  "rules": {
    "foo": {
      ".read": true,
      ".write": false
    }
  }
}
```



使用.read 与 .write 可以声明了/foo节点以及子节点的读写权限，true为允许，false 为禁止。

true与 false 可以通过表达式来表示。

### 用户认证

内置的 auth对象，你可以授权给不同的用户，只允许授权的用户进行数据操作。



```javascript
{
  "rules": {
    "users": {
      "$uid": {
        ".write": "$uid === auth.uid"
      }
    }
  }
}
```



通过使用 auth对象，只允许允许写入 uid，

### 校验数据

.validate 你可以校验数据的合法性，只允许符合规则的数据存入。



```javascript
{
  "rules": {
    "foo": {
      ".validate": "newData.isString() && newData.val().length < 100"
    }
  }
}
```





使用.validate 表达式使 foo节点只允许通过使用字符串类型，并且只允许存入长度<100的值。

### 数据索引

通过.indexOn，你可以针对数据节点进行索引，提高查询数据的速度。



```javascript
{
  "rules": {
    "dinosaurs": {
      ".indexOn": ["height", "length"]
    }
  }
}
```



使用了.indexOn对 student的子节点 age，score进行索引，进行查询的时候将会提高效率。

## 安全表达式基础知识

#### 1.设置节点的读取权限

```javascript
"rules": {
  "message"{
    "message1"{    
      ".read": true,
      ".write":  true,
      ".validate": true
    }
    "message2"{
      ".read": true,
      ".write": false,
      ".validate": false
    }
  }
}
```

message1节点，所有人都可以读取数据，所有人都可以输入数据，所有类型的数据都能够输入。

message2节点，所有人都可以读取数据，所有人都不能输入数据，任何数据类型都不能输入。

规则表达式使用 JSON结构，规则表达式的层级结构应该与数据库一致。

注意：应用创建之后，系统默认为公开。所有人都能读写，为了你的数据安全，尽快配置规则表达式

#### 2.用$通配符选择子节点

实际生产环境中，不可能为每一个数据节点单独配置规则，因此我们可以使用 $ 通配符来选择子节点。

例如你储存了一个 message 的信息内容列表：

```javascript
{
  "messages": {
    "message0": {
      "content": "Hello",
      "timestamp": 1405704370369,
      "uid":17343512
    },
    "message1": {
      "content": "Goodbye",
      "timestamp": 1405704395231,
      "uid":72366233
    },
    ...
  }
}
```

不可能为每一个 message 配置规则。那么你可以使用$通配节点。例如

```javascript
{
  "rules": {
    "messages": {
      "$message_content": {
        ".read": true,
        ".write": true
      }
      "$message_timestamp": {
        ".read": false
        ".write": false
      }
    }
  }
}
```

还能够使用 $other 选择未列出的子节点：

```javascript
{
  "rules": {
    "messages": {
      "$message_content": {
        ".read": true,
        ".write": true
      }
      "$other": {
        ".read": false
        ".write": false
      }
    }
  }
}
```

除了content 可以读写之外，messages下的其他所有节点都不可读取。

### 3.使用内置对象与内置函数

规则表达式中提供了许多内置对象，利用这些内置对象提供的函数和属性，你可以配置各种规则。例如可以使用 auth对象进行授权:

```javascript
{
  "rules": {
    "messages": {
      "$messages_timestamp": {
        ".read": "$messages_uid == auth.uid"
        ".write": "$messages_uid == auth.uid"
      }
    }
  }
}
```

使用 auth对象能够迅速配置，只有当前用户才能够读写自己的数据的权限设置

auth 代表已经登录的用户对象，

规则表达式采用了一种类 javascript的语法，能够在表达式中使用内置对象和内置函数，最终返回的结果应该是 "true"或者"false"。

**内置对象**

内置对象描述now云端的时间戳，以毫秒为单位。root类型的对象，代表操作之前，数据根节点newData类型的对象，代表假设数据操作成功，之后此节点下的数据，也就是此节点下的旧数据和本次操作写入的新数据合并之后的数据。data类型的对象，代表此节点被操作前的原始数据。authAuth类型对象，代表已经登录用户对象。$variables节点的名称变量，代表动态路径的通配符。

还有许多内置函数和运算符，可以查看 [规则表达式API 文档](http:/#)

**data与 newData**

内置对象data指的是写操作发生之前，当前节点的已有数据。newData则指的是，假定写操作会成功，那么写操作成功之后当前节点的数据。newData代表的是旧数据和即将写入的数据合并之后的数据。

为了演示它们的用法，考虑一个这样的场景：我们需要一个规则，在指定的路径下，当前节点数据不存在的时候可以写入，也可以删除已存在的数据，但不能对已有数据进行修改：

```javascript
// data不存（没有旧数据）在或者newData不存在（删除数据）的情况下，可以写入。
// 也就是说可以是全新写入，也可以是删除，但是不能修改已有数据。
".write": "!data.exists() || !newData.exists()"
```

**使用 root 引用其他路径(Demo 需要修改)**

通过使用内置变量root，data和newData，你可以访问到任何数据节点。

参考下面的例子，只有当allow_writes节点的值为true，且父节点没有设置一个readOnly标志，且新写入的数据中存在名为message的子节点时，数据才被允许写入。

```javascript
{
  ".write": "root.child('allow_writes').val() == true &&
            !data.parent().child('readOnly').exists() &&
            newData.child('message').exists()"
}
```

通过内置变量root,，$通配符，您可以定位并且选择任何数据节点。



### 4.规则的级联

规则表达式遵循一个自上向下延展的原则。如果一个数据节点上的.write或.read规则赋予了读或写的权限，那么它的所有的子节点也都将拥有读或写的权限。子节点上的规则不能覆盖父节点或祖先节点已经赋予的读或写的权限。参考下面的例子

```javascript
 "rules": {
     "message": {
        // 允许/message/节点下的数据被读取
        ".read": "data.child('content').val() == true",
        "content": {
          // 当父节点的表达式授予了读权限时，这一规则设置false无效。
          ".read": false
        }
     }
  }
}
```

例如上面的例子，message节点已经设置了可读，那么无论子节点如何设置，子节点都是可读的。



### 5.规则的原子性

为了保证数据的安全，在一次操作中，只要任何一个数据节点没有访问权限，那么整个操作将会失败。参考这个例子：



```javascript
{
  "rules": {
    "messages": {
      "message1": {
        ".read": true
      },
      "message2": {
        ".read": false
      }
    }
  }
}
```

读取路径/messages会返回错误，即使 message1是可读的，但是因为 message2没有可读的权限，所以整个操作都会失败。

## 数据校验

可以通过.validate 对数据进行校验。

```javascript
{
  "messages:"{
      "rules": {
            "content": {
              ".read": true,
              ".write": true,
              // 写入/foo 的数据必须是字符串类型且长度小于100。
              ".validate": "newData.isString() && newData.val().length() < 100"
            }
       }
   }
}
```

内置变量自带了很多判断函数，例如 isString(), lenght()等。更多的判断函数可以参考 [规则表达式API 文档](http:/#)

**.validate规则不会向下级联。如果在一次操作中，任何一个子节点的.validate规则失败，整个写操作都将失败。当数据被删除（也就是值为null）时，.validate表达式被忽略。**

### 使用正则表达式校验数据

我们可以使用内置的 match()函数来进行正则校验

数据需要满足是字符串，并且字符串是1900-2099年间的YYYY-MM-DD格式

1. ".validate": "newData.isString() &&  newData.val().matches(/^(19|20)[0-9][0-9][-\\/. ](0[1-9]|1[012])[-\\/. ](0[1-9]|[12][0-9]|3[01])$/)"

Wilddog只支持一部分正则表达式的功能，已经能够满足绝大部分的需求，关于正则表达式校验的更多内容请看[正则表达式校验](http:/#)。



```javascript
".validate": "newData.isString() &&  newData.val().matches(/^(19|20)[0-9][0-9][-\\/. ](0[1-9]|1[012])[-\\/. ](0[1-9]|[12][0-9]|3[01])$/)"
```



### 示例介绍

我们来看一个更复杂的例子来了解校验规则的使用

匿名聊天室的例子(可能再换一个例子)

```javascript
{
  "rules": {
    // 默认规则为 false
    // 在这里的设置会让所有的子节点设置都受影响(规则的级联)
    // ".read": false,
    // ".write": false,

    "room_names": {
      // the room names can be enumerated and read
      // 没有可写规则，所以他是不可写的
      // 这里只明确了可以读取
      ".read": true,

      "$room_id": {
        // this is just for documenting the structure of rooms, since
        // they are read-only and no write rule allows this to be set
        ".validate": "newData.isString()"
      }
    },

    "messages": {
      "$room_id": {
        // the list of messages in a room can be enumerated and each
        // message could also be read individually, the list of messages
        // for a room cannot be written to in bulk
        ".read": true,

        // room we want to write a message to must be valid
        ".validate": "root.child('room_names/'+$room_id).exists()",

        "$message_id": {
          // a new message can be created if it does not exist, but it
          // cannot be modified or deleted
          ".write": "!data.exists() && newData.exists()",
          // the room attribute must be a valid key in room_names/ (the room must exist)
          // the object to write must have a name, message, and timestamp
          ".validate": "newData.hasChildren(['name', 'message', 'timestamp'])",

          // the name must be a string, longer than 0 chars, and less than 20 and cannot contain "admin"
          "name": { ".validate": "newData.isString() && newData.val().length > 0 && newData.val().length < 20 && !newData.val().contains('admin')" },

          // the message must be longer than 0 chars and less than 50
          "message": { ".validate": "newData.isString() && newData.val().length > 0 && newData.val().length < 50" },

          // messages cannot be added in the past or the future
          // clients should use firebase.database.ServerValue.TIMESTAMP
          // to ensure accurate timestamps
          "timestamp": { ".validate": "newData.val() <= now" },

          // no other fields can be included in a message
          "$other": { ".validate": false }
        }
      }
    }
  }
}
```

分析校验规则单独分析、拓展

```

```

```

```



## 用户认证

### 基本认证

![img](https://dn-shimo-image.qbox.me/MkrqEWKK7dwefEWW.png!thumbnail)

auth

有关 auth的更多内容，可以使用我们的 [auth](http:/#)功能

**自定义 Token**

超级秘钥

野狗的每个app都有自己的超级密钥，用于在集成用户已有的终端用户系统，为生成的token作签名校验，以保障安全。也可用于用户的服务端到野狗云的超级权限认证（使用超级密钥直接认证的终端为超级权限，不受规则表达式的限制, 超级密钥列表中只有第一个能生成合法的 Wilddog Token ）。

有关 auth的更多内容，可以使用我们的 [auth](http://http/#)功能



## 数据索引

Wilddog提供了强大高效的查询方法，你可以按照任意子节点的顺序对数据进行查询。但是当你的数据量不断增加的时候，查询的效率也会逐渐降低。WildDog提供了数据索引来解决这个问题。你可以在规则表达式中使用 .indexOn 规则为节点建立索引，提高查询效率。

**节点的名称key和优先级priority默认建立索引，不需要额外设置**

Wilddog 根据查询需求的不同，提供了两种.indexOn的索引方式，orderByChild和 orderByValue。

### orderByChild 根据子节点索引

orderByChild 可以根据不同的子节点进行索引，例如我们现在有一些用户的信息：

```javascript
{
  "Jack": {
    "age" : 21,
    "score" : 88,
    "weight": 63
  },
  "Lucy": {
    "age" : 22,
    "score" : 91,
    "weight" : 49
  }
}
```

假设在应用中，我们需要经常对用户的信息按照名称(key)、年龄(score)、分数(score)进行排序，但是不会按照体重(weight)排序，这样我们就可以通过Wilddog提供的.indexOn规则，为名称(key)、年龄(age)、分数(score)这些节点建立索引，来提高查询效率。

我们可以使用如下的设置为高度、长度节点建立索引。名称即为节点的key值，已经默认建立了索引，所以不需要额外设置。

```javascript
{
  "rules": {
    "dinosaurs": {
      ".indexOn": ["age", "score"]
    }
  }
}
```

### orderByValue 根据值索引

orderByValue 可以根据 value的值进行索引。例如我们需要为学生分数建立一个排行榜：

```javascript
{
  "scores": {
    "Jack" : 55,
    "Lucy" : 81,
    "LiLei" : 80,
    "HanMeimei" : 93,
    "Michael" : 66,
    "Jane" : 78
  }
}
```

我们只需要对节点的 value 进行排序，因此我们可以在 /scores节点下对 value 进行索引：

```javascript
{
  "rules": {
    "scores": {
      ".indexOn": ".value"
    }
  }
}
```



### 最后留几个作业？



## 



# API 文档

内容基本不变，但是需要在样式上修改细节

重新命名，修改细节文案。至少得提供中文吧？

重新考虑布局

可以采取表格的形式

