actvitystream
=============

引用维基百科对Activity Streams协议的介绍
> **Activity Streams** is an open format specification for activity stream protocols, which are used to syndicate activities taken in social web applications and services, similar to those in Facebook[1]'s Newsfeed, FriendFeed, the Movable Type Action Streams plugin, etc.

一个活动流是指用户最近执行地一系列的活动。比如在豆瓣发了一篇日记、一个广播等等都是一个用户活动。另一个比较典型的应用场景是QQ空间，当你特别关注了某一位好友，在该好友发了一条说说或照片，你都将收到消息提醒。

因此，这个项目是对此协议的部分实现。服务器端使用NodeJs+Mongodb，之所以选用这两项技术主要是看中了JSON的灵活性，以及NodeJs开发Restful API的快速，同时还可以使用socket.io这项技术实现实时消息提醒功能。

客户端使用Java，用于生成符合协议的json内容。目前只做了部分实现，一般是应用于业务系统。

### 协议中的部分节点介绍

Activity Streams entry 元素总是包含以下元素，其中1,2,3为主元素，它们表示：

1. actor：执行动作的人（例：John Doe）
2. verb：执行的动作类型（例：added）
3. object：动作所针对的项目（例：Ben's photo）
4. target: 这是执行或放置动作的容器对象（例如：动作 John Doe added Ben's photo to John's photo gallery 中的 John's photo gallery）

详细内容请查看[Activity Schema](http://activitystrea.ms/specs/json/schema/activity-schema.html)

## MongoDB集合字段说明

### ActivityStream集合

* _id：activity唯一标识符；
* pushTime：activity发布时间；
* status：activity状态，如0表示未读，1表示已读；
* activity：activity对象，该对象为发布activity时activity参数值，该节点是灵活的，结构完全取决与push时提供的数据，这需要服务端和客户端协调格式；

## 消息实时订阅

主要以web端举例

```javascript
<script src="/socket.io/socket.io.js"></script>
<script>
  var socket = io('http://localhost:3000/');
  socket.on('connect', function () {
    socket.send({
		"logicId": "login", //向服务器注册当前用户，否则无法接收到发送给该用户的消息
		"username": "s1001" //用户id, 必须是用户的唯一标识
	});

    socket.on('activity', function (msg) { //监听消息
      // my msg
    });
  });
</script>
```

msg的消息一般有两种, 第一种是向服务器发送注册请求后, 如果有未读消息，服务器将回发未读消息数的消息, 如:

```json
{
	"type": "count",
	"count: 2,
	"message": "您有2条未读消息"
}
```

第二种是消息实体, 如:

```json
{
    "type": "new",
    "activity": {
        "actor": {
            "displayName": "jack",
            "id": "s1000",
            "objectType": "person"
        },
        "object": {
            "displayName": "rose",
            "id": "t0001",
            "objectType": "person"
        }, 
        "objectType": "activity",
        "target": {
            "displayName": "绘画小组",
            "id": "6673890",
            "objectType": "group"
        },
        "verb": "request"
    },
    "message": "您有1条新消息"
}
```

## HTTP API接口

### 创建一条活动流

#### URL
	/api/activity

#### METHOD
	POST(必须是POST)

#### 参数

* **activity**：符合协议的json内容(必需项)

举个例子，名字为jack的用户请求加入绘画小组，而群组管理员是rose，那么需要将加群请求发送给rose。url就是:
```
http://192.168.7.50:3001/api/activity?activity={"actor":{"displayName":"jack","id":"s1000","objectType":"person"},"object":{"displayName":"rose","id":"t0001","objectType":"person"},"objectType":"activity","target":{"displayName":"绘画小组","id":"6673890","objectType":"group"},"verb":"request"}
```

#### 响应结果说明

**节点说明**：

* code：表示响应结果状态，1表示成功，0表示失败
* message：响应结果的文字说明

##### 简要示例

```json
{
  "code": 1,
  "message": "succes"
}
```

### 更新活动状态

#### URL
	/api/activity/update

#### METHOD
	POST(必须是POST)

#### 参数

* **id**：活动流的id(必需项)
* **status**：活动流的状态，0表示未读，1表示已读(必需项)

#### 响应结果说明

**节点说明**：

* code：表示响应结果状态，1表示成功，0表示失败
* message：响应结果的文字说明

##### 简要示例

```json
{
  "code": 1,
  "message": "succes"
}
```

### 查询

#### URL
	/api/activity

#### METHOD
	GET(必须是GET)

#### 参数

* **mode**: 查询方式，可选值为all,one和page, 默认值为page（all： 查询出所有符合条件的activity，one：只查询出一条符合条件的activity，page：分页查询）；
* **asc**: pushTime字段排序方式，可选值（true：升序，false：降序<此为默认值>）；
* **id**: 唯一标识值，可以配合mode参数（设为one）作精确查询；
* **currpage**: 查询页码（默认为1），mode参数值为page时有效；
* **pagesize**: 每页查询数据条数（默认为8），mode参数值为page时有效；

##### 高级查询

高级查询部分的参数非常灵活，并且数量和名称可变，完全根据需求查询。
参数名称为字典式查询，如查询所有发送给id为6673890的用户的加群请求：

```
http://192.168.7.50:3001/api/activity?activity.verb=request&activity.target.objectType=group&activity.target.id=6673890&mode=all
```

也就是说以activity开头的参数名完全取决于push activity时传递的activity参数值结构。


#### 响应结果说明

**节点说明**：

* code：表示响应结果状态，1表示成功，0表示失败
* message：响应结果的文字说明
* count：符合查询条件的总记录数, 与data节点一同出现，当mode参数等于all或page时将有此节点；
* data：ActivityStream数组，当mode参数等于all或page时将有此节点；
* datum：单个ActivityStream节点，当mode参数等于one时将有此节点；

#### 简要示例

##### 精确查询指定的一条activity数据

```json
{
    "code": 1,
    "message": "成功查询到数据",
    "datum": {
        "_id": "afd123",
        "pushTime": 1424530631,
        "status": 1,
        "activity": {
            "actor": {
                "displayName": "jack",
                "id": "s1000",
                "objectType": "person"
            },
            "object": {
                "displayName": "rose",
                "id": "t0001",
                "objectType": "person"
            },
            "objectType": "activity",
            "target": {
                "displayName": "绘画小组",
                "id": "6673890",
                "objectType": "group"
            },
            "verb": "request"
        }
    }
}
```

#### 查询列表数据

```json
{
    "code": 1,
    "message": "成功查询到数据",
    "data": [{
		"_id": "afd123",
		"pushTime": 1424530631,
		"status": 1,
		"activity": {
		"actor": {
			"displayName": "jack",
			"id": "s1000",
			"objectType": "person"
		},
		"object": {
			"displayName": "rose",
			"id": "t0001",
			"objectType": "person"
		},
		"objectType": "activity",
		"target": {
			"displayName": "绘画小组",
			"id": "6673890",
			"objectType": "group"
		},
		"verb": "request"
		}
	}]
}
```

## Copyright & License

Copyright (c) 2015  Released under the [MIT license](LICENSE).
