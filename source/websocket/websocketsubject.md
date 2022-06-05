通过 RxJS WebSocketSubject 只需要几行代码就可以从服务器传输实时数据。

WebSocket 为 HTML5 服务已经有一段时间了，它可以在所有现代浏览器上使用。

与 HTTP 连接不同，在 HTTP 连接中，客户端需要不断轮询服务器以获得最新的更新;
WebSocket 允许浏览器通过单个套接字连接不断地接收和发送消息到服务器。

任意的 WebSocket URL 是这样的:

```
ws://www.sample.com
wss://www.sample.com // encrypted websocket
```

加密的 WebSocket URL 类似于 HTTPS 连接。

## 为什么使用 RxJS

📌 框架不可知

📌 反应性编程

📌 简单的集成

通常， `socket.io` 用于构建实时 JS 服务器和客户端。

在本演练中，我们将重点关注客户端部分，而 RxJS 将是一个很好的选择。
在处理 WebSocket 连接时，我们还可以利用其他的 RxJS 操作符使我们的 JS 应用程序更具响应性。

在本指南的末尾有一个完整的工作示例。

## 安装

简单地在你的项目根目录下运行这个命令来添加 RxJS 库作为一个依赖:

```sh
npm i -S rxjs
```

## 让我们连接

要连接到现有的 WebSocket 端点，我们需要创建一个 WebSocketSubject 的实例。

我们可以使用 RxJS 提供的 websocket 工厂函数来实现这一点。
该函数接受一个 URL 字符串或配置对象。
配置对象遵循 WebSocketSubjectConfig 接口。

```ts
import { webSocket } from "rxjs/webSocket";

const wsSubject$ = webSocket("wss://www.gasnow.org/ws/gasprice");
// or
const wsSubjec2$ = webSocket({
  url: "wss://www.gasnow.org/ws/gasprice",
});
```

在这个演示中，我们使用了一个来自 gasnow.org 的公共 WebSocket 端点。
服务器每 8 秒推送一次以太坊天然气价格数据。

## RxJS 在行动

要开始接收数据，我们只需订阅 WebSocketSubject 实例。
我们可以从 RxJS 中注销接收到的天然气价格数据。

```ts
import { tap } from "rxjs/operators";
import { webSocket } from "rxjs/webSocket";

const wsSubjec2$ = webSocket({
  url: "wss://www.gasnow.org/ws/gasprice",
});

wsSubject$.pipe(tap((data) => console.log(data))).subscribe();
```

注意，如果多次订阅同一个 WebSocketSubject 实例，所有订阅者将侦听相同的套接字连接。
这有助于我们节省一些资源!

```ts
import { tap } from "rxjs/operators";
import { webSocket } from "rxjs/webSocket";

const wsSubjec2$ = webSocket({
  url: "wss://www.gasnow.org/ws/gasprice",
});

const subA = wsSubject$.pipe(tap((data) => console.log(data))).subscribe();

const subB = wsSubject$.pipe(tap((d) => console.log(d))).subscribe();
```

## 处理网络错误与重试

如果没有正确的网络错误处理，我们的实现将不完整。
如果连接中有错误，我们应该尝试重新连接到套接字。
这就是我们可以利用 RxJS 的力量的地方。

RxJS 提供了 retryWhen 操作符，它允许我们在发生错误时重试可观察序列。
此外，delayWhen 和 timer 操作符使我们能够在启动重试机制之前设置 X 时间的延迟。

```ts
import { tap. retryWhen } from 'rxjs/operators';
import { webSocket } from 'rxjs/webSocket';
import { timer } from 'rxjs';

const wsSubjec2$ = webSocket({
  url: 'wss://www.gasnow.org/ws/gasprice',
});

wsSubject$
    .pipe(
      tap((data) => console.log(data)),
      retryWhen((errors) => errors.pipe(delayWhen((val) => timer(val * 1000))))
    )
    .subscribe();
```

> 但是我们如何知道连接何时关闭/丢失呢?

当 CloseEvent 发生时，我们可以利用 WebSocketSubjectConfig 中的 closeObserver 属性来通知我们。
从 WebAPI 文档中，我们可以看到 wasClean 属性可用于确定连接是干净地关闭还是突然终止。

```ts
import { tap, retryWhen } from "rxjs/operators";
import { webSocket } from "rxjs/webSocket";
import { timer } from "rxjs";

const wsSubjectConfig = {
  url: "wss://www.gasnow.org/ws/gasprice",
  // added this to the config object
  closeObserver: {
    // this is triggered when connection is closed
    next(event) {
      if (!event.wasClean) {
        connect(); //
      }
    },
  },
};

const wsSubject$ = webSocket(wsSubjectConfig);

function connect() {
  wsSubject$
    .pipe(
      tap((data) => console.log(data)),
      retryWhen((errors) => errors.pipe(delayWhen((val) => timer(val * 1000))))
    )
    .subscribe();
}

connect();
```

这个检查很重要，因为我们将在后面看到，如果我们手动关闭连接，closeObserver 也将捕获 CloseEventas。

## 通过避免内存泄漏来提高性能

一旦我们订阅了 WebSocketSubjectinstance，我们将继续接收数据，即使我们导航离开当前视图。
因此，重要的是要取消订阅，以防止内存泄漏，并避免为同一个连接重新创建多个实例。

要关闭连接，只需调用 complete 方法:

这里的代码片段

```ts
import { webSocket } from "rxjs/webSocket";

const wsSubjectConfig = {
  url: "wss://www.gasnow.org/ws/gasprice",
  closeObserver: {
    next(event) {}, // 这在连接关闭时触发
  }, // 添加到config对象
};

const wsSubject$ = webSocket(wsSubjectConfig);

wsSubject$.subscribe();

function disconnect() {
  if (wsSubject$) {
    wsSubject$.complete();
  }
}
```

当连接完全关闭时， `closingObserver.next` 和 `closeObserver.next` 都将被触发。
这为我们提供了一种连接到 WebSocketSubject 生命周期的方法。
然后，我们可以运行一个函数或通知用户连接已关闭。

## 总结

本文的主要目的是展示如何使用 RxJS 建立 WebSocket 连接。
因此，有些代码可能没有完全优化。
RxJS 不仅功能强大，而且可以直接集成到任何前端框架中。

香草 JS 的完整代码可以在下面找到。

stackblitz 上的完整工作示例
