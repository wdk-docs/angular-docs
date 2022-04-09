# 使用 Angular Universal 时的重要注意事项

## 介绍

尽管 Universal 项目的目标是能够在服务器上无缝地呈现 Angular 应用，但还是有一些不一致的地方需要考虑。
首先，服务器和浏览器环境之间存在明显的差异。
在服务器上呈现时，应用程序处于临时或“快照”状态。
应用程序只完全呈现一次，返回生成的 HTML，在下一次呈现之前销毁其余的应用程序状态。
其次，服务器环境本身不具有与浏览器相同的功能(并且具有一些浏览器不具有的功能)。
例如，服务器没有任何 cookie 的概念。
你可以填充这个和其他功能，但没有完美的解决方案。
在后面的部分中，我们将介绍可能的缓解措施，以减少在服务器上呈现时的错误范围。

也请注意 SSR 的目标:改善您的应用程序的初始渲染时间。
这意味着，应该避免或充分防范任何可能在初始呈现中降低应用程序速度的情况。
同样，我们将在后面的章节中回顾如何实现这一点。

## "window is not defined"

使用 Angular Universal 时最常见的问题之一是服务器环境中缺少浏览器全局变量。
这是因为 Universal 项目使用[domino](https://github.com/fgnass/domino)作为服务器 DOM 渲染引擎。
因此，某些功能将不会在服务器上显示或支持。
这包括“window”和“document”全局对象、cookie、某些 HTML 元素(比如 canvas)，以及其他一些元素。
这里没有详尽的列表，因此请注意，如果您看到类似这样的错误，其中没有定义以前可访问的全局变量，很可能是因为该全局变量不能通过 domino 使用。

> 有趣的事实: Domino 代表 “Node 中的 DOM”

### 如何修复?

#### 策略 1:注射

通常，需要的全局变量可以通过依赖注入(DI)在 Angular 平台中获得。
例如，全局 `document` 可通过 `DOCUMENT` 令牌获得。
此外， `window` 和 `location` 的 _very_ 基元版本通过 `DOCUMENT` 对象存在。

例如:

```ts
// example.service.ts
import { Injectable, Inject } from "@angular/core";
import { DOCUMENT } from "@angular/common";

@Injectable()
export class ExampleService {
  constructor(@Inject(DOCUMENT) private _doc: Document) {}

  getWindow(): Window | null {
    return this._doc.defaultView;
  }

  getLocation(): Location {
    return this._doc.location;
  }

  createElement(tag: string): HTMLElement {
    return this._doc.createElement(tag);
  }
}
```

请谨慎使用这些推荐人，并降低你对他们能力的期望。
`localStorage` 是一个经常被请求的 API，它不会以你希望的方式工作。
如果你需要编写自己的库组件，请考虑使用这个方法在服务器上提供类似的功能(这就是 Angular CDK 和 Material 所做的)。

#### 策略 2:警卫

如果你不能从 Angular 平台中注入你需要的正确的全局值，你可以`guard`调用浏览器代码，只要你不需要在服务器上访问这些代码。
例如，通常调用全局`window`元素是为了获取窗口大小或其他一些可视方面。
然而，在服务器上，没有`screen`的概念，因此很少需要这个功能。

你可以在网上或其他地方读到推荐的方法是使用' `isPlatformBrowser` '或' `isPlatformServer` '。
这个指导是**错误的**。
这是因为您最终在应用程序代码中创建了特定于平台的代码分支。
这不仅不必要地增加了应用程序的大小，而且还增加了必须维护的复杂性。
通过将代码分离到独立的特定于平台的模块和实现中，您的基本代码可以保留业务逻辑和特定于平台的异常
按照它们应该的方式处理:在逐项抽象的基础上。
这可以通过使用 Angular 的依赖注入(DI)来实现，以便在运行时删除有问题的代码并添加替换代码。

这里有一个例子:

```ts
// window-service.ts
import { Injectable } from "@angular/core";

@Injectable()
export class WindowService {
  getWidth(): number {
    return window.innerWidth;
  }
}
```

```ts
// server-window.service.ts
import { Injectable } from "@angular/core";
import { WindowService } from "./window.service";

@Injectable()
export class ServerWindowService extends WindowService {
  getWidth(): number {
    return 0;
  }
}
```

```ts
// app-server.module.ts
import {NgModule} from '@angular/core';
import {WindowService} from './window.service';
import {ServerWindowService} from './server-window.service';

@NgModule({
  providers: [{
    provide: WindowService,
    useClass: ServerWindowService,
  }]
})
```

如果你有一个由第三方提供的组件，它不是通用兼容的，
除了你的基础应用模块之外，你可以为浏览器和服务器创建两个独立的模块(服务器模块你应该已经有了)。
基础应用模块将包含所有平台无关的代码，浏览器模块将包含所有浏览器特定/服务器不兼容的代码，服务器模块亦然。
为了避免编辑过多的模板代码，你可以为库组件创建一个无操作组件。

这里有一个例子:

```ts
// example.component.ts
import { Component } from "@angular/core";

@Component({
  selector: "example-component",
  template: `<library-component></library-component>`, // 这是由第三方库提供的
  // 这会导致Universal的渲染问题
})
export class ExampleComponent {}
```

```ts
// app.module.ts
import {NgModule} from '@angular/core';
import {ExampleComponent} from './example.component';

@NgModule({
  declarations: [ExampleComponent],
})
```

```ts
// browser-app.module.ts
import {NgModule} from '@angular/core';
import {LibraryModule} from 'some-lib';
import {AppModule} from './app.module';

@NgModule({
  imports: [AppModule, LibraryModule],
})
```

```ts
// library-shim.component.ts
import { Component } from "@angular/core";

@Component({
  selector: "library-component",
  template: "",
})
export class LibraryShimComponent {}
```

```ts
// server.app.module.ts
import { NgModule } from "@angular/core";
import { LibraryShimComponent } from "./library-shim.component";
import { AppModule } from "./app.module";

@NgModule({
  imports: [AppModule],
  declarations: [LibraryShimComponent],
})
export class ServerAppModule {}
```

#### 策略 3:垫片

如果其他所有方法都失败了，并且您必须能够访问某种浏览器功能，那么您可以对服务器环境的全局范围进行修补，以包括您需要的全局范围。

例如:

```ts
// server.ts
global["window"] = {
  // 你需要在这里实现的属性…
};
```

这可以应用于任何未定义的元素。
这样做时请小心，因为使用全局作用域通常被认为是一种反模式。

> 有趣的事实: 垫片是一个功能补丁，在给定的平台上永远不会被支持。
> polyfill 是计划支持的功能补丁，或在新版本上支持的功能补丁

## 应用程序很慢，或者更糟，不能渲染

Angular Universal 的渲染过程很简单，但也很容易被一些善意的或看似无害的代码阻塞或减慢。
首先，介绍一些渲染过程的背景知识。
当对平台服务器(Angular Universal platform)发出渲染请求时，只会执行一个路由导航。
当导航完成时，意味着所有 Zone.js 宏任务都完成了，DOM 无论处于什么状态都返回给用户。

> 一个 Zone.js 宏任务只是一个 JavaScript 宏任务，它在/是 Zone.js 补丁中执行

这意味着，如果有一个过程，比如微任务，需要一些时间才能完成，或者需要很长时间
HTTP 请求时，呈现过程将不会完成，或将花费更长的时间。
宏任务包括对全局变量的调用，比如' setTimeout '和' setInterval '，以及' Observables '。
在不取消它们的情况下调用它们，或者让它们在服务器上运行超过需要的时间，可能会导致次优的呈现。

> 如果您还不了解 JavaScript 事件循环，并了解微任务和宏任务之间的区别，那么这可能是值得的。
> 这里有一个很好的[参考](https://javascript.info/event-loop)。

## 我的 HTTP, Firebase, WebSocket 等等。

在渲染之前不会完成!

与上面等待宏任务完成的部分类似，另一方面是平台在完成呈现之前不会等待微任务完成。
在 Angular Universal 中，我们已经将 Angular HTTP 客户端打了补丁，将其转换为一个宏任务，以确保任何需要的 HTTP 请求都能在给定的渲染中完成。
然而，这种类型的补丁可能并不适用于所有的微任务，因此建议您根据自己的最佳判断如何继续。
您可以查看代码参考，了解 Universal 如何包装任务，将其转换为宏任务，或者您可以简单地选择更改给定任务的服务器行为。
