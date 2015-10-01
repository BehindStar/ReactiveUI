# ReactiveList

一个与 ReactiveUI 同时到来的内置类型是一个改进版本的 .NET `ObservableCollection`（讽刺的是，**不是**一个 Observable）。 `ReactiveList` 可以在任何使用 List
或 ObservableCollection 的地方使用。

### 订阅改变

`ReactiveList` 提供了一些有用的 Observable ，可以被订阅以获取列表改变通知。同时也能在列表改变前通知你：

* **(Before)ItemsAdded** - 在项目增加时发出信号
* **(Before)ItemsRemoved** - 在项目移除时发出信号
* **(Before)ItemsMoved** - 在项目移动时发出信号
* **CountChang(ing/ed)** - 在项目数量（任何原因）改变时发出信号
* **Changed** - 通过 `NotifyCollectionChangedEventArgs` 传递所有更改（`NotifyCollectionChanged` 的 Observable 版本）.
* **ShouldReset** - 在发生显著改变时，发出信号通知观察者应该重新读取整个集合

### Reset 的含义

理解 ShouldReset Observable 的含义十分重要。该事件的意思是，集合已经彻底改变了，你应该重新读取内容。许多人将 *Reset* 和 *Clear* 混为一谈，认为其意思是集合已经清空了。

这非常重要，因为如果只订阅 `ItemsAdded` 和 `ItemsRemoved`，将不能正确跟踪集合中的每个项目。ReactiveList 能够跟踪这种情况，并且尝试通知你。比如，有一个关于维护集合中的 “运行数量” 的例子：

```cs
// 该例子仅供演示，CountChanged 和这个一样。
TweetList = new ReactiveList<Tweet>();

var count = 0;

var addedOrRemoved = Observable.Merge(
    TweetList.ItemsAdded.Subscribe(_ => 1),
    TweetList.ItemsRemoved.Subscribe(_ => -1));

addedOrRemoved.Subscribe(x => count += x);
TweetList.ShouldReset.Subscribe(_ => count = TweetList.Count);
```

如果你想在集合的每个对象添加或删除时执行一些代码，该文档中的 `ActOnEveryObject` 方法会自动处理。

### 使用更改跟踪

ReactiveList 不仅仅能监测列表更改，也能选择性的告诉你列表**里面**任何项的更改。有一个关于获得文档列表中的某个文档被修改的通知的例子：

```cs
DocumentList = new ReactiveList<Document>() {
    ChangeTrackingEnabled = true,
};

DocumentList.ItemChanged
    .Where(x => x.PropertyName == "IsDirty" && x.Sender.IsDirty)
    .Select(x => x.Sender)
    .Subscribe(x => {
        Console.WriteLine("Make sure to save {0}!", x.DocumentName);
    });
```

注意必须将  `ChangeTrackingEnabled` 设置为 `true`，由于性能原因，更改跟踪默认情况下是关闭的。有一点是**没必要**做的。

ReactiveList 只跟踪列表中的直接对象，它不会跟踪整个对象层次（`listOfItems[0].Foo.Bar = true` 不会触发更改通知，但是 `listOfItems[0].Foo = Bar` 会）。

### 抑制通知

由于 ReactiveList 通常绑定到 UI 元素上，例如 ListBox。一次更新许多小的更改能够显著提高性能。对列表的修改最终反映到 UI 元素的创建、销毁以及重新布局和重新渲染上，这些操作的代价都十分昂贵。

因此，当你想一次应用列表的多个更改时，使用 `SuppressChangeNotifications`——在操作期间禁用更改通知，然后发送一个 **Reset** 通知触发 UI 重载。 

```cs
// 没有这个 using 声明，与这个列表相关的 ListBox 将会刷新 **4** 次！
using (TweetsList.SuppressChangeNotifications()) {
    TweetsList.Clear();

    for(int page=0; page < 3; page++) {
        var tweets = await GetTweets(page);
        TweetsList.AddRange(tweets);
    }
}
```

Range 方法类似 `AddRange`， `InsertRange`，等等，自动抑制更改通知，如果列表的更改百分比在设定值以上（比如，如果改变了列表的90%，触发一个 Reset 更有意义，但是如果改变了5%，那么最好触发 Adds/Deletes）。

### CreateCollection

在测试中这个方法十分有用，用于通过一个 Observable 创建一个自动生成内容的列表。在这个例子中，测试对象状态变得更为容易，也可以通过数量变化检查正确性：

```cs
[Fact]
public void MakeSureTweetListGetsUpdatedOnRefresh()
{
    var fixture = new TweetsListViewModel();
    var output = fixture.TweetList.ItemsAdded.CreateCollection();

    Assert.Equal(0, output.Count);

    fixture.Refresh.Execute();
    Assert.Equal(1, output.Count);
}
```
