---
title: RelayCommand
author: Sergio0694
description: A command whose sole purpose is to relay its functionality to other objects by invoking delegates
keywords: windows 10, uwp, windows community toolkit, uwp community toolkit, uwp toolkit, mvvm, componentmodel, property changed, notification, binding, command, delegate, net core, net standard
dev_langs:
  - csharp
---

# RelayCommand

[`RelayCommand`](https://docs.microsoft.com/dotnet/api/microsoft.toolkit.mvvm.input.RelayCommand) 和 [`RelayCommand<T>`](https://docs.microsoft.com/dotnet/api/microsoft.toolkit.mvvm.input.RelayCommand-1) 是`ICommand`的实现，可以向视图公开方法或委托。这些类型作为视图模型和UI元素之间绑定命令的一种方式。 

## How they work

`RelayCommand`和`RelayCommand<T>`有以下主要功能:  

- 它们提供了`Iccommand`接口的基本实现。  
- 他们还实现了`IRelayCommand`(和`IRelayCommand<T>`)接口，它公开了一个`NotifyCanExecuteChanged`方法来引发`CanExecuteChanged`事件。  
- 它们公开了接受`Action`和`Func<T>`等委托的构造函数，这允许包装标准方法和lambda表达式。  

## Working with `ICommand`

下面展示了如何设置一个简单的命令:

```csharp
public class MyViewModel : ObservableObject
{
    public MyViewModel()
    {
        IncrementCounterCommand = new RelayCommand(IncrementCounter);
    }

    private int counter;

    public int Counter
    {
        get => counter;
        private set => SetProperty(ref counter, value);
    }

    public ICommand IncrementCounterCommand { get; }

    private void IncrementCounter() => Counter++;
}
```

And the relative UI could then be (using WinUI XAML):

```xml
<Page
    x:Class="MyApp.Views.MyPage"
    xmlns:viewModels="using:MyApp.ViewModels">
    <Page.DataContext>
        <viewModels:MyViewModel x:Name="ViewModel"/>
    </Page.DataContext>

    <StackPanel Spacing="8">
        <TextBlock Text="{x:Bind ViewModel.Counter, Mode=OneWay}"/>
        <Button
            Content="Click me!"
            Command="{x:Bind ViewModel.IncrementCounterCommand}"/>
    </StackPanel>
</Page>
```

`Button`绑定到视图模型中的`Iccommand`，它包装了私有的`IncrementCounter`方法。`TextBlock`显示`Counter`属性的值，并在每次属性值改变时更新

## 示例代码

还有更多的例子 [unit tests](https://github.com/Microsoft/WindowsCommunityToolkit//blob/master/UnitTests/UnitTests.Shared/Mvvm).

## Requirements

| Device family | Universal, 10.0.16299.0 or higher |
| --- | --- |
| Namespace | Microsoft.Toolkit.Mvvm |
| NuGet package | [Microsoft.Toolkit.Mvvm](https://www.nuget.org/packages/Microsoft.Toolkit.Mvvm/) |

## API

* [RelayCommand source code](https://github.com/Microsoft/WindowsCommunityToolkit//blob/master/Microsoft.Toolkit.Mvvm/Input/RelayCommand.cs)
* [RelayCommand&lt;T> source code](https://github.com/Microsoft/WindowsCommunityToolkit//blob/master/Microsoft.Toolkit.Mvvm/Input/RelayCommand{T}.cs)
