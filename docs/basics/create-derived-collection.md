# CreateDerivedCollection 和派生列表

`ReactiveList.CreateDerivedCollection` 是 MVVM 编程中一个非常有用的方法，允许你创建一个 ReactiveList 的投影作为其他列表。`CreateDerivedCollection` 允许对源 ReactiveList 投影、排序和过滤。

一旦派生列表创建完成，该列表就会基于源集合动态更新——在源集合添加、删除项目时，该列表也会自动更新内容。如果源集合启用了 `ChangeTrackingEnabled` ，源集合的更改也会更新到派生集合。

比如说，如果有一个菜单项的源集合，并且你有一个 `过滤` 方法 `x => x.IsEnabled`，改变任何菜单项的 `IsEnabled` 都会隐藏或显示列表的项目。注意派生列表是只读的——不能人工修改派生列表。

### 经典例子：从模型创建视图模型

```cs
public class TweetsListViewModel : ReactiveObject
{
    ReactiveList<Tweet> Tweets = new ReactiveList<Tweet>();
    IReactiveDerivedList<TweetTileViewModel> TweetTiles;

    public TweetsListViewModel()
    {
        TweetTiles = Tweets.CreateDerivedCollection(
            x => new TweetTileViewModel() { Model = x });

        // 向 Tweets 添加新项目
        // TweetTiles 中出现了新的视图模型
        Tweets.Add(new Tweet() { Title = "Hello!", });
    }
}
```

### 更有用的例子：过滤和排序

`CreateDerivedCollection` 可以做一些有趣的工作，但是为了演示，需要更多的类。

```cs
public class Tweet 
{
    public DateTime CreatedAt { get; set; }
}

public class TweetTileViewModel : ReactiveObject
{
    bool isHidden;
    public bool IsHidden {
        get { return isHidden; }
        set { this.RaiseAndSetIfChanged(ref isHidden, value); }
    }

    public Tweet Model { get; set; }
}
```

现在，开始进行过滤和排序。需要特别注意的是，由于重新过滤列表基于 IsHidden 的值，因此视图模型必须是 ReactiveObject 且触发更改通知，否则 CreateDerivedList 没办法知道什么时候进行过滤。

因为必须在视图模型上添加 `IsHidden` ，所以需要设置两层过滤。如果模型是 ReactiveObject 的话就更简单了，但通常这是不可能的。

仔细看一看：

```cs
public class TweetsListViewModel : ReactiveObject
{
    ReactiveList<Tweet> Tweets = new ReactiveList<Tweet>();

    IReactiveDerivedList<TweetTileViewModel> TweetTiles;
    IReactiveDerivedList<TweetTileViewModel> VisibleTiles;

    public TweetsListViewModel()
    {
        TweetTiles = Tweets.CreateDerivedCollection(
            x => new TweetTileViewModel() { Model = x },	//选择
            x => true,										//过滤
            (x, y) => x.CreatedAt.CompareTo(y.CreatedAt));	//排序

        VisibleTiles = TweetTiles.CreateDerivedCollection(
            x => x,											
            x => !x.IsHidden);	
    }
}
```

### 总结

CreateDerivedCollection 允许定义如何变换一个列表，以及如何根据源列表变化。
