
## 一、引言


站长接触 AOT 已有 3 个月之久，此前在《[好消息：NET 9 X86 AOT的突破 \- 支持老旧Win7与XP环境](https://github.com)》一文中就有所提及。在这段时间里，站长使用 Avalonia 开发的项目也成功完成了 AOT 发布测试。然而，这一过程并非一帆风顺。站长在项目功能完成大半部分才开始进行 AOT 测试，期间遭遇了不少问题，可谓是 “踩坑无数”。为了方便日后回顾，也为了给广大读者提供参考，在此将这段经历进行总结。



> .NET AOT是将.NET代码提前编译为本机代码的技术。其优势众多，启动速度快，减少运行时资源占用，还提高安全性。AOT发布后无需再安装.NET运行时等依赖。.NET 8、9 AOT发布后，可在XP、Win7非SP1操作系统下运行。这使得应用部署更便捷，能适应更多老旧系统环境，为开发者拓展了应用场景，在性能提升的同时，也增加了系统兼容性，让.NET应用的开发和部署更具灵活性和广泛性，给用户带来更好的体验。


## 二、经验之谈


### （一）测试策略的重要性


从项目创建伊始，就应养成良好的习惯，即只要添加了新功能或使用了较新的语法，就及时进行 AOT 发布测试。否则，问题积累到后期，解决起来会异常艰难，站长就因前期忽视了这一点，付出了惨痛的代价。无奈的解决方法是重新创建项目，然后逐个还原功能并进行 AOT 测试。经过了一周的加班AOT测试，每个 AOT 发布过程大致如下：


1. 内网 AOT 发布一次需 2、3 分钟，这段时间只能看看需求文档、技术文章、需求文档、技术文章。。。
2. 发布完成，运行无效果，体现在双击未出现界面，进程列表没有它，说明程序崩溃了，查看系统应用事件日志，日志中通常会包含异常警告信息。
3. 依据日志信息检查代码，修改相关 API。
4. 再次进行 AOT 发布，重复上述 1 \- 3 步骤。


经过一周的努力，项目 AOT 后功能测试终于正常，至此收工。


### （二）AOT 需要注意的点及解决方法


#### 1\. 添加rd.xml


在主工程创建一个XML文件，例如`Roots.xml`，内容大致如下：



```


|  | <linker> |
| --- | --- |
|  | <assembly fullname="CodeWF.Toolbox.Desktop" preserve="All" /> |
|  | linker> |


```

需要支持AOT的工程，在该XML中添加一个`assembly`节点，`fullname`是程序集名称，`CodeWF.Toolbox.Desktop`是站长小工具的主工程名，[点击](https://github.com)查看源码。


在主工程添加`ItemGroup`节点关联该XML文件：



```


|  | <ItemGroup> |
| --- | --- |
|  | <TrimmerRootDescriptor Include="Roots.xml" /> |
|  | ItemGroup> |


```

#### 2\. Prism支持


站长使用了Prism框架及DryIOC容器，若要支持 AOT，需要添加以下 NuGet 包：



```


|  | <PackageReference Include="Prism.Avalonia" Version="8.1.97.11073" /> |
| --- | --- |
|  | <PackageReference Include="Prism.DryIoc.Avalonia" Version="8.1.97.11073" /> |


```

`rd.xml`需要添加



```


|  | <assembly fullname="Prism" preserve="All" /> |
| --- | --- |
|  | <assembly fullname="DryIoc" preserve="All" /> |
|  | <assembly fullname="Prism.Avalonia" preserve="All" /> |
|  | <assembly fullname="Prism.DryIoc.Avalonia" preserve="All" /> |


```

#### 3\. App.config读写


在.NET Core中使用`System.Configuration.ConfigurationManager`包操作App.config文件，`rd.xml`需添加如下内容：



```


|  | <assembly fullname="System.Configuration.ConfigurationManager" preserve="All" /> |
| --- | --- |


```

使用`Assembly.GetEntryAssembly().location`失败，目前使用`ConfigurationManager.OpenExeConfiguration(ConfigurationUserLevel.None)`获取的应用程序程序配置，指定路径的方式后续再研究。


#### 4\. HttpClient使用


`rd.xml`添加如下内容：



```


|  | <assembly fullname="System.Net.Http" preserve="All" /> |
| --- | --- |


```

#### 5\. Dapper支持


Dapper的AOT支持需要安装`Dapper.AOT`包，`rd.xml`添加如下内容：



```


|  | <assembly fullname="Dapper" preserve="All" /> |
| --- | --- |
|  | <assembly fullname="Dapper.AOT" preserve="All" /> |


```

数据库操作的方法需要添加`DapperAOT`特性，举例如下：



```


|  | [DapperAot] |
| --- | --- |
|  | public static bool EnsureTableIsCreated() |
|  | { |
|  | try |
|  | { |
|  | using var connection = new SqliteConnection(DBConst.DBConnectionString); |
|  | connection.Open(); |
|  |  |
|  | const string sql = $@" |
|  | CREATE TABLE IF NOT EXISTS {nameof(JsonPrettifyEntity)}( |
|  | {nameof(JsonPrettifyEntity.IsSortKey)} Bool, |
|  | {nameof(JsonPrettifyEntity.IndentSize)} INTEGER |
|  | )"; |
|  |  |
|  | using var command = new SqliteCommand(sql, connection); |
|  | return command.ExecuteNonQuery() > 0; |
|  | } |
|  | catch (Exception ex) |
|  | { |
|  | return false; |
|  | } |
|  | } |


```

#### 6\. System.Text.Json


参考[JsonExtensions.cs](https://github.com)


序列化



```


|  | public static bool ToJson<T>(this T obj, out string? json, out string? errorMsg) |
| --- | --- |
|  | { |
|  | if (obj == null) |
|  | { |
|  | json = default; |
|  | errorMsg = "Please provide object"; |
|  | return false; |
|  | } |
|  |  |
|  | var options = new JsonSerializerOptions() |
|  | { |
|  | WriteIndented = true, |
|  | Encoder = JavaScriptEncoder.UnsafeRelaxedJsonEscaping, |
|  | TypeInfoResolver = new DefaultJsonTypeInfoResolver() |
|  | }; |
|  | try |
|  | { |
|  | json = JsonSerializer.Serialize(obj, options); |
|  | errorMsg = default; |
|  | return true; |
|  | } |
|  | catch (Exception ex) |
|  | { |
|  | json = default; |
|  | errorMsg = ex.Message; |
|  | return false; |
|  | } |
|  | } |


```

反序列化



```


|  | public static bool FromJson<T>(this string? json, out T? obj, out string? errorMsg) |
| --- | --- |
|  | { |
|  | if (string.IsNullOrWhiteSpace(json)) |
|  | { |
|  | obj = default; |
|  | errorMsg = "Please provide json string"; |
|  | return false; |
|  | } |
|  |  |
|  | try |
|  | { |
|  | var options = new JsonSerializerOptions() |
|  | { |
|  | Encoder = JavaScriptEncoder.UnsafeRelaxedJsonEscaping, |
|  | TypeInfoResolver = new DefaultJsonTypeInfoResolver() |
|  | }; |
|  | obj = JsonSerializer.Deserialize(json!, options); |
|  | errorMsg = default; |
|  | return true; |
|  | } |
|  | catch (Exception ex) |
|  | { |
|  | obj = default; |
|  | errorMsg = ex.Message; |
|  | return false; |
|  | } |
|  | } |


```

#### 7\. 反射问题


参考项目[CodeWF.NetWeaver](https://github.com)


1. 创建指定类型的`List`或`Dictionary`实例：



```


|  | public static object CreateInstance(Type type) |
| --- | --- |
|  | { |
|  | var itemTypes = type.GetGenericArguments(); |
|  | if (typeof(IList).IsAssignableFrom(type)) |
|  | { |
|  | var lstType = typeof(List<>); |
|  | var genericType = lstType.MakeGenericType(itemTypes.First()); |
|  | return Activator.CreateInstance(genericType)!; |
|  | } |
|  | else |
|  | { |
|  | var dictType = typeof(Dictionary<,>); |
|  | var genericType = dictType.MakeGenericType(itemTypes.First(), itemTypes[1]); |
|  | return Activator.CreateInstance(genericType)!; |
|  | } |
|  | } |


```

2. 反射调用`List`和`Dictionary`的`Add`方法添加元素失败，下面是伪代码：



```


|  | // List |
| --- | --- |
|  | var addMethod = type.GetMethod("Add"); |
|  | addMethod.Invoke(obj, new[]{ child }) |
|  |  |
|  | // Dictionary |
|  | var addMethod = type.GetMethod("Add"); |
|  | addMethod.Invoke(obj, new[]{ key, value }) |


```

解决办法，转换为实现的接口调用：



```


|  | // List |
| --- | --- |
|  | (obj as IList).Add(child); |
|  |  |
|  | // Dictionary |
|  | (obj as IDictionary)[key] = value; |


```

3. 获取数组、`List`、`Dictionary`的元素个数


同上面Add方法反射获取Length或Count属性皆返回0，`value.Property("Length", 0)`，封装的Property非AOT运行正确：



```


|  | public static T Property<T>(this object obj, string propertyName, T defaultValue = default) |
| --- | --- |
|  | { |
|  | if (obj == null) throw new ArgumentNullException(nameof(obj)); |
|  | if (string.IsNullOrEmpty(propertyName)) throw new ArgumentNullException(nameof(propertyName)); |
|  |  |
|  | var propertyInfo = obj.GetType().GetProperty(propertyName); |
|  | if (propertyInfo == null) |
|  | { |
|  | return defaultValue; |
|  | } |
|  |  |
|  | var value = propertyInfo.GetValue(obj); |
|  |  |
|  | try |
|  | { |
|  | return (T)Convert.ChangeType(value, typeof(T)); |
|  | } |
|  | catch (InvalidCastException) |
|  | { |
|  | return defaultValue; |
|  | } |
|  | } |


```

AOT成功：直接通过转换为基类型或实现的接口调用属性即可：



```


|  | // 数组 |
| --- | --- |
|  | var length = ((Array)value).Length; |
|  |  |
|  | // List |
|  | if (value is IList list) |
|  | { |
|  | var count = list.Count; |
|  | } |
|  |  |
|  | // Dictionary |
|  | if (value is IDictionary dictionary) |
|  | { |
|  | var count = dictionary.Count; |
|  | } |


```

#### 8\. Windows 7支持


如遇AOT后无法在`Windows 7`运行，请添加`YY-Thunks`包：



```


|  | "YY-Thunks" Version="1.1.4-Beta3" /> |
| --- | --- |


```

并指定目标框架为`net9.0-windows`。


#### 9\. Winform\\兼容XP


如果第8条后还运行不了，请参考上一篇文章《[.NET 9 AOT的突破 \- 支持老旧Win7与XP环境 \- 码界工坊 (dotnet9\.com)](https://github.com):[FlowerCloud机场](https://hushicha.org)》添加VC\-LTL包，这里不赘述。


#### 10\. 其他


还有许多其他需要注意的地方，后续想起来逐渐完善本文。


## 三、总结


AOT 发布测试虽然过程中可能会遇到诸多问题，但通过及时的测试和正确的配置调整，最终能够实现项目的顺利发布。希望以上总结的经验能对大家在 AOT 使用过程中有所帮助，让大家在开发过程中少走弯路，提高项目的开发效率和质量。同时，也期待大家在实践中不断探索和总结，共同推动技术的进步和发展。


AOT可参考项目：


* CodeWF.NetWeaver: [https://github.com/dotnet9/CodeWF.NetWeaver](https://github.com)
* CodeWF.Tools：[https://github.com/dotnet9/CodeWF.Tools](https://github.com)
* CodeWF.Toolbox：[https://github.com/dotnet9/CodeWF.Toolbox](https://github.com)


