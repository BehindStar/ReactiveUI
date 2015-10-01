# WhenAny + WhenAnyValue 的含义

ReactiveUI 的核心功能之一是能够将属性转换为 Observable，通过 `WhenAny`；将 Observable 转换为属性，通过 `ToProperty`。

### WhenAny 有什么用

* 观察单个属性：

```cs
this.WhenAny(x => x.Foo, x => x.Value)
```

* 观察多个属性：

```cs
// This signals when any one of these three properties change
this.WhenAny(x => x.Red, x => x.Green, x => x.Blue,
    (r,g,b) => new Color(r.Value, g.Value, b.Value));
```

* 观察嵌套属性：

```cs
this.WhenAny(x => x.Foo.Bar.Baz, x => x.Value);
```

### WhenAny的不同类型

有几种 WhenAny 的变体都十分有用：

* **WhenAny** - 提供一个属性列表，和决定初始值的选择器

* **WhenAnyValue** - 提供一个属性，可以获得最新值

* **WhenAnyObservable** - 提供一个 Observable 属性列表，这些属性将会使用 `CombineLatest` 组合起来。这对于在视图中订阅命令非常有用：

```cs
// 不要这样——订阅是在 *旧* 对象上面，但是已经被替换，所以现在他不会收到任何东西
this.ViewModel.SomeCommand
    .Subscribe(_ => Console.WriteLine("Hello"));
ViewModel = new ToasterViewModel();

// 替代
this.WhenAnyObservable(x => x.ViewModel.SomeCommand)
    .Subscribe(_ => Console.WriteLine("Hello"));

// 依然有效
ViewModel = new ToasterViewModel();
```

WhenAny 有一些特殊之处，有必要知道：

### Expression Evaluation Semantics

考虑这段代码：

```cs
this.WhenAny(x => x.Foo.Bar.Baz, _ => "Hello!")
    .Subscribe(x => Console.WriteLine(x));

// 示例 1# WhenAny + WhenAnyValue 的含义

ReactiveUI 的核心功能之一是能够将属性转换为 Observable，通过 `WhenAny`；将 Observable 转换为属性，通过 `ToProperty`。

### WhenAny 有什么用

* 观察单个属性：

```cs
this.WhenAny(x => x.Foo, x => x.Value)
```

* 观察多个属性：

```cs
// 三个属性的任何一个发生了变化，就会触发
this.WhenAny(x => x.Red, x => x.Green, x => x.Blue,
    (r,g,b) => new Color(r.Value, g.Value, b.Value));
```

* 观察嵌套属性：

```cs
this.WhenAny(x => x.Foo.Bar.Baz, x => x.Value);
```

### WhenAny的不同类型

有几种 WhenAny 的变体都十分有用：

* **WhenAny** - 提供一个属性列表，和一个选择器用于决定初始值

* **WhenAnyValue** - 提供一个属性，可以获得最新值

* **WhenAnyObservable** - 提供一个 Observable 属性列表，这些属性将会使用 `CombineLatest` 组合起来。这对于在视图中订阅命令非常有用：

```cs
// 不要这样——订阅是在 *旧* 对象上面，但是该对象已经被替换，所以现在它不会收到任何东西
this.ViewModel.SomeCommand
    .Subscribe(_ => Console.WriteLine("Hello"));
ViewModel = new ToasterViewModel();

// 替代
this.WhenAnyObservable(x => x.ViewModel.SomeCommand)
    .Subscribe(_ => Console.WriteLine("Hello"));

// 依然有效
ViewModel = new ToasterViewModel();
```

WhenAny 有一些特殊之处，有必要知道：

### Expression Evaluation Semantics

考虑这段代码：

```cs
this.WhenAny(x => x.Foo.Bar.Baz, _ => "Hello!")
    .Subscribe(x => Console.WriteLine(x));

// 示例 1
this.Foo.Bar.Baz = null;
>>> Hello!

// 示例 2: 没有输出！
this.Foo.Bar = null;

// 示例 3
this.Foo.Bar = new Bar() { Baz = "Something" };
>>> Hello!
```

`WhenAny` 仅在读取指定表达式不会抛出空引用异常时，才发送通知。在示例 1 中，即使 Baz 为 `null`，但是由于表达式可以求值，依然可以得到通知。

在示例2中，对 `this.Foo.Bar.Baz` 求值不会返回 `null` ，而是崩溃了。`WhenAny` 因此抑制了通知生成。为 `Bar` 设置新值获得了新通知。

### Distinctness

`WhenAny` 仅在 *最终值* **发生改变** 时发出通知。即便是结果改变是由于表达式链中间值的原因。例子可以说明这一点：

```cs
this.WhenAny(x => x.Foo.Bar.Baz, _ => "Hello!")
    .Subscribe(x => Console.WriteLine(x));

// 示例 1
this.Foo.Bar.Baz = "Something";
>>> Hello!

// 示例 2: 没有输出！
this.Foo.Bar.Baz = "Something";

// 示例 3: 依然没有
this.Foo.Bar = new Bar() { Baz = "Something" };

// 示例 4: 结果改变了，有输出
this.Foo.Bar = new Bar() { Baz = "Else" };
>>> Hello!
```

### 更多

* `WhenAny` 在你订阅时，就会给你当前值 —— 这是有效的行为主体（BehaviorSubject）。

* `WhenAny` 是一个冷 Observable，在最后直接连接到 UI 组件的事件。对于诸如依赖属性这样的事件， 这可能是一个进行优化的（小）地方，通过 `Publish`。

this.Foo.Bar.Baz = null;
>>> Hello!

// 示例 2: Nothing printed!
this.Foo.Bar = null;

// 示例 3
this.Foo.Bar = new Bar() { Baz = "Something" };
>>> Hello!
```

`WhenAny` 仅在读取指定表达式不会抛出空引用异常时，才发送通知。在示例 1 中，即使 Baz 为 `null`，但是由于表达式可以求值，依然可以得到通知。

在示例2中，对 `this.Foo.Bar.Baz` 求值不会返回 `null` ，会导致崩溃。`WhenAny` 因此抑制了通知生成。为 `Bar` 设置新值获得了新通知。

### Distinctness

`WhenAny` 仅在 *最终值* **发生改变** 时发出通知。即便是结果改变是由于表达式链中间值的原因。例子可以说明这一点：

```cs
this.WhenAny(x => x.Foo.Bar.Baz, _ => "Hello!")
    .Subscribe(x => Console.WriteLine(x));

// 示例 1
this.Foo.Bar.Baz = "Something";
>>> Hello!

// 示例 2: 没有输出！
this.Foo.Bar.Baz = "Something";

// 示例 3: 依然没有
this.Foo.Bar = new Bar() { Baz = "Something" };

// 示例 4: 结果改变了，有输出
this.Foo.Bar = new Bar() { Baz = "Else" };
>>> Hello!
```

### 更多

* `WhenAny` 在你订阅时，就会给你当前值 —— 这是有效的行为主体（BehaviorSubject）。

* `WhenAny` 是一个冷 Observable，在最后直接连接到 UI 组件的事件。对于诸如依赖属性这样的事件， 这可能是一个进行优化的（小）地方，通过 `Publish`。
