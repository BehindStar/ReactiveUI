# 依赖解析

依赖解析是核心框架内置的功能，允许库和 ReactiveUI 自身使用其他库的类，而不需要直接引用它们。这对于跨平台应用来说十分有用，因为依赖解析允许可移植代码使用不可移植的 API，只要这些 API 可以通过一个接口描述。

ReactiveUI 的依赖解析更正确的叫法应该是服务定位模式。仔细的考虑要如何使用 API，因为这能更有效的使用 API，并让代码更容易测试。如果不正确的使用 API 的话，代码将很难测试和理解，因为 [解析器自身可以有效的成为类的一部分](http://blog.ploeh.dk/2010/02/03/ServiceLocatorisanAnti-Pattern/)，但是是以一种隐式和不明显的方式。

### 使用依赖解析

ReactiveUI 提供了如下方法用于解析服务（注意这是一个简化版本，不是实际定义）：

```cs
public interface IDependencyResolver
{
    // 返回最近注册到该类型和约定的服务
    T GetService<T>(string contract = null);

    // 返回所有注册到该类型和约定的服务
    IEnumerable<T> GetServices<T>(string contract = null)
}
```

给定类型 T （通常是一个接口），可以获得一个 T 的实现。

If the T registered is very common ("string" for example), or you want
to distinguish by a method other than type, you can use the "contract"
parameter which is an arbitrary key that you provide.

当前 ReactiveUI 使用的解析器（你的应用也应该使用这个），是由 `RxApp.DependencyResolver` 提供的。

### 注册新依赖


`RxApp.DependencyResolver` 的默认实现也实现了另一个接口（通过属性 `RxApp.MutableResolver` 访问）：

```cs
public interface IMutableDependencyResolver : IDependencyResolver
{
    void Register(Func<object> factory, Type serviceType, string contract = null);
}
```

该解析器允许为接口注册新实现。这通常在程序启动时进行（Cocoa ，在 `AppDelegate`中，WPF，在 `App` 中）。

这个设计似乎过于简单，但是实际上，已经表现了那些将会在桌面、移动应用中使用的，最有用的生命期范围。例如：

```cs
var r = RxApp.MutableResolver;

// 每次创建一个新实例
r.Register(() => new FooBar(), typeof(IFooBar));

// 返回单例
var foobar = new FooBar();
r.Register(() => foobar, typeof(IFooBar));

// 返回单例，但是在第一次请求时才创建
var foobar = new Lazy<FooBar>();
r.Register(() => foobar.Value, typeof(IFooBar));
```

### 通用跨平台模式

依赖解析对于将存在于特定平台代码的逻辑转移到跨平台代码中非常有用。首先，定义一个需要的接口——这个例子用于演示，不是最佳实践。

```
public interface IYesNoDialog
{
    // 是的话返回 'true' ，不是的话返回 'false'。
    IObservable<bool> Prompt(string title, string description);
}
```

现在可以在视图模型中使用这个接口了：

```cs
public class MainViewModel
{
    public ReactiveCommand<Object> DeleteData { get; protected set; }

    public MainViewModel(IYesNoDialog dialogFactory = null)
    {
        // 如果构造函数没有提供自己的实现，
        // 那么从解析器获取一个。
        // 这对于在测试中使用假实现非常有用。
        dialogFactory = dialogFactory ?? RxApp.DependencyResolver.GetService<IYesNoDialog>();

        var title = "Delete the data?";
        var desc = "Should we delete your important Data?";

        DeleteData = ReactiveCommand.CreateAsyncObservable(() => dialogFactory.Prompt(title, desc)
            .Where(x => x == true)
            .SelectMany(async x => DeleteTheData()));

	DeleteData.ThrownExceptions(ex => this.Log().WarnException(ex, "Couldn't delete the data"));
    }
}
```
现在，我们的实现可以在 iOS 和 Android 之间有很大不同了——一个 iOS 实现的例子：

```cs
public class AlertDialog : IYesNoDialog
{
    public IObservable<bool> Prompt(string title, string description)
    {
        var dlgDelegate = new UIAlertViewDelegateRx();
        var dlg = new UIAlertView(title, description, dlgDelegate, "No", "Yes");
        dlg.Show();

        return dlgDelegate.ClickedObs
            .Take(1)
            .Select(x => x.Item2 == 1);
    }
}
```

### ModernDependencyResolver 和解析器初始化

ReactiveUI 中，`IDependencyResolver` 的默认实现是一个叫做 `ModernDependencyResolver` 的类。要初始化该类或者其他 `IMutableDependencyResolver` 实现，调用 `InitializeResolver` 扩展方法。

```cs
var r = new MutableDependencyResolver();
r.InitializeResolver();
```

通着这 **不是必须的**，你应该使用默认解析器。本指南的高级章节将会描述如何连接第三方依赖解析框架。但是，读者应该放弃这个想法，并使用默认解析器。
