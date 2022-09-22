# Angular FormArray: Complete Guide

21 JAN 2022

学习如何通过在运行时添加或删除表单控件，使用 FormArray 构建动态 Angular 表单。
构建一个可就地编辑的数据表。

在这篇文章中，你将学到所有你需要知道的关于 Angular FormArray 构造的知识，它在 Angular 的响应式表单中可用。

我们将学习什么是 Angular 的 FormArray，它与普通的 FormGroup 有什么区别，什么时候使用它以及为什么要使用它。

我们将给出一个通用用例的示例，如果没有 FormArray 就很难实现这个用例:一个每行有多个表单控件的可编辑表，用户可以根据需要在其中添加或删除新的可编辑行。

## 表的内容

在这篇文章中，我们将讨论以下主题:

- 什么是 Angular 的 FormArray?
- formmarray 和 FormGroup 的区别是什么?
- FormArray API
- 使用 FormArray 创建一个就地可编辑表
- formArrayName 指令
- 什么时候使用 Angular 的 FormArray 和 FormGroup?
- 总结

这篇文章是我们正在进行的 Angular 表单系列的一部分，你可以在这里找到所有的文章。

废话不多说，让我们开始学习我们需要知道的关于 Angular FormArray 的一切!

## 什么是 Angular 的 FormArray?

在 Angular 响应式表单中，每个表单都有一个表单模型，通过编程的方式定义，使用的是 FormControl 和 FormGroup API，或者使用更简洁的 FormBuilder API，我们将在本指南中使用它。

在大多数情况下，表单的所有表单字段都是预先知道的，所以我们可以使用 FormGroup 为表单定义一个静态模型:

```ts
form = this.fb.group({
  title: [
    "",
    {
      validators: [
        Validators.required,
        Validators.minLength(5),
        Validators.maxLength(60),
      ],
      asyncValidators: [courseTitleValidator(this.courses)],
      updateOn: "blur",
    },
  ],
  releasedAt: [new Date(), Validators.required],
  category: ["BEGINNER", Validators.required],
  downloadsAllowed: [false, Validators.requiredTrue],
  longDescription: ["", [Validators.required, Validators.minLength(3)]],
});
```

正如我们所看到的，使用 FormGroup，我们可以定义一组相关的表单控件、它们的初始值和表单验证规则，并为表单的每个字段提供属性名。

这是可能的，因为这是一个静态表单，具有预先定义的所有已知字段的数量，这是表单最常见的情况。

但是如果是其他更高级但仍然经常遇到的情况，即表单更加动态，并且不是所有表单字段都预先知道(如果有的话)的情况呢?

想象一个动态表单，其中表单控件由用户添加或删除，具体取决于它与 UI 的交互。

一个例子就是根据来自后端的数据完全动态构建的表单!

动态表单的另一个更常见的例子是就地可编辑表，其中用户可以添加或删除包含多个可编辑表单控件的行:

![Angular 表单数组示例——一个可就地编辑的表](https://angular-university.s3-us-west-1.amazonaws.com/blog-images/angular-form-array/angular-form-array-example.png)

在此动态表单中，用户可以使用 add 和 Delete 按钮向表单添加或删除新控件。

每当用户单击 Add 按钮时，一个新的教训行将添加到包含两个新表单控件的表单中。

我们将在本指南中使用 FormArray 实现这个例子。
但主要问题是:为什么不能使用 FormGroup 实现这个表单?

## FormArray 和 FormGroup 的区别是什么?

与我们最初通过调用 fb.group() API 使用 FormGroup 定义表单模型的示例不同，在就地可编辑表的情况下，我们不知道前面的表单控件的数量。

这是因为我们无法预先知道表的行数。
用户可能使用 add 按钮添加未知数量的行，甚至可能在中途使用 delete class 按钮删除它们。

如果不知道确切的行数，我们就无法使用 FormGroup 定义表单模型。
另外，也很难给字段指定预定义的名称。

但是我们可以使用 FormArray 来为这个就地可编辑表定义一个表单模型。

就像 FormGroup 一样，FormArray 也是一个表单控件容器，它聚合其子组件的值和有效性状态。

但与 FormGroup 不同的是，FormArray 容器不需要我们预先知道所有的控件以及它们的名称。

实际上，FormArray 可以有不确定数量的表单控件，从 0 开始!然后可以根据用户与 UI 的交互方式动态添加和删除控件。

然后，每个控件在表单控件数组中都有一个数字位置，而不是唯一的名称。

可以使用 FormArray API 在运行时随时从表单模型中添加或删除表单控件。

## FormArray API

以下是 FormArray API 中最常用的方法:

- controls: 这是一个包含数组中所有控件的数组
- length: 这是数组的总长度
- at(index): 返回给定数组位置的表单控件
- push(control): 将新控件添加到数组的末尾
- removeAt(index): 移除数组的给定位置上的控件
- getRawValue(): 通过控件获取所有表单控件的值。每个控件的值属性

现在让我们学习如何使用这个 API 来实现就地可编辑的数据表。

## 使用 FormArray 创建一个就地可编辑表

让我们先定义一个响应式表单模型，使用 FormBuilder API:

```ts
@Component({
selector: 'form-array-example',
templateUrl: 'form-array-example.component.html',
styleUrls: ['form-array-example.component.scss']
})
export class FormArrayExampleComponent {

    form = this.fb.group({
        ...
        other form controls ...
        lessons: this.fb.array([])
    });

    constructor(private fb:FormBuilder) {}

    get lessons() {
      return this.form.controls["lessons"] as FormArray;
    }

}
```

正如我们所看到的，除了可编辑数据表本身之外，我们没有向表单添加任何其他控件。

这只是为了保持示例的简单性，但是如果需要的话，没有什么可以阻止我们添加任何其他表单属性。

在我们的例子中，可编辑表中的所有控件都将在使用 fb.array() API 构建的 FormArray 实例中。

最初，FormArray 实例是空的，不包含任何表单控件，这意味着可编辑表最初是空的。

我们向组件中添加了一个用于 classes 属性的 getter，以便以一种简单且类型安全的方式访问 FormArray 实例。

另外，请注意在前面显示的可编辑表截图中，表中的每一行都包含两个控件:一个课程标题字段和一个课程级别(或难度)字段。

我们希望这些字段成为该组件的父表单的一部分，并影响其有效性状态。
如果某一课的标题出现了错误，那么整个表格应该被认为是无效的。

为此，我们将为每个表行创建一个 FormGroup，并将向其中添加两个表单字段，以及各自的表单控件验证器。

Dynamically adding controls to a FormArray

为了向表中添加新行，我们需要在组件模板中添加一个 add Lesson 按钮:

```html
<button mat-mini-fab (click)="addLesson()">
  <mat-icon class="add-course-btn">add</mat-icon>
</button>
```

当点击这个按钮时，我们将触发以下代码:

```ts
addLesson() {
    const lessonForm = this.fb.group({
        title: ['', Validators.required],
        level: ['beginner', Validators.required]
    });

    this.lessons.push(lessonForm);
}
```

如我们所见，为了向表中添加一行，我们首先创建一个简单表单的表单模型，每一行只有两个字段:课程标题和难度。

然后我们把这个教训行 FormGroup，它本身也是一个表单控件，我们使用 push API 将它添加到 FormArray 的最后一个位置。

记住，我们可以向 FormArray 添加任何表单控件，这也包括表单组，表单组本身也是控件。

Dynamically removing controls from a FormArray

如果您注意到在前面显示的可编辑表截图中，每个教训行都有一个关联的删除图标，用户可以使用该图标删除整个教训行。

下面是这个按钮的点击处理程序:

```ts
deleteLesson(lessonIndex: number) {
    this.lessons.removeAt(lessonIndex);
}
```

为了删除一个教训行，我们所要做的就是使用 removeAt API 在给定的行索引处从 FormArray 中删除相应的 FormGroup。

注意，在添加教训和删除教训按钮中，要添加或删除行，只需从表单模型中添加或删除控件。

这是因为我们的 UI 模型是通过循环 FormArray 元素和添加相应的表单控件来构建的。

## formArrayName 指令

为了结束我们的练习，我们现在将展示可编辑表组件的完整代码，并回顾模板。
让我们从组件代码开始:

```ts
@Component({
    selector: 'form-array-example',
    templateUrl: 'form-array-example.component.html',
    styleUrls: ['form-array-example.component.scss']
})
export class FormArrayExampleComponent {

    form = this.fb.group({
        ...
        other form controls ...
        lessons: this.fb.array([])
    });

    constructor(private fb:FormBuilder) {}

    get lessons() {
      return this.form.controls["lessons"] as FormArray;
    }

    addLesson() {
      const lessonForm = this.fb.group({
        title: ['', Validators.required],
        level: ['beginner', Validators.required]
      });
      this.lessons.push(lessonForm);
    }

    deleteLesson(lessonIndex: number) {
      this.lessons.removeAt(lessonIndex);
    }

}
```

下面是可编辑表组件模板的样子:

```html
<h3>Add Course Lessons:</h3>
<div class="add-lessons-form" [formGroup]="form">
  <ng-container formArrayName="lessons">
    <ng-container *ngFor="let lessonForm of lessons.controls; let i = index">
      <div class="lesson-form-row" [formGroup]="lessonForm">
        <mat-form-field appearance="fill">
          <input matInput formControlName="title" placeholder="Lesson title" />
        </mat-form-field>
        <mat-form-field appearance="fill">
          <mat-select formControlName="level" placeholder="Lesson level">
            <mat-option value="beginner">Beginner</mat-option>
            <mat-option value="intermediate">Intermediate</mat-option>
            <mat-option value="advanced">Advanced</mat-option>
          </mat-select>
        </mat-form-field>
        <mat-icon class="delete-btn" (click)="deleteLesson(i)">
          delete_forever</mat-icon
        >
      </div>
    </ng-container>
  </ng-container>

  <button mat-mini-fab (click)="addLesson()">
    <mat-icon class="add-course-btn">add</mat-icon>
  </button>
</div>
```

现在让我们分析一下这里发生了什么:

- 这个可编辑表是用常用的 Angular Material 组件实现的
- 我们将 formGroup 指令应用到一个表单容器，并将其链接到组件的父表单
- 这意味着父表单将包含表单的所有子控件，包括用户创建的每一行的每一个控件
- 在父表单中，我们应用 formArrayName 指令，它将容器元素链接到表单的 classes 属性
- 这个指令允许 FormArray 实例跟踪其子组件的值和有效性状态
- 然后我们使用 ngFor 循环使用 FormArray 课程的表单控件
- FormArray 的每个控件本身都是一个表单组，包含两个行控件(标题和级别)
- 然后，我们使用 formGroup 指令为每个表行定义一个嵌套表单，其中包含两个教训行控件
- 然后，我们使用 formControlName 指令将课程字段绑定到模板，就像我们通常在任何响应式表单中所做的那样

正如我们所看到的，我们的可编辑表表单只是一个带有嵌套子表单列表的表单，每一行一个表单。
每个表行表单内部包含两个控件。

有了这个，我们的就地可编辑表就完全实现了!用户可以自由地在表单中添加和删除教训行。

只有当用户添加的每一行都填写了有效值时，父表单才被认为是有效的。

## 总结

让我们快速总结一下我们所学到的关于 FormArray 的所有知识，并对 FormGroup 进行最后的比较。

正如我们所看到的，FormArray 构造非常强大，在我们想要以更动态的方式构建表单模型的情况下尤其有用。

### 什么时候使用 Angular 的 FormArray 和 FormGroup?

在构建 Angular 表单的模型时，大多数时候我们都希望使用 FormGroup 而不是 FormArray，因此这应该是默认选择。

正如我们所看到的，FormArray 容器对于那些我们不知道前面的表单控件的数量或它们的名称的罕见情况是理想的。

FormArray 容器对于更动态的情况非常理想，在这种情况下表单的内容通常是在运行时定义的，这取决于用户交互甚至后端数据。

在我们的例子中，我们使用 FormArray 来实现就地可编辑表，因为我们相信这是它最常见的用例。

在表示例中，FormArray 中的控件是 FormGroup 实例，包含表单控件本身，但请注意，这不是强制性的。

FormArray 可以包含任何类型的控件，其中包括普通表单控件、FormGroup 实例甚至其他 FormArray 实例。

使用 FormArray，你可以在 Angular 中实现各种高级动态表单场景，但请记住，除非你真的需要
FormGroup 是大多数情况下的正确选择。

我希望你喜欢这篇文章，如果你想了解更多关于 Angular 表单的知识，我们建议你查看 Angular Forms In Depth 课程，里面详细介绍了各种高级表单主题(包括 FormArray)。
