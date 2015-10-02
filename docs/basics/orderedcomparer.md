# OrderedComparer

OrderedComparer 是一个比较器，允许以类似 LINQ 的排序语法创建一个比较器。这对于创建一个在派生集合中使用的比较器来说比较方便。

## 对员工进行排序

假如有一个 ```ReactiveList<Employee>``` 其 ```Employee``` 是这样定义的：

```cs
public class Employee
{
    public string Name {get; set;}
	public string Department {get; set;}
    public double Salary {get; set;}
}
```

这是一个很简单的模型，来写一个比较方法来对所有员工按部门进行排序，然后是名称，最后是薪水（降序）：

```cs
public int SortEmployee(Employee x, Employee y) 
{
    int department = x.Department.CompareTo(y.Department);
    if(department != 0) return 0;

    int name = x.Name.CompareTo(y.Name);
    if(name != 0) return 0;

    // Swap x and y for descending
    return y.Salary.CompareTo(x.Salary)
}
```

看起来还可以但是如果有个员工没有部门？出现了 ```NullReferenceException```。还有许多这样的边际例子，你可以想象得到这个方法是多复杂，且在处理更复杂的模型时更容易出错。现在用 ```OrderedComparer``` 写一个等效替代：

```cs
readonly static IComparer<Employee> employeeComparer = OrderedComparer<Employee>
    .OrderBy(x => x.Department)
    .ThenBy(x => x.Name)
    .ThenByDescending(x => x.Salary);

public int SortEmployee(Employee x, Employee y) 
{
    return employeeComparer.Compare(x, y);
}
```

## 在 CreateDerivedCollection 中使用

不幸的是，CreateDerivedCollection 不接受一个 IComparer<Employee> 作为参数，它需要一个 ```Comparison<Employee>``` 委托。幸运的是 ``IComparer<Employee>.Compare`` 就是。

```cs
var employees = new ReactiveList<Employee> { ... }
var orderedEmployees = employees.CreateDerivedCollect(
   x => x, 
   orderer: OrderedComparer<Employee>
	   .OrderBy(x => x.Department)
	   .ThenBy(x => x.Name)
	   .ThenByDescending(x => x.Salary).Compare; // .Compare 在最后
);
```

现在要测试的是，放一些员工到这个列表中，并验证 orderedEmployees 是否和你希望的顺序匹配。如果使用使用上面的例子创建一个专用排序方法的话，你甚至可以直接传入方法：

```cs
var employees = new ReactiveList<Employee> { ... }
var orderedEmployees = employees.CreateDerivedCollect(x => x, orderer:  SortEmployee);
```

### 组合自定义比较器

默认情况下 ```OrderBy```， ```OrderByDescending```， ```ThenBy``` 和 ```ThenByDescending``` 使用 ```Comparer<T>.Default``` 作为实际比较器，就和 LINQ to object 一样。对于字符串，使用的是 ```StringComparer.CurrentCulture```。

你可以一步一步的换掉那些。比如说如果你想按名称以不分区域性（Culture）和大小写的方式排序：

```cs
       orderer: OrderedComparer<Employee>
           .OrderBy(x => x.Name, StringComparer.OrdinalIgnoreCase)
```

注意: 如果对实现了  ```ICompare``` 或 ```ICompare<T>``` 的类进行排序，将会使用类自身的比较器。
