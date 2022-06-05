> 浏览器提供的兼容 w3c 的 WebSocket 对象的包装器。

```
webSocket<T>(urlConfigOrSource: string | WebSocketSubjectConfig<T>): WebSocketSubject<T>
```

## 参数

| 参数              | 类型                                | 说明                                                          |
| ----------------- | ----------------------------------- | ------------------------------------------------------------- |
| urlConfigOrSource | string \| WebSocketSubjectConfig<T> | WebSocket 端点作为一个 url 或一个带有配置和附加观察者的对象。 |

## 返回

**WebSocketSubject<T>**: 允许通过 WebSocket 连接发送和接收消息的主题。

## 描述

通过 WebSocket 与服务器通信的主题

webSocket 是一个工厂函数，它产生一个 WebSocketSubject，可以用它来建立与任意端点的 webSocket 连接。
webSocket 接受一个带有 webSocket 端点 url 的字符串作为参数，或者一个用于提供附加配置的 WebSocketSubjectConfig 对象，以及用于跟踪 webSocket 连接生命周期的观察者。

当 WebSocketSubject 被订阅时，它尝试建立一个套接字连接，除非已经建立了一个。
这意味着许多订阅者将始终侦听同一个套接字，从而节省资源。
但是，如果两个实例是由 WebSocketSubject 组成的，即使这两个实例提供了相同的 url，它们也会尝试建立单独的连接。
当 WebSocketSubject 的使用者退订时，套接字连接将被关闭，只有当没有更多的订阅者仍在监听时才会关闭。
如果一段时间后，使用者再次开始订阅，则重新建立连接。

一旦建立了连接，无论何时从服务器发来新消息，WebSocketSubject 都会将该消息作为流中的值发出。
默认情况下，来自套接字的消息通过 `JSON.parse` 解析。
如果你想定制反序列化的处理方式(如果有的话)，你可以在 WebSocketSubject 中提供定制的 resultSelector 函数。
当连接关闭时，流将完成，前提是它没有发生任何错误。
如果在任何时候(启动、维护或关闭连接)出现错误，stream 也会出现 WebSocket API 抛出的错误。

由于是一个 Subject, WebSocketSubject 允许从服务器接收和发送消息。
为了与连接的端点通信，请使用 next、error 和 complete 方法。
Next 向服务器发送一个值，因此请记住，这个值事先不会被序列化。
因此，在使用结果调用 next 之前，必须手工调用 `JSON.stringify`。
还要注意的是，如果在下一个值的时刻没有套接字连接(例如，没有人订阅)，这些值将被缓冲，并在最终建立连接时发送。
方法关闭套接字连接。
Error 也会这样做，并通过状态代码和字符串通知服务器发生了错误，其中包含了发生的细节。
由于状态码在 WebSocket API 中是必需的，WebSocketSubject 不允许像常规 Subject 一样，将任意值传递给错误方法。
需要使用一个对象来调用它，该对象具有带有状态码号的 code 属性和带有描述错误细节的字符串的可选原因属性。

调用 next 并不会影响 WebSocketSubject 的订阅者——他们不知道什么东西被发送到服务器(当然，除非服务器以某种方式响应消息)。
另一方面，因为调用 complete 会触发关闭套接字连接的尝试。
如果连接关闭而没有任何错误，流将完成，从而通知所有订阅者。
由于调用 error 也会关闭套接字连接，只是对服务器使用了不同的状态码，如果关闭本身没有发生错误，那么订阅的 Observable 就不会像预期的那样出错，而是像往常一样完成。
在这两种情况下(调用 complete 或 error)，如果关闭套接字连接的过程导致一些错误，则 stream 将出错。

## 多路复用

`WebSocketSubject` 有一个额外的操作符，不能在其他 `subject` 中找到。
它被称为 `multiplex` ，用于模拟打开多个套接字连接，而实际上只维护一个。
例如，一个应用程序有聊天面板和体育新闻的实时通知。
因为这是两个不同的函数，所以为每个函数建立两个独立的连接是有意义的。
甚至可能有两个带有 `WebSocket` 端点的独立服务，它们运行在单独的机器上，只有 `GUI` 将它们组合在一起。
为每个功能使用套接字连接可能会消耗过多的资源。
使用单个 `WebSocket` 端点作为其他服务(在本例中是聊天和体育新闻服务)的网关是一种常见的模式。
即使在客户端应用程序中只有一个连接，也需要能够将流当作两个独立的套接字来操作。
这就消除了在网关中为给定服务手动注册和取消注册的问题，并过滤出相关消息。
这就是 `multiplex` 方法的作用。

方法接受三个参数。
前两个函数分别返回订阅和取消订阅消息。
这些消息将被发送到服务器，无论产生的可观察对象的消费者何时订阅或取消订阅。
服务器可以使用它们来验证某些类型的消息是否应该开始或停止转发给客户端。
对于上述示例应用，网关服务器在获得具有适当标识符的订阅消息后，可以决定连接到真正的体育新闻服务，并从该服务开始转发消息。
请注意，这两个消息都将作为函数的返回发送，它们在默认情况下使用 `JSON.stringify` 序列化，就像通过 `next` 推送的消息一样。
还要记住，这些消息将在每次订阅和取消订阅时发送。
这是潜在的危险，因为 `Observable` 的一个消费者可能会退订，而服务器可能会停止发送消息，因为它收到了退订消息。
这需要在服务器上处理，或者在从 `multiplex` 返回的 `Observable` 上使用 `publish`。

multiplex 的最后一个参数是一个 `messageFilter` 函数，它应该返回一个布尔值。
它用于过滤掉服务器发送给那些属于模拟 WebSocket 流的消息。
例如，服务器可能在消息对象上用某种类型的字符串标识符标记这些消息，如果套接字发出的对象上有这样的标识符，messageFilter 将返回 true。
在 messageFilter 中返回 false 的消息将被跳过，并且不会向下传递。

multiplex 的返回值是一个 Observable，消息来自模拟的套接字连接。
注意，这不是 WebSocketSubject，因此再次调用 next 或 multiplex 将会失败。
要将值推送到服务器，请使用根 WebSocketSubject。

## 举例

```ts title="监听来自服务器的消息"
import { webSocket } from "rxjs/webSocket";
const subject = webSocket("ws://localhost:8081");
subject.subscribe({
  next: (msg) => console.log("message received: " + msg), // 每当有来自服务器的消息时调用。
  error: (err) => console.log(err), // 如果WebSocket API在任何时候发出某种错误信号，就调用该函数。
  complete: () => console.log("complete"), // 在连接关闭时调用(无论出于何种原因)。
});
```

```ts title="向服务器推送消息"
import { webSocket } from "rxjs/webSocket";
const subject = webSocket("ws://localhost:8081");
subject.subscribe();
// 注意，至少要有一个消费者订阅创建的主题-否则，“nexted”的值将只是缓冲而不发送，因为没有建立连接!
subject.next({ message: "some message" });
// 这将在建立连接后向服务器发送消息。
// 记住值是序列化的JSON.stringify默认!
subject.complete(); // 关闭连接。
subject.error({ code: 4000, reason: "我想我们的应用程序坏了!" });
// 也关闭连接，但让服务器知道关闭是由某些错误引起的。
```

```ts title="多路复用 WebSocket"
import { webSocket } from "rxjs/webSocket";

const subject = webSocket("ws://localhost:8081");

const observableA = subject.multiplex(
  () => ({ subscribe: "A" }), // 当服务器收到这条消息时，它将开始为'A'发送消息…
  () => ({ unsubscribe: "A" }), // .．.一旦得到这个，它就会停止。
  (message) => message.type === "A" // 如果函数返回 true 消息将沿流传递。如果函数返回 false 则跳过。
);

const observableB = subject.multiplex(
  // B也是如此。
  () => ({ subscribe: "B" }),
  () => ({ unsubscribe: "B" }),
  (message) => message.type === "B"
);

const subA = observableA.subscribe((messageForA) => console.log(messageForA));
// 此时，WebSocket连接已经建立。
// 服务器获取'{"subscribe": "A"}'消息，并开始为'A'发送消息，我们在这里记录它。

const subB = observableB.subscribe((messageForB) => console.log(messageForB));
// 因为我们已经有了一个连接，所以我们只向服务器发送'{"subscribe": "B"}'消息。
// 它开始为B发送消息，我们在这里记录。

subB.unsubscribe();
// 消息'{"unsubscribe": "B"}'被发送到服务器，服务器停止发送'B'消息。

subA.unsubscribe();
// 消息'{"unsubscribe": "A"}'将使服务器停止为'A'发送消息。
// 由于没有根Subject的更多订阅者，套接字连接关闭。
```
