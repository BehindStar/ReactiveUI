## ReactiveUI 7.0 发布记录

Hi，好久不见，现在有一个有趣的新版本了。我们不得不花一些时间在内部，用于培训和指导将来的版本维护者。.NET 的生态系统改变了我们，我们没有持续集成系统。要有路线图和继任计划，绝不允许长期存在的分支。实现持续集成，并让你的发布过程简化到按一个按钮那么简单。

### 什么是ReactiveUI？
ReactiveUI 源自于函数响应编程，在此基础上诞生了ReactiveCocoa（Cocoa/Swift）框架。我们也在内部讨论过我们是否是一个框架，因为这个项目的核心实质上是Reactive Extension的一些扩展方法。

ReactiveUI项目7年前就开始了，现在已经老得可以上小学了，但不像青少年，它非常稳定。ReactiveUI已经成熟多年，成为一个坚实和精确的选择，来构建你的下一个应用程序。此外，从另一个框架迁移到ReactiveUI是非常容易的。

当阅读代码时，你会发现ReactiveUI在实现中并没有喧宾夺主，但我们一直抱有一些信念，这些信念是项目的基础和基础。

我们认为代码是人之间的沟通，也在计算机上运行。如果你为人类优化，那么在很长一段时间你的项目会变得更好。软件应该可以被其他人理解，这是非常重要的。

我们相信只有反应式扩展的力量允许你在一个可读的地方描述某个功能的想法。

仔细想想你的典型用户界面？到处都是业务逻辑，满是悲剧。与其告诉计算机如何做，不如定义计算机的工作，然后让他以自己的方式做。如果这听起来很奇怪，让我们向你介绍一下Microsoft Excel。

与其在代码的不同分支中让 ViewModel isLoading = true/false，不如像Microsoft Excel 一样，写一个=SUM(A1: B2) 这样的表达式。

Async/await 会像瘟疫一样到处传染。从现在开始解放你的代码库。

ReactiveUI在GitHub，Slack，Microsoft的产品中使用，并由来自世界各地不同公司的顾问提供支持。

### Rx 困难吗

一点也不。学习 Rx 是作为一个软件工程师最值得的事情。单元测试的起步很难，依赖注入也是如此。在这过程中学习到的方法会永远改变你，最好的知识与实现和语言无关。我们设计了ReactiveUI，所以你可以从一个async / await代码库缓慢地过渡到你感觉舒适的步伐。

## 变更内容

### ReactiveCommand 改进。

`ReactiveCommand` 再次被完全重写了（抱歉）。

* 取消了接口。 所有 `IReactiveCommand` 的用法都应该用 `ReactiveCommand` 替代。可能有一些类型信息（见下文）。
* 静态创建方法的变化：
	* 在调用 `CreateXxx` 方法时，_总是_需求提供执行逻辑，包括 “同步” 命令（比如那些由 Create 创建的命令）。因此，无需调用 `Create` 然后订阅，直接在调用 `Create` 的时候提供执行逻辑。   
    * 为了保持一致性，执行逻辑总是作为第一个参数。其他参数（`canExecute`， `scheduler`）是可选的。
	* `CreateAsyncObservable` 改为 `CreateFromObservable`。
	* `CreateAsyncTask` 改为 `CreateFromTask`。
* 在 `ReactiveCommand<TParam, TResult>` 中使用 `TParam` 指定参数类型。
	* 如果命令需要一个参数，无需使用 `object` 并进行转换了。只需要在创建命令的时候显式指定参数类型即可。当然，有必要的话依然可以使用 `object`，或者将其作为迁移的中间步骤。
* 显式实现 `ICommand` 。因此：
	* `ReactiveCommand` 的公开方法 `Execute` 是响应式的（其返回 `IObservable<TResult>`）。如果没订阅它的话，不会执行任何动作。
    * `CanExecuteObservable` 简写为 `CanExecute`
* `CanExecute` 和 `IsExecuting` 等 observable 现在是 behavioral 的了。这样，他们将为观察者总是提供最新的值（如果没有最新值，则提供默认值）。
* `RoutingState` 使用了新的实现。因此，它的所有命令都会受到影响。
* 移除了 `ToCommand` 扩展方法。该方法用于方便的创建 `IObservable<bool>`，并将其作为某个命令的 `canExecute` 。如果你使用了这些方法，可以用 ReactiveCommand 上的某个 CreateXxx 方法替代。

Old:

```cs
var canExecute = ...;
var someCommand = ReactiveCommand.Create(canExecute);
someCommand.Subscribe(x => /* 执行逻辑 */);

var someAsyncCommand1 = ReactiveCommand.CreateAsyncObservable(canExecute, someObservableMethod);
var someAsyncCommand2 = ReactiveCommand.CreateAsyncTask(canExecute, someTaskMethod);
```
someCommand.Execute();

New:

```cs
var canExecute = ...;
var someCommand = ReactiveCommand.Create(() => /* 执行逻辑 */);

var someAsyncCommand1 = ReactiveCommand.CreateAsyncObservable(someObservableMethod, canExecute);
var someAsyncCommand2 = ReactiveCommand.CreateAsyncTask(someTaskMethod, canExecute);

someCommand.Execute().Subscribe();
```

更多的细节，参见[这里](https://docs.reactiveui.net/en/user-guide/commands/index.html)

为了方便你的迁移，所有先前的类型都可以在 `ReactiveUI.Legacy` 命名空间中找到。注意，`RoutingState` 没有遗留版本，因此与其交互的所有代码都需要略微改变。

### 交互

`UserError` 被扩展和重绘了。我们称其为互动，你会喜欢它的。我们这样做的原因是，在非错误情景下使用 UserError 有点别扭。我们意识到人们需要的是一种通用的机制，可以通过一个视图模型询问问题，等待答案。它不必是一个错误，我们不是那么悲观！你可能要求确认文件删除，或者一些其他的东西。

从 `UserError` 迁移到交互架构，不能一一替换了事。在开始之前，先看看这些建议：

* 通读[文档](http://docs.reactiveui.net/en/user-guide/interactions/index.html)

* 确定你是否需要共享交互逻辑，如果是，将其定义到一个合适的位置（通常就是静态类）。

* 对于任何无需共享的交互逻辑，在你的视图模型上创建交互实例，并将其作为视图的一个公开属性。

* 通常需要相应的视图处理交互，通过调用视图模型公开的交互属性上的 `RegisterHandler` 方法。

* 视图模型调用交互对象上的 `Handle` 方法，传递一个输入值。

* Recover commands 不再内置。如果你需要在交互中使用此机制，你可以编写一个合适的类，将其作为交互的输入。 

为了方便你的迁移，所有先前的类型都可以在 `ReactiveUI.Legacy` 命名空间中找到。

### ToProperty

在先前的版本中，ToProperty 是懒惰的。就是说，除非有东西 `pulling` 目标属性，否则没有任何效果。这是出于性能的考虑，因为你可能解析昂贵的属性，但这只在特定情况下使用。

虽然这对性能有好处，但它经常令人困惑并且与预期背道而驰。因此，`ToProperty` 不在是懒惰的了，它立即订阅，以确保属性的值反映给对应的观察管道。但是，可以将 `true` 传递给 `deferSubscription` 参数来获得原始行为。

### 自动化

现在 ReactiveUI 可以自动构建和发布了。

因此，你、社区可以更快速的获得新版本了。

## 详情

此版本修复了 113个 的已知问题。

参见原文
