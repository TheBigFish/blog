# qwebchannel.js 源码分析

QT5.9.1
chatserver-cpp
chatclient-html

## 流程分析

### 初始化阶段

1. 上传 init

```json
{ "type": 3, "id": 0 }
```

2. 下发初始化信息

```json
{
  "data": {
    "chatserver": {
      "methods": [
        ["deleteLater", 3],
        ["login", 11],
        ["logout", 12],
        ["sendMessage", 13],
        ["keepAliveResponse", 14]
      ],
      "properties": [
        [0, "objectName", [1, 2], ""],
        [1, "userList", [1, 7], []]
      ],
      "signals": [
        ["destroyed", 0],
        ["newMessage", 5],
        ["keepAlive", 6],
        ["userCountChanged", 8]
      ]
    }
  },
  "id": 0,
  "type": 10
}
```

3. 上送连接信号（5 - newMessage）

```json
{ "type": 7, "object": "chatserver", "signal": 5 }
```

4. 上送连接信号（6 - keepAlive）

```json
{ "type": 7, "object": "chatserver", "signal": 6 }
```

注意，属性 userList 对应的信号 userListChanged 信号不会上送连接状态
代码中 isPropertyNotifySignal 被设置为 True。
只要服务端有属性改变，改变的属性连同属性改变信号同时发送下来。

5. 上送空闲

```json
{ "type": 4 }
```

---

至此初始化完成

### 运行阶段

1. 服务端发送信号(keepAlive)

```json
{ "object": "chatserver", "signal": 6, "type": 1 }
```

2. 用户登录，客户端调用方法(login)

```json
{ "type": 6, "object": "chatserver", "method": 11, "args": ["a"], "id": 1 }
```

3. 服务端回复(注意 id 与方法调用的 id 一致)

```json
{ "data": true, "id": 1, "type": 10 }
```

4. 服务端更新属性(1 - userList)，同时出发信号(7 - userListChanged)

```json
{
  "data": [
    {
      "object": "chatserver",
      "properties": { "1": ["a"] },
      "signals": { "7": [] }
    }
  ],
  "type": 2
}
```

见 html 代码

```js
channel.objects.chatserver.userListChanged.connect(function() {
  $("#userlist").empty();
  //access the property
  channel.objects.chatserver.userList.forEach(function(user) {
    $("#userlist").append(user + "<br>");
  });
});
```

5. 上送心跳连接(14 - keepAliveResponse)

```json
{ "type": 6, "object": "chatserver", "method": 14, "args": ["a"], "id": 2 }
```

6. 服务端回复心跳连接

```json
{ "data": null, "id": 2, "type": 10 }
```

7. 上送聊天信息(13-sendMessage id - 3)

```json
{
  "type": 6,
  "object": "chatserver",
  "method": 13,
  "args": ["a", "hello"],
  "id": 3
}
```

8. 服务端广播聊天信息(5-newMessage)

```json
{
  "args": ["14:46:32", "a", "hello"],
  "object": "chatserver",
  "signal": 5,
  "type": 1
}
```

9. 服务端回复上送聊天信息(id - 3)

```json
{ "data": true, "id": 3, "type": 10 }
```

几个细节：

- 客户端调用方法（invokeMethod）,附带 id，服务端执行方法后返回 response，对应该 id 信息
- 属性对应一个属性改变的信号，服务端改变属性时会将属性值及对应信号同时下发
- 客户端注册属性改变信号回调时，不会给服务端上送注册信息（connectToSignal）

## 客户端源码

仅仅保留 js
客户端做了下列事情：

- 注册 userListChanged 事件回调
  userList 改变时，服务端会下发 userList 属性改变数据包，qwebchannel.js 会更新 objects.chatserver.userList 属性，同时包内包含 userListChanged 信号。
  导致回调函数执行，删除页面的用户列表并重新刷新。

- 注册 newMessage 回调
  当服务端广播消息时，在网页显示消息发送时间、发送者、消息内容

- 注册 keepAlive 回调
  当收到服务端心跳信号时，调用 keepAliveResponse 方法进行响应

- 当有用户登录时，调用 login 方法上送登录信息，同时注册响应回调，判断服务端的返回值

- 调用 sendMessage 发送用户信息，不关心返回，不注册响应回调

```html
<!DOCTYPE html>
<html>
  <head>
    <script>
      "use strict";
      var wsUri = "ws://localhost:12345";
      window.loggedin = false;

      window.onload = function() {
        var socket = new WebSocket(wsUri);

        socket.onclose = function() {
          console.error("web channel closed");
        };
        socket.onerror = function(error) {
          console.error("web channel error: " + error);
        };
        socket.onopen = function() {
          window.channel = new QWebChannel(socket, function(channel) {
            //connect to the changed signal of a property
            channel.objects.chatserver.userListChanged.connect(function() {
              $("#userlist").empty();
              //access the property
              channel.objects.chatserver.userList.forEach(function(user) {
                $("#userlist").append(user + "<br>");
              });
            });
            //connect to a signal
            channel.objects.chatserver.newMessage.connect(function(
              time,
              user,
              message
            ) {
              $("#chat").append(
                "[" + time + "] " + user + ": " + message + "<br>"
              );
            });
            //connect to a signal
            channel.objects.chatserver.keepAlive.connect(function(args) {
              if (window.loggedin) {
                //call a method
                channel.objects.chatserver.keepAliveResponse(
                  $("#loginname").val()
                );
                console.log("sent alive");
              }
            });
          });
        };
      };
    </script>
  </head>
  <body>
    <script>
      $("#loginForm").submit(submitForm);
      function submitForm(event) {
        console.log("DEBUG login: " + channel);
        channel.objects.chatserver.login($("#loginname").val(), function(arg) {
          console.log("DEBUG login response: " + arg);
          if (arg === true) {
            $("#loginError").hide();
            $("#loginDialog").dialog("close");
            window.loggedin = true;
          } else {
            $("#loginError").show();
          }
        });
        console.log($("#loginname").val());
        if (event !== undefined) event.preventDefault();
        return false;
      }
    </script>

    <script>
      $("#messageForm").submit(submitMessage);
      function submitMessage(event) {
        channel.objects.chatserver.sendMessage(
          $("#loginname").val(),
          $("#message").val()
        );
        $("#message").val("");
        if (event !== undefined) event.preventDefault();
        return false;
      }
    </script>
  </body>
</html>
```

## 服务端源码

信号：
void newMessage(QString time, QString user, QString msg);
void keepAlive();
void userListChanged();
void userCountChanged();

属性：
Q_PROPERTY(QStringList userList READ userList NOTIFY userListChanged)
QStringList userList() const;

方法：
Q_INVOKABLE bool login(const QString& userName);
Q_INVOKABLE bool logout(const QString& userName);
Q_INVOKABLE bool sendMessage(const QString& user, const QString& msg);
Q_INVOKABLE void keepAliveResponse(const QString& user);

注意 js 和 C++ 传输对应的参数类型

keepAliveResponse 返回值为 void, 对应 js 收到 data 为 null

```C
#ifndef ChatServer_H
#define ChatServer_H

#include <QObject>
#include <QStringList>

QT_BEGIN_NAMESPACE

class QTimer;

class ChatServer : public QObject
{
    Q_OBJECT

    Q_PROPERTY(QStringList userList READ userList NOTIFY userListChanged)

public:
    explicit ChatServer(QObject *parent = 0);
    virtual ~ChatServer();

public:
    //a user logs in with the given username
    Q_INVOKABLE bool login(const QString& userName);

    //the user logs out, will be removed from userlist immediately
    Q_INVOKABLE bool logout(const QString& userName);

    //a user sends a message to all other users
    Q_INVOKABLE bool sendMessage(const QString& user, const QString& msg);

    //response of the keep alive signal from a client.
    // This is used to detect disconnects.
    Q_INVOKABLE void keepAliveResponse(const QString& user);

    QStringList userList() const;

protected slots:
    void sendKeepAlive();
    void checkKeepAliveResponses();

signals:
    void newMessage(QString time, QString user, QString msg);
    void keepAlive();
    void userListChanged();
    void userCountChanged();

private:
    QStringList m_userList;
    QStringList m_stillAliveUsers;
    QTimer* m_keepAliveCheckTimer;
};

QT_END_NAMESPACE

#endif // ChatServer_H

```

## qwebchannel 源码

```js
"use strict";

var QWebChannelMessageTypes = {
  signal: 1,
  propertyUpdate: 2,
  init: 3,
  idle: 4,
  debug: 5,
  invokeMethod: 6,
  connectToSignal: 7,
  disconnectFromSignal: 8,
  setProperty: 9,
  response: 10
};

// transport 为 websocket 对象、
// initCallback 为连接建立，方法、属性、信号 初始化后的回调函数
var QWebChannel = function(transport, initCallback) {
  if (typeof transport !== "object" || typeof transport.send !== "function") {
    console.error(
      "The QWebChannel expects a transport object with a send function and onmessage callback property." +
        " Given is: transport: " +
        typeof transport +
        ", transport.send: " +
        typeof transport.send
    );
    return;
  }

  var channel = this;
  this.transport = transport;

  // websocket 发送包数据
  this.send = function(data) {
    if (typeof data !== "string") {
      data = JSON.stringify(data);
    }
    channel.transport.send(data);
  };

  // 给 websocket 注册消息回调
  // 处理服务端发来的 信号、方法响应、属性更新三类消息
  this.transport.onmessage = function(message) {
    var data = message.data;
    if (typeof data === "string") {
      data = JSON.parse(data);
    }
    switch (data.type) {
      case QWebChannelMessageTypes.signal:
        channel.handleSignal(data);
        break;
      case QWebChannelMessageTypes.response:
        channel.handleResponse(data);
        break;
      case QWebChannelMessageTypes.propertyUpdate:
        channel.handlePropertyUpdate(data);
        break;
      default:
        console.error("invalid message received:", message.data);
        break;
    }
  };

  // 客户端执行服务端方法
  this.execCallbacks = {};
  this.execId = 0;
  // 如果没有设置回调，则直接调用 send 发送数据
  // 如果有返回回调，则给发送数据增加 id,
  this.exec = function(data, callback) {
    if (!callback) {
      // if no callback is given, send directly
      // 方法初始化时， addMethod 会为每一个方法都注册一个默认的回调函数
      channel.send(data);
      return;
    }
    if (channel.execId === Number.MAX_VALUE) {
      // wrap
      channel.execId = Number.MIN_VALUE;
    }
    // 待发送数据不能有 id 属性，因为 id 是方法调用包的唯一标致
    if (data.hasOwnProperty("id")) {
      console.error(
        "Cannot exec message with property id: " + JSON.stringify(data)
      );
      return;
    }
    // 增加包 id 并发送数据
    data.id = channel.execId++;
    channel.execCallbacks[data.id] = callback;
    channel.send(data);
  };

  // 存储对象
  // 一个 C++ class 对应一个 object
  this.objects = {};

  // 收到服务端信号时触发信号回调
  this.handleSignal = function(message) {
    var object = channel.objects[message.object];
    if (object) {
      object.signalEmitted(message.signal, message.args);
    } else {
      console.warn(
        "Unhandled signal: " + message.object + "::" + message.signal
      );
    }
  };

  // 根据 id 对注册的响应回调函数进行调用
  this.handleResponse = function(message) {
    if (!message.hasOwnProperty("id")) {
      console.error(
        "Invalid response message received: ",
        JSON.stringify(message)
      );
      return;
    }
    channel.execCallbacks[message.id](message.data);
    delete channel.execCallbacks[message.id];
  };

  // 响应 属性更新
  this.handlePropertyUpdate = function(message) {
    for (var i in message.data) {
      var data = message.data[i];
      var object = channel.objects[data.object];
      if (object) {
        object.propertyUpdate(data.signals, data.properties);
      } else {
        console.warn(
          "Unhandled property update: " + data.object + "::" + data.signal
        );
      }
    }
    channel.exec({ type: QWebChannelMessageTypes.idle });
  };

  this.debug = function(message) {
    channel.send({ type: QWebChannelMessageTypes.debug, data: message });
  };

  // 给服务端发送 init 消息
  // 第一笔报文
  channel.exec({ type: QWebChannelMessageTypes.init }, function(data) {
    // 收到服务端初始化信息
    // 1、 构建 object
    for (var objectName in data) {
      var object = new QObject(objectName, data[objectName], channel);
    }
    // now unwrap properties, which might reference other registered objects
    for (var objectName in channel.objects) {
      channel.objects[objectName].unwrapProperties();
    }
    if (initCallback) {
      initCallback(channel);
    }
    channel.exec({ type: QWebChannelMessageTypes.idle });
  });
};

function QObject(name, data, webChannel) {
  // __id__ 为 object 名
  this.__id__ = name;
  // 将自身注册进全局 webChannel 的对象数组
  webChannel.objects[name] = this;

  // List of callbacks that get invoked upon signal emission
  // 信号回调对象
  this.__objectSignals__ = {};

  // Cache of all properties, updated when a notify signal is emitted
  // 属性对象
  this.__propertyCache__ = {};

  var object = this;

  // ----------------------------------------------------------------------

  this.unwrapQObject = function(response) {
    // 如果属性是数组则逐个展开
    if (response instanceof Array) {
      // support list of objects
      var ret = new Array(response.length);
      for (var i = 0; i < response.length; ++i) {
        ret[i] = object.unwrapQObject(response[i]);
      }
      return ret;
    }
    if (!response || !response["__QObject*__"] || response.id === undefined) {
      return response;
    }

    // 如果该对象已存在，则直接返回该对象的引用
    var objectId = response.id;
    if (webChannel.objects[objectId]) return webChannel.objects[objectId];

    if (!response.data) {
      console.error(
        "Cannot unwrap unknown QObject " + objectId + " without data."
      );
      return;
    }

    var qObject = new QObject(objectId, response.data, webChannel);
    qObject.destroyed.connect(function() {
      if (webChannel.objects[objectId] === qObject) {
        delete webChannel.objects[objectId];
        // reset the now deleted QObject to an empty {} object
        // just assigning {} though would not have the desired effect, but the
        // below also ensures all external references will see the empty map
        // NOTE: this detour is necessary to workaround QTBUG-40021
        var propertyNames = [];
        for (var propertyName in qObject) {
          propertyNames.push(propertyName);
        }
        for (var idx in propertyNames) {
          delete qObject[propertyNames[idx]];
        }
      }
    });
    // here we are already initialized, and thus must directly unwrap the properties
    qObject.unwrapProperties();
    return qObject;
  };

  // 对属性进行展开，属性可能是一个 object
  this.unwrapProperties = function() {
    for (var propertyIdx in object.__propertyCache__) {
      object.__propertyCache__[propertyIdx] = object.unwrapQObject(
        object.__propertyCache__[propertyIdx]
      );
    }
  };

  // 注册信号
  function addSignal(signalData, isPropertyNotifySignal) {
    var signalName = signalData[0];
    var signalIndex = signalData[1];
    object[signalName] = {
      connect: function(callback) {
        if (typeof callback !== "function") {
          console.error(
            "Bad callback given to connect to signal " + signalName
          );
          return;
        }

        object.__objectSignals__[signalIndex] =
          object.__objectSignals__[signalIndex] || [];
        object.__objectSignals__[signalIndex].push(callback);

        if (!isPropertyNotifySignal && signalName !== "destroyed") {
          // only required for "pure" signals, handled separately for properties in propertyUpdate
          // also note that we always get notified about the destroyed signal
          webChannel.exec({
            type: QWebChannelMessageTypes.connectToSignal,
            object: object.__id__,
            signal: signalIndex
          });
        }
      },
      disconnect: function(callback) {
        if (typeof callback !== "function") {
          console.error(
            "Bad callback given to disconnect from signal " + signalName
          );
          return;
        }
        object.__objectSignals__[signalIndex] =
          object.__objectSignals__[signalIndex] || [];
        var idx = object.__objectSignals__[signalIndex].indexOf(callback);
        if (idx === -1) {
          console.error(
            "Cannot find connection of signal " +
              signalName +
              " to " +
              callback.name
          );
          return;
        }
        object.__objectSignals__[signalIndex].splice(idx, 1);
        if (
          !isPropertyNotifySignal &&
          object.__objectSignals__[signalIndex].length === 0
        ) {
          // only required for "pure" signals, handled separately for properties in propertyUpdate
          webChannel.exec({
            type: QWebChannelMessageTypes.disconnectFromSignal,
            object: object.__id__,
            signal: signalIndex
          });
        }
      }
    };
  }

  /**
   * Invokes all callbacks for the given signalname. Also works for property notify callbacks.
   */
  // 对信号注册的多个回调函数逐一进行回调
  function invokeSignalCallbacks(signalName, signalArgs) {
    var connections = object.__objectSignals__[signalName];
    if (connections) {
      connections.forEach(function(callback) {
        callback.apply(callback, signalArgs);
      });
    }
  }

  this.propertyUpdate = function(signals, propertyMap) {
    // update property cache
    // 更新对象的属性
    for (var propertyIndex in propertyMap) {
      var propertyValue = propertyMap[propertyIndex];
      object.__propertyCache__[propertyIndex] = propertyValue;
    }

    for (var signalName in signals) {
      // Invoke all callbacks, as signalEmitted() does not. This ensures the
      // property cache is updated before the callbacks are invoked.
      // 调用属性改变信号
      invokeSignalCallbacks(signalName, signals[signalName]);
    }
  };

  this.signalEmitted = function(signalName, signalArgs) {
    invokeSignalCallbacks(signalName, this.unwrapQObject(signalArgs));
  };

  function addMethod(methodData) {
    var methodName = methodData[0];
    var methodIdx = methodData[1];
    object[methodName] = function() {
      var args = [];
      var callback;
      for (var i = 0; i < arguments.length; ++i) {
        var argument = arguments[i];
        if (typeof argument === "function") callback = argument;
        else if (
          argument instanceof QObject &&
          webChannel.objects[argument.__id__] !== undefined
        )
          args.push({
            id: argument.__id__
          });
        else args.push(argument);
      }

      webChannel.exec(
        {
          type: QWebChannelMessageTypes.invokeMethod,
          object: object.__id__,
          method: methodIdx,
          args: args
        },
        function(response) {
          if (response !== undefined) {
            var result = object.unwrapQObject(response);
            if (callback) {
              callback(result);
            }
          }
        }
      );
    };
  }

  /*
    "properties": [
        [0, "objectName", [1, 2], ""],
        [1, "userList", [1, 7], []]
      ],
  */
  function bindGetterSetter(propertyInfo) {
    var propertyIndex = propertyInfo[0];
    var propertyName = propertyInfo[1];
    var notifySignalData = propertyInfo[2];
    // initialize property cache with current value
    // NOTE: if this is an object, it is not directly unwrapped as it might
    // reference other QObject that we do not know yet
    object.__propertyCache__[propertyIndex] = propertyInfo[3];

    if (notifySignalData) {
      if (notifySignalData[0] === 1) {
        // 1 表示属性初始化设置，将属性名改为  propertyName + "Changed"
        // signal name is optimized away, reconstruct the actual name
        notifySignalData[0] = propertyName + "Changed";
      }
      // 注册属性改变信号，同时设置 true 不给服务端发送信号注册消息
      addSignal(notifySignalData, true);
    }

    Object.defineProperty(object, propertyName, {
      configurable: true,
      get: function() {
        var propertyValue = object.__propertyCache__[propertyIndex];
        if (propertyValue === undefined) {
          // This shouldn't happen
          console.warn(
            'Undefined value in property cache for property "' +
              propertyName +
              '" in object ' +
              object.__id__
          );
        }

        return propertyValue;
      },

      // 设置属性值触发 websocket 发送设置属性消息给服务端
      set: function(value) {
        if (value === undefined) {
          console.warn(
            "Property setter for " +
              propertyName +
              " called with undefined value!"
          );
          return;
        }
        object.__propertyCache__[propertyIndex] = value;
        var valueToSend = value;
        if (
          valueToSend instanceof QObject &&
          webChannel.objects[valueToSend.__id__] !== undefined
        )
          valueToSend = { id: valueToSend.__id__ };
        webChannel.exec({
          type: QWebChannelMessageTypes.setProperty,
          object: object.__id__,
          property: propertyIndex,
          value: valueToSend
        });
      }
    });
  }

  // ----------------------------------------------------------------------

  data.methods.forEach(addMethod);

  data.properties.forEach(bindGetterSetter);

  data.signals.forEach(function(signal) {
    addSignal(signal, false);
  });

  for (var name in data.enums) {
    object[name] = data.enums[name];
  }
}

//required for use with nodejs
if (typeof module === "object") {
  module.exports = {
    QWebChannel: QWebChannel
  };
}
```
