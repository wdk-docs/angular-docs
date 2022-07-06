# 如何使用 Angular 组件继承来扩展类

> https://www.digitalocean.com/community/tutorials/angular-component-inheritance

## 介绍

如果你花过时间在 Angular 上，你可能遇到过这样的情况:当你想要共享数据或功能时，你使用了服务/提供商。

如果您想要的不是通用数据功能，而是通用 UI 功能呢?例如，考虑一个简单的场景，您希望使用按钮从一个组件导航到另一个组件。实现这个的一个简单方法是创建一个按钮，在代码中调用一个方法，然后使用 Angular 的路由器导航到页面。
如果不希望在每个组件中重复相同的代码，该怎么办?
Typescript 和 Angular 会给你一种处理这种封装的方法。继承的组件!

在 TypeScript 中使用类继承，你可以声明一个包含常见 UI 功能的基础组件，并使用它来扩展任何你想要的标准组件。
如果您习惯于任何专注于面向对象方法(如 c#)的语言，那么您将很容易识别这种方法为继承。
在 Typescript 中，它仅仅被称为扩展一个类。

在本文中，您将在基本组件中封装通用代码，并对其进行扩展，以创建可重用的页面导航按钮。

## 先决条件

要完成本教程，你需要:

你可以按照如何安装 Node.js 和创建本地开发环境来做。
熟悉如何设置 Angular 项目。
本教程已经验证过 Node v16.4.2、npm v7.19.1 和@angular/core v12.1.1。

## 步骤 1 -设置项目

让我们从使用 Angular CLI 创建一个新应用开始。

如果你以前没有安装过 Angular CLI，可以使用 npm 全局安装:

```sh
npm install -g @angular/cli
```

接下来，使用 CLI 创建新的应用程序:

```sh
ng new AngularComponentInheritance --style=css --routing --skip-tests
```

!!! note

    我们正在给ng new命令传递一些标志，以向我们的应用添加路由(——routing)，而不是添加任何测试文件 (--skip-tests).

导航到项目目录:

```sh
cd AngularComponentInheritance
```

然后，运行如下命令创建一个 Base 组件:

```sh
ng generate component base --inline-template --inline-style --skip-tests --module app
```

!!! note

    这里的 `--module` 标志指定组件应该属于哪个模块。

该命令将创建一个 `base.component.ts` 文件，并将其添加为 app 模块的声明。

## 步骤 2 -构建基本组件

用代码编辑器打开`base/base.component.ts`文件:

```ts title='src/app/base/base.component.ts'
import { Component, OnInit } from "@angular/core";

@Component({
  selector: "app-base",
  template: ` <p>base works!</p> `,
  styles: [],
})
export class BaseComponent implements OnInit {
  constructor() {}

  ngOnInit(): void {}
}
```

这个 UI 永远不会显示，所以除了一个简单的 UI 之外，不需要添加任何东西。

接下来，把 Router 注入到组件中:

```ts title='src/app/base/base.component.ts'
import { Component, OnInit } from "@angular/core";
import { Router } from "@angular/router";

@Component({
  selector: "app-base",
  template: ` <p>base works!</p> `,
  styles: [],
})
export class BaseComponent implements OnInit {
  constructor(public router: Router) {}
  ngOnInit(): void {}
}
```

请注意可访问性级别。
由于继承，保持这个声明为`public`是很重要的。

接下来，给基础组件添加一个名为 `openPage` 的方法，它接受一个字符串并使用它导航到一个路由(注意:使用下面的标记代替模板字面量的单引号):

```ts title='src/app/base/base.component.ts'
import { Component, OnInit } from "@angular/core";
import { Router } from "@angular/router";

@Component({
  selector: "app-base",
  template: ` <p>base works!</p> `,
  styles: [],
})
export class BaseComponent implements OnInit {
  constructor(public router: Router) {}

  ngOnInit(): void {}

  openPage(routename: string) {
    this.router.navigateByUrl(`/${routename}`);
  }
}
```

这为我们提供了所需的基本功能，所以让我们在一些组件上使用它。

## 步骤 3 -继承组件

我们将运行三个 Angular CLI 命令来生成更多组件:

```sh
ng generate component pageone --skip-tests --module app
ng generate component pagetwo --skip-tests --module app
ng generate component pagethree --skip-tests --module app
```

这些命令将生成一个 `PageoneComponent` 、 `PagetwoComponent` 和 `PagethreeComponent` ，并将它们作为应用程序的声明添加到应用程序中。

打开`app-routing.modulle.ts`，它是在我们第一次生成应用程序时由 CLI 创建的，并为每个页面添加一个路径:

```ts title='src/app/app-routing.module.ts'
import { NgModule } from "@angular/core";
import { RouterModule, Routes } from "@angular/router";
import { PageoneComponent } from "./pageone/pageone.component";
import { PagetwoComponent } from "./pagetwo/pagetwo.component";
import { PagethreeComponent } from "./pagethree/pagethree.component";

const routes: Routes = [
  { path: "", component: PageoneComponent },
  { path: "pageone", component: PageoneComponent },
  { path: "pagetwo", component: PagetwoComponent },
  { path: "pagethree", component: PagethreeComponent },
];

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule],
})
export class AppRoutingModule {}
```

打开页面组件， 让它扩展 `BaseComponent` :

```ts title='src/app/pageone/pageone.component.ts'
import { Component, OnInit } from "@angular/core";
import { BaseComponent } from "../base/base.component";

@Component({
  selector: "app-pageone",
  templateUrl: "./pageone.component.html",
  styleUrls: ["./pageone.component.css"],
})
export class PageoneComponent extends BaseComponent implements OnInit {
  constructor() {}

  ngOnInit(): void {}
}
```

添加路由器并使用 super 将其注入到 BaseComponent 构造函数中:

```ts title='src/app/pageone/pageone.component.ts'
import { Component, OnInit } from "@angular/core";
import { Router } from "@angular/router";
import { BaseComponent } from "../base/base.component";

@Component({
  selector: "app-pageone",
  templateUrl: "./pageone.component.html",
  styleUrls: ["./pageone.component.css"],
})
export class PageoneComponent extends BaseComponent implements OnInit {
  constructor(public router: Router) {
    super(router);
  }

  ngOnInit(): void {}
}
```

这将获取注入的路由器模块，并将其传递给扩展组件。

接下来，对剩余的页面组件重复这些更改。

由于继承自基组件，所以在基组件上定义的任何东西都可以被所有扩展它的组件使用。我们用基函数。

让我们在 `page.onecomponent.html` 模板中添加两个按钮:

```html title='src/app/pageone/pageone.component.html'
<button type="button" (click)="openPage('pagetwo')">Page Two</button>
<button type="button" (click)="openPage('pagethree')">Page Three</button>
```

注意到使用 openPage 方法不需要额外的限制吗?
如果考虑如何在 c# 等语言中进行类似的继承，可以调用 base.openPage()等方法。
你不必在 TypeScript 中这样做的原因是在转译过程中发生的魔法。
TypeScript 会把代码转换成 JavaScript，而基础组件模块会被导入到 pageone 组件中，以便该组件可以直接使用它。

看看编译后的 JavaScript 会让这一点更清楚:

```ts title=''
var PageoneComponent = /** @class */ (function (_super) {
  __extends(PageoneComponent, _super);
  function PageoneComponent(router) {
    var _this = _super.call(this, router) || this;
    _this.router = router;
    return _this;
  }
  PageoneComponent.prototype.ngOnInit = function () {};
  PageoneComponent = __decorate(
    [
      Object(_angular_core__WEBPACK_IMPORTED_MODULE_0__["Component"])({
        selector: "app-pageone",
        template: __webpack_require__(
          "./src/app/pageone/pageone.component.html"
        ),
        styles: [
          __webpack_require__("./src/app/pageone/pageone.component.css"),
        ],
      }),
      __metadata("design:paramtypes", [
        _angular_router__WEBPACK_IMPORTED_MODULE_2__["Router"],
      ]),
    ],
    PageoneComponent
  );
  return PageoneComponent;
})(_base_base_component__WEBPACK_IMPORTED_MODULE_1__["BaseComponent"]);
```

这也是为什么我们需要将注入的模块保持为公有而不是私有。
在 TypeScript 中，`super()`是调用基组件的构造函数的方式，它要求传递所有注入的模块。
当模块是私有的，它们成为单独的声明。
将它们保持为公共的，并使用 super 传递它们，它们仍然是单个声明。

## 步骤 4 -完成应用程序

花点时间用你的代码编辑器删除 app.component.html 中的样板代码，只留下`<router-outlet>`:

```html title='src/app/app.component.html'
<router-outlet></router-outlet>
```

完成 Pageone 后，让我们使用 CLI 来运行这个应用程序并研究一下它的功能:

```sh
ng serve
```

单击其中一个按钮，就会看到您被引导到预期的页面。

为单个组件封装功能的开销太大了，所以让我们扩展 Pagetwo 和 Pagethree 组件，并添加按钮来帮助导航到其他页面。

首先，打开 `pagetwo.component.ts` 并像 `pageone.component.ts` 那样更新它:

```ts title='src/app/pagetwo/pagetwo.component.ts'
import { Component, OnInit } from "@angular/core";
import { Router } from "@angular/router";
import { BaseComponent } from "../base/base.component";

@Component({
  selector: "app-pagetwo",
  templateUrl: "./pagetwo.component.html",
  styleUrls: ["./pagetwo.component.css"],
})
export class PagetwoComponent extends BaseComponent implements OnInit {
  constructor(public router: Router) {
    super(router);
  }

  ngOnInit(): void {}
}
```

然后，打开`pagetwo.component.html`并为 `pageone` 和 `pagethree` 添加一个按钮:

```ts title='src/app/pagetwo/pagetwo.component.html'
<button type="button" (click)="openPage('pageone')">
  Page One
</button>

<button type="button" (click)="openPage('pagethree')">
  Page Three
</button>
```

接下来，打开 `pagethree.component.ts` 并像 `pageone.component.ts` 一样更新它:

```ts title='src/app/pagethree/pagethree.component.ts'
import { Component, OnInit } from "@angular/core";
import { Router } from "@angular/router";
import { BaseComponent } from "../base/base.component";

@Component({
  selector: "app-pagethree",
  templateUrl: "./pagethree.component.html",
  styleUrls: ["./pagethree.component.css"],
})
export class PagethreeComponent extends BaseComponent implements OnInit {
  constructor(public router: Router) {
    super(router);
  }

  ngOnInit(): void {}
}
```

然后，打开 `pagethree.component.html` 并为 pageone 和 pagetwo 添加一个按钮:

```html title='src/app/pagethree/pagethree.component.html'
<button type="button" (click)="openPage('pageone')">Page One</button>

<button type="button" (click)="openPage('pagetwo')">Page Two</button>
```

现在你可以在不重复任何逻辑的情况下浏览整个应用程序。

## 结论

在本文中，您将通用代码封装在基本组件中，并对其进行扩展，以创建可重用的页面导航按钮。

从这里可以很容易地看到如何跨许多组件扩展通用功能。无论你是在处理导航、通用模态警报 UI 还是其他什么，使用 TypeScript 授予的继承模型来扩展组件都是一个强大的工具，可以放在我们的工具箱里。

本教程的代码可以在[GitHub](https://github.com/do-community/AngularComponentInheritance)上找到。
