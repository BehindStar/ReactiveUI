# ReactiveUI.Events

尽管 ReactiveUI 主要集中于创建视图模型，也有一个独立的库帮助你编写视图中的代码。该库叫做 `ReactiveUI.Events`（NuGet 包名为 `ReactiveUI-Events`）

这个库由代码生成，为 UI 框架的所有事件添加了 Observable，通过一个新的扩展方法 `Events()`。可以在视图模型中替换大部分使用 `Observable.FromEventPattern` 的代码。`Events` 直接映射事件参数。

### 示例

```cs
var router = RxApp.GetService<IScreen>().Router;

this.Events().KeyUp
	.Where(x => x.Key == Key.Escape)
	.InvokeCommand(router.NavigateBack);
```

```cs
var windowChanged = Observable.Merge(
	this.Events().SizeChanged.Select(_ => Unit.Default),
	this.WhenAny(x => x.Left, x => x.Top, (l,t) => Unit.Default));

windowChanged
	.Throttle(TimeSpan.FromMilliseconds(700), RxApp.MainThreadScheduler)
	.Subscribe(_ => SaveWindowPosition());
```
