# ReactiveCommand

每个 MVVM 框架的核心目标是提供一个 `ICommand` 的实现。该接口代表了构成一个视图模型的两个主要部分——属性和命令——之一。为了保证实现 MVVM + Rx，ReactiveUI 提供了自己的 `ICommand` 实现，叫做 `ReactiveCommand`，它与大部分的其他实现有一些不同。

命令表示在用户界面中所采取的离散动作——复制、打开、或者确定都是命令的例子。通常这些命令绑定到执行命令的控件，比如按钮。Cocoa 通过 [Target Action
Framework](https://developer.apple.com/library/ios/documentation/general/conceptual/CocoaEncyclopedia/Target-Action/Target-Action.html)描述了这一概念。

许多命令都由用户直接调用，但一些操作也可以通过命令建模，尽管主要是通过编程调用。比如，许多与周期性的加载或刷新资源相关的代码，都可以很好的使用命令建模。

### 基础

由于实际上调用命令就代表一个事件，ReactiveCommand 自身是一个 `IObservable<object>`。object 通过 IObservable 传递，是 `ICommand` 的 `Execute` 方法的命令参数：

```cs
var command = ReactiveCommand.Create();

command.Subscribe(x => this.Log().Info("The number is {0}", x));

command.Execute(4);
>>> The number is 4
```

虽然 ReactiveCommand 支持命令参数，但是不建议使用，通常总是传递 `null` 给 Execute 方法。相比于使用参数，应该使用视图模型上的属性。

由于 ReactiveCommand 是一个 Observable，所有的 Rx 操作符都可以使用。这是一些特定的例子：

```cs

// 注意：这是用于演示目的，异步 ReactiveCommand 章节有更好的方式
LoadTweets
    .Where(_ => IsLoggedIn == true)
    .SelectMany(async x => await FetchTweets())
    .ObserveOn(RxApp.MainThreadScheduler)
    .Subscribe(x => LoadedTweets = x);

// 在命令被调用 *或* 窗体激活时刷新
shouldRefreshTweets = Observable.Merge(
    this.Events().ActivatedObs.Select(_ => Unit.Default),
    this.WhenAnyObservable(x => x.ViewModel.Refresh).Select(_ => Unit.Default));

shouldRefreshTweets
    .Where(x => this.ViewModel != null)
    .Subscribe(_ => ViewModel.RefreshData());
```

### 通过 Observable 定义 CanExecute

到目前为止，我们创建的所有命令，总是可以执行——它们的 `CanExecute` 返回 true。要指定一个命令是否可以执行，相较于使用 `Func<object, bool>`，我们使用 `IObservable<bool>`。因为要描述的不仅仅是命令是否可以执行，还有值什么时候发生变化，无需提供 `CanExecuteChanged` 的实现。

注意在 ReactiveCommand 中传递给 `CanExecute` 的参数将被忽略的。这是因为其与 `CanExecuteChanged` 的概念根本不兼容：如果 `CanExecute(bar)` 为 `true` 且 `CanExecute(baz)` 为
`false`，那么什么时候触发 `CanExecuteChanged`呢？

可以使用一个 `Subject<bool>` 对象给 CanExecute 传递信息，其是一个 Observable 对象，你可以手动控制它。工作方式如下：

```cs
var commandCanExecute = new Subject<bool>();
var command = ReactiveCommand.Create(commandCanExecute);

commandCanExecute.OnNext(false);
command.CanExecute(null);
>>> false

commandCanExecute.OnNext(true);
command.CanExecute(null);
>>> true
```

### 组合 WhenAny 和 CanExecute

通常情况下，一个更为合适的 `CanExecute` 依赖于视图模型上的其他属性。由于我们想在属性变化时得到通知，我们使用 `WhenAny` 方法，并使用 `Select` 得到一个布尔值。例如：

```cs
// 是否可以发布 Tweet，依赖于用户是否输入且输入足够的短
PostTweet = ReactiveCommand.Create(
    this.WhenAnyValue(x => x.TweetContents)
        .Select(x => !String.IsNullOrWhitespace(x) && x.Length < 140));

// 也可以使用内置于 WhenAny 的选择器，不使用额外的 Select
OkButton = ReactiveCommand.Create(
    this.WhenAny(x => x.Red, x => x.Green, x => x.Blue,
        (r,g,b) => r.Value != null && g.Value != null && b.Value != null));
```

差不多所有命令都可以使用这种模式来决定什么时候可以执行。由于命令与属性关联在一起，许多验证类型的任务可以通过这种方式完成。

### 通过 WhenAnyObservable 在视图中监听命令

不像传统的 `ICommand` 实现，ReactiveCommand 的 `Executed` 信号可以被多个对象监听。这对于解耦来说**非常**有用，因为视图现在可以监听视图模型，并执行视图相关的代码，比如设置控件焦点或滚动位置。

你可能会简单的这样实现：`ViewModel.SomeCommand.Subscribe(x => ...)`，但是这个代码在视图模型改变（指的是换了一个模型）时会失效——你将会订阅到错误的命令，可能永远不会触发。`WhenAnyObservable` 方法可以解决这个问题：

```cs
// 错误代码
this.ViewModel.ClearMessageText
    .Subscribe(x => MessageTextBox.GetFocus());

// 这样做，自动处理视图模型为空和变化
this.WhenAnyObservable(x => x.ViewModel.ClearMessageText)
    .Subscribe(x => MessageTextBox.GetFocus());
```

### 组合命令

创建一个简单调用几个其他命令的命令有时候非常有用。ReactiveCommand 的 `CreateCombined` 方法可以实现。使用该方法的好处是，新命令的 `CanExecute` 反映了子命令的**与**结果（例如，如果任何子命令不能被调用，那么父命令也不能调用）。这对于某个命令有异步操作时非常有用。

```cs
RefreshUsers.Subscribe(_ => this.Log().Info("Refreshing Users!"));
RefreshLists.Subscribe(_ => this.Log().Info("Refreshing Lists!"));

RefreshAll = ReactiveCommand.CreateCombined(
    RefreshUsers, RefreshLists);

RefreshAll.Execute(null);

>>> Refreshing Users!
>>> Refreshing Lists!
```

### 通过 Observable 调用和创建命令

有一些框架内置的简便方法用于调用命令。任何 Observable 都可以作为通过 `InvokeCommand` 调用命令的信号：

```cs
// 在用户按下 ESC 时调用退出命令
// 在执行前自动进行 CanExecute 检查
this.Events().KeyUpObs
    .Where(x => x.EventArgs.Key == Key.Escape)
    .InvokeCommand(this, x => x.ViewModel.Close);
```

另一个方法 `ToCommand` 允许你从 `IObservable<bool>` 对象直接创建命令。上面的 `CanExecute` 例子可以简写为：

```cs
PostTweet = this.WhenAny(x => x.TweetContents, 
        x => !String.IsNullOrWhitespace(x.Value) && x.Value.Length < 140)
    .ToCommand();

OkButton = this.WhenAny(x => x.Red, x => x.Green, x => x.Blue,
        (r,g,b) => r.Value != null && g.Value != null && b.Value != null)
    .ToCommand();
```
