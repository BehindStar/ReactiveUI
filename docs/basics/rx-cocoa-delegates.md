# Rx Cocoa Delegates
 
作为事件库的一部分，基于 Cocoa 平台的 ReactiveUI.Events 创建了所有 AppKit/UIKit 中的委托类的子类，并为那些不返回值的委托事件方法创建了 Observable。这些类都在和它们的父类相同的命名空间下，但是有一个 `Rx` 后缀，同时所有的 Observable 都以 `Obs` 结束。

比如：

```cs

var tvd = new UITableViewDelegateRx();
tvd.ScrolledObs
    .Subscribe(_ => Console.WriteLine("Hey we scrolled!"));
tableView.Delegate = tvd;
```

### 注意事项

注意那些返回值的事件不能转换为 Observable —— 这些事件以 `Should` 开头。比如：

```cs

var tvd = new UITableViewDelegateRx();

// 编译错误，不存在
tvd.ShouldScrollToTopObs
    .Subscribe(_ => Console.WriteLine("How do we return true?!?"))
```

因为这个类是一个普通类，所以你可以自己派生它，并实现 `ShouldXXXX` 方法。
