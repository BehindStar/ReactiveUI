# ToProperty 和输出属性

ReactiveUI 的一个核心功能是，能够将属性装换为 Observable，通过 `WhenAny`；将 Observable 转换为属性，通过 `ToProperty`。在 ReactiveUI 中，这些属性叫做 **输出属性**，他们是有效使用这个框架的很大一部分。

假设有一个颜色选择对话框，该对话框有4个属性：

* Red
* Green
* Blue
* FinalColor

第四个属性，`FinalColor` ，与其他的不同。他不是一个可读写属性，其值由上面三个属性一起决定。在其他框架中，它也是一个只读属性，通过代码中的多个部分设定。这是 UI 代码像意大利面条一样丑陋的开始。

ReactiveUI 允许你显式描述属性之间的依赖，以一种很难写错的方式。

### 基础用法

首先，需要使用 `ObservableAsPropertyHelper<T>` 类，创建一个输出属性。所有的输出属性代码都应该这样写：

```cs
ObservableAsPropertyHelper<Color> finalColor;
public Color FinalColor {
    get { return finalColor.Value; }
}
```

注意 `FinalColor` 没有写访问器。我们使用 Observable 来描述颜色如何变化，在构造函数中进行：

```cs
var colorValues = this.WhenAnyValue(x => x.Red, x => x.Green, x => x.Blue,
        (r,g,b) => new {r,g,b})
    .Select(x => new Color(x.r, x.g, x.b));

colorValues.ToProperty(this, x => x.FinalColor, out finalColor);
```

### 延迟观察

知道 `ToProperty` 的含义很重要——它与 `Subscribe` 在概念上类似：`Subscribe` 与 Observable 的关系，类似于 `foreach` 和 Enumerable。

从 ReactiveUI 6.0 开始，`ToProperty` 变 **懒惰** 了——它不进行 Subscribe ，直到第一次被请求（通常是视图绑定）时（通过 `Value`）。`ToProperty` 和下面的代码差不多：

```cs
// 这个与 ->
theObservable.ToProperty(this, x => x.Foo, out foo);

//
// -> 这个 相似
//

var backingFoo = theObservable
    .Do(x => Foo = x)
    .Publish()
    .RefCount();

public string Foo {
    get {
        backingDisposable = backingDisposable ?? backingFoo.Subscribe();
        return foo;
    }
}
```

你可能觉得 ToProperty “丢失” 事件是个问题，最简单的解决方式是使用 `Concat` 进行处理。例如：

```cs
// 存在问题，因为 CanExecuteObservable 是 Hot（不管有没有订阅者，都发送数据）
someCommand.CanExecuteObservable
    .ToProperty(this, x => x.CanExecute, out canExecute);

// 让 Observable 变得 cold （当有一个订阅者时，才送数据）
Observable.Defer(() => Observable.Return(someCommand.CanExecute(null)))
    .Concat(someCommand.CanExecuteObservable)
    .ToProperty(this, x => x.CanExecute, out canExecute);
```
