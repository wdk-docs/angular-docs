# Angular Signals

**Angular Signals**是一个系统，它可以细粒度地跟踪你的状态在整个应用程序中的使用方式和位置，允许框架优化渲染更新。

!!! note

    Angular 信号可以在[开发者预览](/guide/releases#developer-preview)中找到。他们已经准备好让你尝试，但在稳定之前可能会改变。

## 什么是信号?

**signal**是对一个值的包装，当该值发生变化时，它可以通知感兴趣的使用者。信号可以包含任何值，从简单的原语到复杂的数据结构。

信号的值总是通过 getter 函数读取，这允许 Angular 跟踪信号在哪里被使用。

信号可以是 _可写_ 的也可以是 _只读_ 的。

### 可写的信号

可写信号提供了一个 API 来直接更新它们的值。
你可以通过使用信号的初始值调用`signal`函数来创建可写信号:

```ts
const count = signal(0);

// Signals are getter functions - calling them reads their value.
console.log("The count is: " + count());
```

To change the value of a writable signal, you can either `.set()` it directly:

```ts
count.set(3);
```

or use the `.update()` operation to compute a new value from the previous one:

```ts
// Increment the count by 1.
count.update((value) => value + 1);
```

When working with signals that contain objects, it's sometimes useful to mutate that object directly. For example, if the object is an array, you may want to push a new value without replacing the array entirely. To make an internal change like this, use the `.mutate` method:

```ts
const todos = signal([{ title: "Learn signals", done: false }]);

todos.mutate((value) => {
  // Change the first TODO in the array to 'done: true' without replacing it.
  value[0].done = true;
});
```

Writable signals have the type `WritableSignal`.

### 计算信号

A **computed signal** derives its value from other signals. Define one using `computed` and specifying a derivation function:

```typescript
const count: WritableSignal<number> = signal(0);
const doubleCount: Signal<number> = computed(() => count() * 2);
```

The `doubleCount` signal depends on `count`. Whenever `count` updates, Angular knows that anything which depends on either `count` or `doubleCount` needs to update as well.

#### 计算型对象既是惰性求值型对象，也是记忆型对象

`doubleCount`'s derivation function does not run to calculate its value until the first time `doubleCount` is read. Once calculated, this value is cached, and future reads of `doubleCount` will return the cached value without recalculating.

When `count` changes, it tells `doubleCount` that its cached value is no longer valid, and the value is only recalculated on the next read of `doubleCount`.

As a result, it's safe to perform computationally expensive derivations in computed signals, such as filtering arrays.

#### 计算信号不是可写信号

你不能直接给计算信号赋值。也就是说,

```ts
doubleCount.set(3);
```

produces a compilation error, because `doubleCount` is not a `WritableSignal`.

#### 计算信号依赖是动态的

Only the signals actually read during the derivation are tracked. For example, in this computed the `count` signal is only read conditionally:

```ts
const showCount = signal(false);
const count = signal(0);
const conditionalCount = computed(() => {
  if (showCount()) {
    return `The count is ${count()}.`;
  } else {
    return "Nothing to see here!";
  }
});
```

When reading `conditionalCount`, if `showCount` is `false` the "Nothing to see here!" message is returned _without_ reading the `count` signal. This means that updates to `count` will not result in a recomputation.

If `showCount` is later set to `true` and `conditionalCount` is read again, the derivation will re-execute and take the branch where `showCount` is `true`, returning the message which shows the value of `count`. Changes to `count` will then invalidate `conditionalCount`'s cached value.

Note that dependencies can be removed as well as added. If `showCount` is later set to `false` again, then `count` will no longer be considered a dependency of `conditionalCount`.

## 在`OnPush`组件中读取信号

当`OnPush`组件在其模板中使用信号的值时，Angular 将跟踪该信号作为该组件的依赖。
当该信号被更新时，Angular 会自动[标记](/api/core/ChangeDetectorRef#markforcheck)组件，以确保它在下一次变化检测运行时得到更新。
有关`OnPush`组件的更多信息，请参阅[跳过组件子树](/guide/change-detection-skip -subtrees)指南。

## Effects

信号很有用，因为它们可以在发生变化时通知感兴趣的消费者。
**effect**是指当一个或多个信号值发生变化时运行的操作。
你可以使用`effect`函数创建特效:

```ts
effect(() => {
  console.log(`The current count is: ${count()}`);
});
```

特效总是**至少运行一次**。
当 effect 运行时，它会跟踪读取的任何信号值。
只要这些信号的值发生变化，特效就会再次运行。
与计算信号类似，effect 动态跟踪它们的依赖关系，并且只跟踪最近执行中读取的信号。

在变化检测过程中，effect 总是**异步**执行。

### effect 用途

在大多数应用程序代码中很少需要 effect，但在特定情况下可能会很有用。下面是一些使用`effect`可能是一个很好的解决方案的例子:

- 记录显示的数据，以及当它发生变化时，可以用于分析或作为调试工具
- 与`window.localStorage`保持数据同步
- 添加无法用模板语法表达的自定义 DOM 行为
- 对`<canvas>`、图表库或其他第三方 UI 库进行自定义渲染

#### 何时不使用

避免使用 effect 来传播状态变化。这可能会导致`ExpressionChangedAfterItHasBeenChecked`错误，无限循环的更新，或者不必要的变化检测循环。

由于存在这些风险，默认情况下是不允许设置信号的，但在绝对必要的情况下可以启用。

### 注入上下文

默认情况下，向`effect()`函数注册一个新的 effect 需要一个“注入上下文”(访问`inject`函数)。
最简单的方法是在组件、指令或服务的`constructor`中调用`effect`:

```ts
@Component({...})
export class EffectiveCounterCmp {
  readonly count = signal(0);
  constructor() {
    // Register a new effect.
    effect(() => {
      console.log(`The count is: ${this.count()})`);
    });
  }
}
```

或者，也可以将 effect 赋值给字段(也会给它一个描述性的名称)。

```ts
@Component({...})
export class EffectiveCounterCmp {
  readonly count = signal(0);

  private loggingEffect = effect(() => {
    console.log(`The count is: ${this.count()})`);
  });
}
```

要在构造函数之外创建 effect，你可以通过`effect`的选项将`Injector`传递给它:

```ts
@Component({...})
export class EffectiveCounterCmp {
  readonly count = signal(0);
  constructor(private injector: Injector) {}

  initializeLogging(): void {
    effect(() => {
      console.log(`The count is: ${this.count()})`);
    }, {injector: this.injector});
  }
}
```

### 销毁 effect

当您创建一个 effect 时，当它的封闭上下文被销毁时，它将自动被销毁。
这意味着当组件被销毁时，在组件中创建的 effect 也会被销毁。
指令、服务等内部的 effect 也是如此。

effect 返回一个 `EffectRef`，可以通过`.destroy()`操作手动销毁它们。
这也可以与`manualCleanup`选项结合使用，以创建一个持续到手动销毁的效果。
当不再需要这些效果时，要小心清理它们。

## 高级的主题

### 信号相等函数

在创建信号时，你可以有选择地提供一个相等函数，用于检查新值是否与之前的值不同。

```ts
import _ from "lodash";

const data = signal(["test"], { equal: _.isEqual });

// Even though this is a different array instance, the deep equality
// function will consider the values to be equal, and the signal won't
// trigger any updates.
data.set(["test"]);
```

等式函数可以提供给可写信号和可计算信号。

对于可写信号，`.mutate()`不会检查是否相等，因为它会在不产生新引用的情况下改变当前值。

### 不跟踪依赖关系的读取

很少情况下，你可能希望在 _不_ 创建依赖的情况下，执行在响应式函数中读取信号的代码，例如`computed`或`effect`。

例如，假设当 `currentUser` 改变时，应该记录 `counter` 的值。创建一个读取两个信号的“效果”:

```ts
effect(() => {
  console.log(`User set to `${currentUser()}` and the counter is ${counter()}`);
});
```

当' currentUser '或' counter '发生变化时，此示例记录一条消息。
然而，如果效果应该只在' currentUser '更改时运行，那么' counter '的读取只是偶然的，并且更改为' counter '不应该记录新消息。

你可以用' untracked '调用它的 getter 来阻止信号被读取:

```ts
effect(() => {
  console.log(`User set to `${currentUser()}` and the counter is ${untracked(counter)}`);
});
```

当 effect 需要调用一些不应被视为依赖的外部代码时，' untracked '也很有用:

```ts
effect(() => {
  const user = currentUser();
  untracked(() => {
    // If the `loggingService` reads signals, they won't be counted as
    // dependencies of this effect.
    this.loggingService.log(`User set to ${user}`);
  });
});
```

### Effect 清理函数

Effect 可能会启动长时间运行的操作，如果 Effect 被销毁或在第一个操作完成之前再次运行，则应该取消这些操作。
当你创建一个 Effect 时，你的函数可以选择接受一个`onCleanup`函数作为它的第一个参数。
这个`onCleanup`函数允许您注册一个回调函数，该回调函数将在下次运行 Effect 开始之前调用，或者在 Effect 被销毁时调用。

```ts
effect((onCleanup) => {
  const user = currentUser();

  const timer = setTimeout(() => {
    console.log(`1 second ago, the user became ${user}`);
  }, 1000);

  onCleanup(() => {
    clearTimeout(timer);
  });
});
```
