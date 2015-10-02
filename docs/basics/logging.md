# 日志

ReactiveUI 也有自己的日志框架，可以用来调试你的程序和 ReactiveUI。你可能会问，“真糟糕，又是一个日志框架？”。RxUI 自己实现日志框架是为了移植性——没有一个主流通用的日志框架支持所有 ReactiveUI 支持的平台，并且许多是面向服务的框架且不适合用于简单的移动应用程序的日志记录。

### this.Log() 和 IEnableLogger

ReactiveUI 的记录器用起来有一点麻烦，和其他框架相比 —— 它的思路来自于 Rails 的记录器。要使用它，需要让你的类实现 `IEnableLogger` 接口：

```cs
public class MyClass : IEnableLogger
{
    // IEnableLogger 实际上不需要实现任何东西
}
```

现在，你可以在你的类上调用 `Log` 方法了。由于扩展方法的工作机制，你必须前置一个 `this`：

```cs
this.Log().Info("Downloaded {0} tweets", tweets.Count);
```

日志有**五个**级别： `Debug`， `Info`， `Warn`， `Error`，和 `Fatal`。此外，还有一些特殊的方法用于记录异常 —— 比如，`this.Log().InfoException(ex, "Failed to post the message")`。

这招不能在静态方法中使用，必须使用另一个备用的方法，`LogHost.Default.Info(...)`。

### 调试 Observable

ReactiveUI 有一些帮助类用于调试 IObservable。最简单的方法是 `Log`，可以记录 Observable 发生的异常：

```cs
// 注意: 由于 Log 实际上和 Rx 操作符如 Select 或 Where 类似，
// 所以在被订阅之前不会记录任何东西。
this.WhenAny(x => x.Name, x => x.Value)
    .SelectMany(async x => GoogleForTheName(x))
    .Log(this, "Result of Search")
    .Subscribe();
```

另一个用于调试 Observable 的方法是 `LoggedCatch`。这个方法做的工作和 Rx 的 `Catch` 操作符一样，除此之外还记录异常到记录器。例如：

```cs
var userAvatar = await FetchUserAvatar()
    .LoggedCatch(this, Observable.Return(default(Avatar)));
```

### 配置记录器

要配置记录器，注册一个 `ILogger` 的实现（有一些内置的实现，比如 `DebugLogger`）。这个例子演示了如何以自定义级别使用内置记录器：

```cs
// 只需要记录错误
var logger = new DebugLogger() { LogLevel = LogLevel.Error };
RxApp.MutableResolver.RegisterConstant(logger, typeof(ILogger));
```

如果真的需要控制日志记录的方式，需要实现 `IFullLogger`，其允许你控制每个日志重载。

### 向 Visual Studio 的 IntelliTrace 发送日志信息

1. 修改 Visual Studio 的 IntelliTrace 设置， 通过 Debug>IntelliTrace>Open IntelliTrace Settings>IntelliTrace Events>Tracing> 选中除了 Assertion under tracing 之外的所有选项。如果保持默认设置，就不能获得 Debug 级别跟踪 —— 通过选中所有跟踪事件，可以获得 Debug 级别跟踪。

1. 确保安装了 `reactiveui-nlog` 包到你的单元测试程序集（如果你使用 Windows Store Test Library 话，真是不幸；但是一个 “正常” 单元测试库没问题）

1. 添加一个 nlog.config 文件到你的单元测试你项目。 ***确保你设置了 "复制到输出文件夹" 属性为 "如果较新则复制" 或 "总是复制" ***。如果是默认配置 "不复制"，那么 NLog 找不到配置文件，将不会知道要记录到跟踪监听器。

1. 这是 `nlog.config` 文件的内容

```xml
<?xml version="1.0" encoding="utf-8" ?>
<nlog xmlns="http://www.nlog-project.org/schemas/NLog.xsd"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" >
  <targets>
    <target name="trace" xsi:type="trace"  layout="RxUI:${message}"/>
  </targets>
  <rules>
    <logger name="ReactiveUI.*"  writeTo="trace" />
  </rules>
</nlog>
```

1. 在单元测试的入口注册 NLogger ：

``` cs
var logManager = RxApp.MutableResolver.GetService<ILogManager>();
RxApp.MutableResolver.RegisterConstant(logManager.GetLogger<NLogLogger>(),typeof(IFullLogger));   
```

*提示: 一个过滤 IntelliTrace 视图只显示 ReactiveUI 的事件的简单办法是，输入 RxUI 到 IntelliTrace 窗体的搜索框*

