---
title: PuttingThingsTogether
author: Sergio0694
description: An overview of how to combine different features of the MVVM Toolkit into a practical example
keywords: windows 10, uwp, windows community toolkit, uwp community toolkit, uwp toolkit, mvvm, service, messenger, messaging, net core, net standard
dev_langs:
  - csharp
---

# 将东西集中放置

现在我们已经概述了通过`Microsoft.Toolkit.Mvvm`包提供的所有不同的组件，我们可以看一个实际的例子，它们都在一起构建一个更大的例子。在这个例子中，我们想为选定的一些子Reddit建立一个非常简单简约的Reddit浏览器。

## 我们想建立什么

让我们首先概述一下我们想要建立的具体内容:

- 一个最小的Reddit浏览器由两个 "widget "组成：一个显示子reddit的帖子，另一个显示当前选择的帖子。这两个widget需要自成一体，彼此之间没有强烈的引用。
- 我们希望用户能够从一个可用的选项列表中选择一个子Reddit，我们希望将所选的子红点保存为一个设置，并在下次加载样本时加载它。
- 我们希望subreddit小部件也提供一个刷新按钮来重新加载当前的子reddit。
- 出于本示例的目的，我们不需要能够处理所有可能的post类型。 我们将为所有加载的文章分配一个示例文本，并直接显示它，使事情更简单。

## 设置视图模型

让我们从支持subreddit小部件的视图模型开始，让我们来看看我们需要的工具:

- **Commands:** 我们需要视图能够请求视图模型重新加载当前列表的文章从选定的子reddit。 我们可以使用`AsyncRelayCommand`类型来包装一个私有方法，该方法将从Reddit获取帖子。 这里，我们通过`IAsyncRelayCommand`接口公开该命令，以避免对我们正在使用的确切命令类型的强引用。 这也将允许我们在将来更改命令类型，而不必担心任何UI组件依赖于所使用的特定类型。
- **Properties:** we need to expose a number of values to the UI, which we can do with either observable properties if they're values we intend to completely replace, or with properties that are themselves observable (eg. `ObservableCollection<T>`). In this case, we have:
  - `ObservableCollection<object> Posts`, which is the observable list of loaded posts. Here we're just using `object` as a placeholder, as we haven't created a model to represent posts yet. We can replace this later on.
  - `IReadOnlyList<string> Subreddits`, which is a readonly list with the names of the subreddits that we allow users to choose from. This property is never updated, so it doesn't need to be observable either.
  - `string SelectedSubreddit`, which is the currently selected subreddit. This property needs to be bound to the UI, as it'll be used both to indicate the last selected subreddit when the sample is loaded, and to be manipulated directly from the UI as the user changes the selection. Here we're using the `SetProperty` method from the `ObservableObject` class.
  - `object SelectedPost`, which is the currently selected post. In this case we're using the `SetProperty` method from the `ObservableRecipient` class to indicate that we also want to broadcast notifications when this property changes. This is done to be able to notify the post widget that the current post selection is changed.
- **Methods:** we just need a private `LoadPostsAsync` method which will be wrapped by our async command, and which will contain the logic to load posts from the selected subreddit.

Here's the viewmodel so far:

```csharp
public sealed class SubredditWidgetViewModel : ObservableRecipient
{
    /// <summary>
    /// Creates a new <see cref="SubredditWidgetViewModel"/> instance.
    /// </summary>
    public SubredditWidgetViewModel()
    {
        LoadPostsCommand = new AsyncRelayCommand(LoadPostsAsync);
    }

    /// <summary>
    /// Gets the <see cref="IAsyncRelayCommand"/> instance responsible for loading posts.
    /// </summary>
    public IAsyncRelayCommand LoadPostsCommand { get; }

    /// <summary>
    /// Gets the collection of loaded posts.
    /// </summary>
    public ObservableCollection<object> Posts { get; } = new ObservableCollection<object>();

    /// <summary>
    /// Gets the collection of available subreddits to pick from.
    /// </summary>
    public IReadOnlyList<string> Subreddits { get; } = new[]
    {
        "microsoft",
        "windows",
        "surface",
        "windowsphone",
        "dotnet",
        "csharp"
    };

    private string selectedSubreddit;

    /// <summary>
    /// Gets or sets the currently selected subreddit.
    /// </summary>
    public string SelectedSubreddit
    {
        get => selectedSubreddit;
        set => SetProperty(ref selectedSubreddit, value);
    }

    private object selectedPost;

    /// <summary>
    /// Gets or sets the currently selected subreddit.
    /// </summary>
    public object SelectedPost
    {
        get => selectedPost;
        set => SetProperty(ref selectedPost, value, true);
    }

    /// <summary>
    /// Loads the posts from a specified subreddit.
    /// </summary>
    private async Task LoadPostsAsync()
    {
        // TODO...
    }
}
```

Now let's take a look at what we need for viewmodel of the post widget. This will be a much simpler viewmodel, as it really only needs to expose a `Post` property with the currently selected post, and to receive broadcast messages from the subreddit widget to update the `Post` property. It can look something like this:

```csharp
public sealed class PostWidgetViewModel : ObservableRecipient, IRecipient<PropertyChangedMessage<object>>
{
    private object post;

    /// <summary>
    /// Gets the currently selected post, if any.
    /// </summary>
    public object Post
    {
        get => post;
        private set => SetProperty(ref post, value);
    }

    /// <inheritdoc/>
    public void Receive(PropertyChangedMessage<object> message)
    {
        if (message.Sender.GetType() == typeof(SubredditWidgetViewModel) &&
            message.PropertyName == nameof(SubredditWidgetViewModel.SelectedPost))
        {
            Post = message.NewValue;
        }
    }
}
```

In this case, we're using the `IRecipient<TMessage>` interface to declare the messages we want our viewmodel to receive. The handlers for the declared messages will be added automatically by the `ObservableRecipient` class when the `IsActive` property is set to `true`. Note that it is not mandatory to use this approach, and manually registering each message handler is also possible, like so:

```csharp
public sealed class PostWidgetViewModel : ObservableRecipient
{
    protected override void OnActivated()
    {
        // We use a method group here, but a lambda expression is also valid
        Messenger.Register<PostWidgetViewModel, PropertyChangedMessage<object>>(this, (r, m) => r.Receive(m));
    }

    /// <inheritdoc/>
    public void Receive(PropertyChangedMessage<object> message)
    {
        if (message.Sender.GetType() == typeof(SubredditWidgetViewModel) &&
            message.PropertyName == nameof(SubredditWidgetViewModel.SelectedPost))
        {
            Post = message.NewValue;
        }
    }
}
```

We now have a draft of our viewmodels ready, and we can start looking into the services we need.

## 构建设置服务

> [!NOTE]
> 该示例使用依赖注入模式构建，这是处理视图模型中的服务的常用方法。 也可以使用其他模式，比如服务定位器模式，但是MVVM Toolkit没有提供内置的api来支持这一点。  

因为我们想要保存和持久化一些属性，所以我们需要一种方法让视图模型能够与应用程序设置交互。 我们不应该在视图模型中直接使用特定于平台的api，因为那样会妨碍我们将所有的视图模型都放在一个可移植的。net标准项目中。 我们可以通过使用服务和`Microsoft.Extensions`中的api来解决这个问题。 依赖注入'库来设置应用程序的`IServiceProvider`实例。 其思想是编写表示我们需要的所有API表面的接口，然后实现特定于平台的类型，在所有应用程序目标上实现这个接口。 视图模型将只与接口交互，因此它们根本不会对任何特定于平台的类型有任何强引用。

下面是设置服务的一个简单接口:

```csharp
public interface ISettingsService
{
    /// <summary>
    /// 为键设置值.
    /// </summary>
    /// <typeparam name="T">绑定到键的对象的类型.</typeparam>
    /// <param name="key">检查的Key</param>
    /// <param name="value">要分配给设置键的值</param>
    void SetValue<T>(string key, T value);

    /// <summary>
    /// 从当前的 <see cref="IServiceProvider"/> 实例中读取一个值，并返回正确类型的类型转换
    /// </summary>
    /// <typeparam name="T">要检索的对象的类型</typeparam>
    /// <param name="key">与请求对象相关联的键</param>
    [Pure]
    T GetValue<T>(string key);
}
```

我们可以假设，实现这个接口的平台特定类型将处理所有必要的逻辑，以实际序列化设置，存储它们到磁盘，然后读取它们。 我们现在可以在我们的`SubredditWidgetViewModel`中使用这个服务，以使`SelectedSubreddit`属性持久化到被请求的对象:

```csharp
/// <summary>
/// 获取要使用的 <see cref="ISettingsService"/> 实例.
/// </summary>
private readonly ISettingsService SettingsService;

/// <summary>
/// 创建一个新的 <see cref="SubredditWidgetViewModel"/> 实例.
/// </summary>
public SubredditWidgetViewModel(ISettingsService settingsService)
{
    SettingsService = settingsService;

    selectedSubreddit = settingsService.GetValue<string>(nameof(SelectedSubreddit)) ?? Subreddits[0];
}

private string selectedSubreddit;

/// <summary>
/// 获取或设置当前选定的子reddit
/// </summary>
public string SelectedSubreddit
{
    get => selectedSubreddit;
    set
    {
        SetProperty(ref selectedSubreddit, value);

        SettingsService.SetValue(nameof(SelectedSubreddit), value);
    }
}
```

这里我们使用依赖注入和构造函数注入，如上所述。 我们已经声明了一个`ISettingsService SettingsService`字段，它只是存储了我们的设置服务(我们在viewmodel构造函数中接收的参数)，然后我们在构造函数中初始化`SelectedSubreddit`属性，通过使用之前的值或只是第一个可用的subreddit。 然后，我们还修改了`SelectedSubreddit ` setter，以便它也将使用设置服务将新值保存到磁盘.

太棒了! 现在我们只需要编写这个服务的平台特定版本，这一次直接在我们的一个应用程序项目中。 下面是该服务在UWP上的样子:

```csharp
public sealed class SettingsService : ISettingsService
{
    /// <summary>
    /// <see cref="IPropertySet"/> 针对当前实例的设置.
    /// </summary>
    private readonly IPropertySet SettingsStorage = ApplicationData.Current.LocalSettings.Values;

    /// <inheritdoc/>
    public void SetValue<T>(string key, T value)
    {
        if (!SettingsStorage.ContainsKey(key)) SettingsStorage.Add(key, value);
        else SettingsStorage[key] = value;
    }

    /// <inheritdoc/>
    public T GetValue<T>(string key)
    {
        if (SettingsStorage.TryGetValue(key, out object value))
        {
            return (T)value;
        }

        return default;
    }
}
```

这个难题的最后一部分是将这个平台特定的服务注入到我们的服务提供者实例中。我们可以在创业时这样做:

```csharp
/// <summary>
/// 获取解析应用程序服务的<see cref="IServiceProvider"/>实例
/// </summary>
public IServiceProvider Services { get; }

/// <summary>
/// 为应用程序配置服务
/// </summary>
private static IServiceProvider ConfigureServices()
{
    var services = new ServiceCollection();

    services.AddSingleton<ISettingsService, SettingsService>();
    services.AddTransient<PostWidgetViewModel>();

    return services.BuildServiceProvider();
}
```

这将注册我们的`SettingsService`的一个单例实例作为实现`ISettingsService`的类型。 我们还将`PostWidgetViewModel`注册为一个临时服务，这意味着每次我们检索一个实例时，它都将是一个新实例(你可以想象，如果想要有多个独立的post widget，这是很有用的)。 这意味着每次我们解析`ISettingsService`实例时，当使用中的应用程序是UWP实例时，它将接收到一个`SettingsService`实例，该实例将使用UWP api在后台操作设置。 完美!  

## 构建Reddit服务

The last component of the backend that we're missing is a service that is able to use the Reddit REST APIs to fetch the posts from the subreddits we're interested in. To build it, we're going to use [refit](https://github.com/reactiveui/refit), which is a library to easily build type-safe services to interact with REST APIs. As before, we need to define the interface with all the APIs that our service will implement, like so:

```csharp
public interface IRedditService
{
    /// <summary>
    /// Get a list of posts from a given subreddit
    /// </summary>
    /// <param name="subreddit">The subreddit name.</param>
    [Get("/r/{subreddit}/.json")]
    Task<PostsQueryResponse> GetSubredditPostsAsync(string subreddit);
}
```

That `PostsQueryResponse` is a model we wrote that maps the JSON response for that API. The exact structure of that class is not important - suffice to say that it contains a collection of `Post` items, which are simple models representing our posts, that like like this:

```csharp
public class Post
{
    /// <summary>
    /// Gets or sets the title of the post.
    /// </summary>
    public string Title { get; set; }

    /// <summary>
    /// Gets or sets the URL to the post thumbnail, if present.
    /// </summary>
    public string Thumbnail { get; set; }

    /// <summary>
    /// Gets the text of the post.
    /// </summary>
    public string SelfText { get; }
}
```

Once we have our service and our models, can plug them into our viewmodels to complete our backend. While doing so, we can also replace those `object` placeholders with the `Post` type we've defined:

```csharp
public sealed class SubredditWidgetViewModel : ObservableRecipient
{
    /// <summary>
    /// Gets the <see cref="IRedditService"/> instance to use.
    /// </summary>
    private readonly IRedditService RedditService = Ioc.Default.GetRequiredService<IRedditService>();

    /// <summary>
    /// Loads the posts from a specified subreddit.
    /// </summary>
    private async Task LoadPostsAsync()
    {
        var response = await RedditService.GetSubredditPostsAsync(SelectedSubreddit);

        Posts.Clear();

        foreach (var item in response.Data.Items)
        {
            Posts.Add(item.Data);
        }
    }
}
```

We have added a new `IRedditService` field to store our service, just like we did for the settings service, and we implemented our `LoadPostsAsync` method, which was previously empty.

The last missing piece now is just to inject the actual service into our service provider. The big difference in this case is that by using `refit` we don't actually need to implement the service at all! The library will automatically create a type implementing the service for us, behind the scenes. So we only need to get an `IRedditService` instance and inject it directly, like so:

```csharp
/// <summary>
/// Configures the services for the application.
/// </summary>
private static IServiceProvider ConfigureServices()
{
    var services = new ServiceCollection();

    services.AddSingleton<ISettingsService, SettingsService>();
    services.AddSingleton(RestService.For<IRedditService>("https://www.reddit.com/"));
    services.AddTransient<PostWidgetViewModel>();

    return services.BuildServiceProvider();
}
```

And that's all we need to do! We now have all our backend ready to use, including two custom services that we created specifically for this app! 🎉

## Building the UI

Now that all the backend is completed, we can write the UI for our widgets. Note how using the MVVM pattern let us focus exclusively on the business logic at first, without having to write any UI-related code until now. Here we'll remove all the UI code that's not interacting with our viewmodels, for simplicity, and we'll go through each different control one by one. The full source code can be found in the sample app.

Before going through the various controls, here's how we can resolve viewmodels for all the different views in our application (eg. the `PostWidgetView`):

```csharp
public PostWidgetView()
{
    this.InitializeComponent();
    this.DataContext = App.Current.Services.GetService<PostWidgetViewModel>();
}

public PostWidgetViewModel ViewModel => (PostWidgetViewModel)DataContext;
```

We're using our `IServiceProvider` instance to resolve the `PostWidgetViewModel` object we need, which is then assigned to the data context property. We're also creating a strongly-typed `ViewModel` property that simply casts the data context to the correct viewmodel type - this is needed to enable `x:Bind` in the XAML code.

Let's start with the subreddit widget, which features a `ComboBox` to select a subreddit, a `Button` to refresh the feed, a `ListView` to display posts and a `ProgressBar` to indicate when the feed is loading. We'll assume that the `ViewModel` property represents an instance of the viewmodel we've described before - this can be declared either in XAML or directly in code behind.

### Subreddit selector:

```xml
<ComboBox
    ItemsSource="{x:Bind ViewModel.Subreddits}"
    SelectedItem="{x:Bind ViewModel.SelectedSubreddit, Mode=TwoWay}">
    <interactivity:Interaction.Behaviors>
        <core:EventTriggerBehavior EventName="SelectionChanged">
            <core:InvokeCommandAction Command="{x:Bind ViewModel.LoadPostsCommand}"/>
        </core:EventTriggerBehavior>
    </interactivity:Interaction.Behaviors>
</ComboBox>
```

Here we're binding the source to the `Subreddits` property, and the selected item to the `SelectedSubreddit` property. Note how the `Subreddits` property is only bound once, as the collection itself sends change notifications, while the `SelectedSubreddit` property is bound with the `TwoWay` mode, as we need it both to be able to load the value we retrieve from our settings, as well as updating the property in the viewmodel when the user changes the selection. Additionally, we're using a XAML behavior to invoke our command whenever the selection changes.

### Refresh button:

```xml
<Button Command="{x:Bind ViewModel.LoadPostsCommand}"/>
```

This component is extremely simple, we're just binding our custom command to the `Command` property of the button, so that the command will be invoked whenever the user clicks on it.

### Posts list:

```xml
<ListView
    ItemsSource="{x:Bind ViewModel.Posts}"
    SelectedItem="{x:Bind ViewModel.SelectedPost, Mode=TwoWay}">
    <ListView.ItemTemplate>
        <DataTemplate x:DataType="models:Post">
            <Grid>
                <TextBlock Text="{x:Bind Title}"/>
                <controls:ImageEx Source="{x:Bind Thumbnail}"/>
            </Grid>
        </DataTemplate>
    </ListView.ItemTemplate>
</ListView>
```

Here we have a `ListView` binding the source and selection to our viewmodel property, and also a template used to display each post that is available. We're using `x:DataType` to enable `x:Bind` in our template, and we have two controls binding directly to the `Title` and `Thumbnail` properties of our post.

### Loading bar:

```xml
<ProgressBar Visibility="{x:Bind ViewModel.LoadPostsCommand.IsRunning, Mode=OneWay}"/>
```

Here we're binding to the `IsRunning` property, which is part of the `IAsyncRelayCommand` interface. The `AsyncRelayCommand` type will take care of raising notifications for that property whenever the asynchronous operation starts or completes for that command.

---

The last missing piece is the UI for the post widget. As before, we've removed all the UI-related code that was not necessary to interact with the viewmodels, for simplicity. The full source code is available in the sample app.

```xml
<Grid>

    <!--Header-->
    <Grid>
        <TextBlock Text="{x:Bind ViewModel.Post.Title, Mode=OneWay}"/>
        <controls:ImageEx  Source="{x:Bind ViewModel.Post.Thumbnail, Mode=OneWay}"/>
    </Grid>

    <!--Content-->
    <ScrollViewer>
        <TextBlock Text="{x:Bind ViewModel.Post.SelfText, Mode=OneWay}"/>
    </ScrollViewer>
</Grid>
```

Here we just have a header, with a `TextBlock` and an `ImageEx` control binding their `Text` and `Source` properties to the respective properties in our `Post` model, and a simple `TextBlock` inside a `ScrollViewer` that is used to display the (sample) content of the selected post.

## Good to go! 🚀

We've now built all our viewmodels, the necessary services, and the UI for our widgets - our simple Reddit browser is completed! This was just meant to be an example of how to build an app following the MVVM pattern and using the APIs from the MVVM Toolkit.

As stated above, this is only a reference, and you're free to modify this structure to fit your needs and/or to pick and choose only a subset of components from the library. Regardless of the approach you take, the MVVM Toolkit should provide a solid foundation to hit the ground running when starting a new application, by letting you focus on your business logic instead of having to worry about manually doing all the necessary plumbing to enable proper support for the MVVM pattern.
