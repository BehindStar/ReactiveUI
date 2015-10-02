# 消息总线

与其他 MVVM 框架一样，ReactiveUI 包含一个消息总线模式的实现。其允许在代码之间发送和接收消息，不需要直接访问彼此。

ReactiveUI 中消息总线只有一个唯一的属性（`MessageBus.Current`），其通过 UI 线程调度消息。这意味着后台线程发送的消息会自动到达主线程。消息总线在不同层之间传送消息时也是有用的（通常是在视图和视图模型之间）。

虽然提供了这个类，因为有时是必须的，但是消息总线应该被当做**最后一招**。消息总线是一个**全局变量**，也就是说它可能导致内存和事件泄露。还有消息总线分离的性质使得它的目标不明确。也鼓励了糟糕的设计，因为许多人会直接在视图模型中代理视图事件。使得视图模型不再纯粹。

### 基础

消息总线是相当简单的。首先，建立一个监听器：

```cs
// 监听任何发送 KeyUpEventArgs 实例的消息。
// 因为消息总线简单的返回 IObservable，
// 因此可以被组合或以多种方式使用。
MessageBus.Current.Listen<KeyUpEventArgs>()
    .Where(e => e.KeyCode == KeyCode.Up)
    .Subscribe(x => Console.WriteLine("Up Pressed!"));
```

现在，使用 `RegisterMessageSource` 将一个 IObservable 连接到事件总线：

```cs
MessageBus.Current.RegisterMessageSource(RootVisual.Events().KeyUpObs);
```

或者，你觉得非常必要但不是非常函数化：

```cs
MessageBus.Current.SendMessage(new KeyUpEventArgs());
```

### 避免使用消息总线的方式

不像其他 MVVM 框架，往往有更正确的解决问题的方式。`WhenAny` 和 `WhenAnyObservable` 常常能被使用来描述如何抵达对象，即使那些对象随时间而改变。这在视图中往往最有用：

```cs
public LoginView()
{
    // 一旦 CredentialsAreValid 变为 `true`，将焦点设置到 Ok 按钮。
    this.WhenAny(x => x.ViewModel.CredentialsAreValid, x => x.Value)
        .Where(x => x != false)
        .Subscribe(_ => OkButton.SetFocus());
}
```

假设有这样一个场景，一个打开文档的视图模型包含一个文档视图模型的列表——每个文档有一个 `Close` 命令。许多其它 MVVM 的传统实现纠结于这个命令的实现，要么保留一个对该列表的引用，要么使用消息总线。

但是，与其这样做，我们可以使用 Rx 的操作符来解决这个问题，以一种更优雅的方式。

```cs
public class DocumentViewModel : ReactiveObject
{
    public ReactiveCommand<Object> Close { get; set; }

    public DocumentViewModel() 
    {
        // 注意我们不真正 **订阅** 以关闭或在 DocumentViewModel 中实现任何东西，
		// 因为关闭是文档列表的责任
        Close = ReactiveCommand.Create();
    }
}

public class MainViewModel : ReactiveObject
{
    public ReactiveList<DocumentViewModel> OpenDocuments { get; protected set; }

    public MainViewModel()
    {
        OpenDocuments = new ReactiveList<DocumentViewModel>();

        // 无论文档的列表什么时候改变，
		// 生成新的 Observable 来表示什么时候任何**当前**文档被请求关闭，
		// 然后切换到该文档。
		// 在得到了关闭的文档的之后，将其从列表中移除
        OpenDocuments.Changed
            .Select(_ => WhenAnyDocumentClosed())
            .Switch()
            .Subscribe(x => OpenDocuments.Remove(x));
    }

    IObservable<DocumentViewModel> WhenAnyDocumentClosed()
    {
        // 将文档投射为一个 Observable 列表，
		// 其在发出信号时返回要关闭的文档，
		// 让后将这些文档汇聚在一起
        return OpenDocuments
            .Select(x => x.Close.Select(_ => x))
            .Merge();
    }
}
```
