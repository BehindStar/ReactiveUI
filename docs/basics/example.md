#基本例子

## 属性示例
```cs
public class SearchViewModel : ReactiveObject, IRoutableViewHost
{
    public ReactiveList<SearchResults> SearchResults { get; set; }
 
    private string searchQuery;
    public string SearchQuery {
        get { return searchQuery; }
        set { this.RaiseAndSetIfChanged(ref searchQuery, value); }
    }
 
    public ReactiveCommand<List<SearchResults>> Search { get; set; }
 
    public ISearchService SearchService { get; set; }
}
```

## 命令示例

```cs
// 构造函数
public SearchViewModel(ISearchService searchService = null)
{
    SearchService = searchService ?? Locator.Current.GetService<ISearchService>();
	
	// 在这里以一种 *声明式* 的方式，指定 Search 命令什么时候可以执行。
	// 现在命令的 IsEnabled 已经非常高效了，因为只在 UI 需要更新的情况下才会更新。
    var canSearch = this.WhenAny(x => x.SearchQuery, x => !String.IsNullOrWhiteSpace(x.Value));
	
	// ReactiveCommand 自身能够支持后台操作，确保同一时间只执行一次。
	// 与此同时，CanExecute 将会自动设置为 false，在执行过程中自动设置 IsExecuting。
    Search = ReactiveCommand.CreateAsyncTask(canSearch, async _ => {
        return await searchService.Search(this.SearchQuery);
    });
	
	// ReactiveCommand 是 IObservable 对象，其值为异步方法的返回值，保证到达 UI 线程。
	// 在后台加载搜索结果，并将其填充到 SearchResults。
    Search.Subscribe(results => {
        SearchResults.Clear();
        SearchResults.AddRange(results);
    });
	
	// ThrownExceptions 可以取得由 CreateAsyncTask 创建的 Observable 抛出的异常。   
    Search.ThrownExceptions
        .Subscribe(ex => {
            UserError.Throw("Potential Network Connectivity Error", ex);
        });
	
	// 在 Search 查询产生变化时，在 1 秒钟内没有新的变化，就会自动调用命令。
    this.WhenAnyValue(x => x.SearchQuery)
        .Throttle(TimeSpan.FromSeconds(1), RxApp.MainThreadScheduler)
        .InvokeCommand(this, x => x.Search);
}
```
