---
name: winui-dev
description: >
  开发 Windows 原生应用，重点 WinUI 3（Windows App SDK）。可帮助规划 WinUI/XAML 页面结构、生成 C# 源码与 XAML 界面、处理数据绑定 MVVM、打包发布 MSIX；也用 Minimal API / MVC 的 ASP.NET Core 或控制台覆盖"Windows 程序"开发需求。
  当用户要求"做个 Windows 应用""写 WinUI3""C# + XAML""桌面应用界面""MSIX 打包"时触发。
license: MIT
compatibility: 源码在本机/iSH 生成；构建 + 运行需 Windows 10/11 + Windows App SDK（VS2022 或用 dotnet 命令行 + msix 工具）；此 skill 交付可读可提交源码，编译此步在 Windows 完成
---

# WinUI 3 (Windows App SDK) 桌面开发

为需要 **C# + XAML 写 Windows 桌面应用**的用户服务。angles-cli 环境无 Windows，因此交付**完整、可编译的源码与工程结构**，用户在 Windows + VS2022（或 `dotnet build`）上编译运行。

## When to use

- 用户要"开发 Windows 桌面应用"
- WinUI 3 / Windows App SDK 窗口、布局、控件、数据绑定、打包
- 从 UWP/WPF 迁移，或首次搭 WinUI 3 工程
- MVVM、导航、主题、打包发布相关需求

## WinUI 3 vs 其他方案速选

| 目标 | 建议 | 说明 |
|---|---|---|
| 现代化 Win 原生 UI | **WinUI 3 (Windows App SDK)** | C#/XAML，未打包或 MSIX 均可 |
| 轻量内部工具 | WPF / .NET | 更老但成熟、文档多 |
| Web 技术桌面 | 可考虑 Blazor Hybrid / WebView | 前端团队友好 |
| 简单脚本 | PowerShell / .NET console | 够用不重 |

多数"Win 桌面应用"需求默认给 **WinUI 3 + MVVM**。

## 工程骨架（WinUI 3 未打包，最小可编译）

WinUI 3 用 Windows App SDK。一个最小 app 最少文件：

**MyApp.csproj**（要点：`WindowsAppSDK` 与 `Microsoft.Windows.SDK.BuildTools` 包、`EnableMsixTooling`、net6/8.0-windowsX.0）：

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <TargetFramework>net8.0-windows10.0.19041.0</TargetFramework>
    <UseWinUI>true</UseWinUI>
    <EnableMsixTooling>true</EnableMsixTooling>
    <Platforms>x64;ARM64</Platforms>
    <RuntimeIdentifiers>win-x64</RuntimeIdentifiers>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.WindowsAppSDK" Version="1.5.240607001" />
    <PackageReference Include="Microsoft.Windows.SDK.BuildTools" Version="10.0.22621.3233" />
  </ItemGroup>
</Project>
```

**App.xaml / App.xaml.cs**（启动入口，`Microsoft.UI.Xaml.Application`）：

```xml
<Application x:Class="MyApp.App"
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <Application.Resources></Application.Resources>
</Application>
```

```csharp
using Microsoft.UI.Xaml;

namespace MyApp;
public partial class App : Application
{
    private Window? _window;
    public App() { InitializeComponent(); }

    protected override void OnLaunched(LaunchActivatedEventArgs args)
    {
        _window = new MainWindow();
        _window.Activate();
    }
}
```

**MainWindow.xaml**：

```xml
<Window x:Class="MyApp.MainWindow"
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
  <StackPanel HorizontalAlignment="Center" VerticalAlignment="Center" Spacing="12">
    <TextBlock x:Name="HelloText" Text="Hello, WinUI 3!" FontSize="28"/>
    <Button Content="点我" Click="OnButtonClick"/>
  </StackPanel>
</Window>
```

```csharp
using Microsoft.UI.Xaml;
using Microsoft.UI.Xaml.Controls;

namespace MyApp;
public sealed partial class MainWindow : Window
{
    public MainWindow()
    {
        this.InitializeComponent();
        Title = "我的 WinUI 应用";
    }
    private void OnButtonClick(object sender, RoutedEventArgs e)
    {
        HelloText.Text = "你点击了按钮";
    }
}
```

## MVVM：数据绑定（推荐不是 code-behind）

`INotifyPropertyChanged` 基类 + ViewModel + x:Bind/命令：

```csharp
// ViewModelBase.cs
using System.ComponentModel;
using System.Runtime.CompilerServices;

public abstract class ViewModelBase : INotifyPropertyChanged
{
    public event PropertyChangedEventHandler? PropertyChanged;
    protected void OnPropertyChanged([CallerMemberName] string? n = null)
        => PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(n));
    protected bool Set<T>(ref T f, T v, [CallerMemberName] string? n = null)
    { if (EqualityComparer<T>.Default.Equals(f, v)) return false; f = v; OnPropertyChanged(n); return true; }
}
```

```csharp
// MainViewModel.cs
public class MainViewModel : ViewModelBase
{
    private string _greeting = "默认文案";
    public string Greeting { get => _greeting; set => Set(ref _greeting, value); }
}
```

在视图给 `DataContext` 并 `x:Bind`（`Mode=TwoWay` 双向）、命令用 `Microsoft.Toolkit.Mvvm` 的 `[RelayCommand]`：

```xml
<TextBlock Text="{x:Bind ViewModel.Greeting, Mode=OneWay}"/>
```

初始化：`public MainWindow(){ InitializeComponent(); DataContext = new MainViewModel(); }`

## 绑定表达式要点（避免坑）

```xml
<!-- 单向（数据→UI）显示属性 -->
Text="{x:Bind Name, Mode=OneWay}"
<!-- 双向，输入框回写模型 -->
Text="{x:Bind EditText, Mode=TwoWay, UpdateSourceTrigger=PropertyChanged}"
<!-- 转换器（Bool→Visibility 等）用 x:Bind + Converter 需 IValueConverter -->
Visibility="{x:Bind IsBusy, Converter={StaticResource BoolToVis}, Mode=OneWay}"
```

- **x:Bind** 编译期绑定，性能好、拼错即报错；**Binding** 运行期绑定，容错大。WinUI3 默认都用 x:Bind。
- 派生子属性（`Text="{x:Bind Sub.Prop}"`）需路径对象非 null，用 `x:Bind` 会空引用保护但无 UI 更新，必要时 `Mode=OneWay` already handles change via source INPC。

## 常用布局控件（界面搭建现查）

```xml
<!-- 自适应 Grid -->
<Grid RowDefinitions="Auto,*" ColumnDefinitions="200,*">
    <TextBlock Grid.Row="0" Grid.ColumnSpan="2" Text="标题"/>
    <ListView Grid.Row="1" ItemsSource="{x:Bind Items}">
        <ListView.ItemTemplate>
            <DataTemplate x:DataType="local:ItemModel">
                <TextBlock Text="{x:Bind Name}"/>
            </DataTemplate>
        </ListView.ItemTemplate>
    </ListView>
</Grid>

<!-- StackPanel 垂直堆叠，垂直居中 -->
<StackPanel Spacing="12" Padding="24" HorizontalAlignment="Center"/>

<!-- NavigationView 主导航侧栏 -->
<NavigationView PaneDisplayMode="Left">
    <NavigationViewItem Content="首页" Icon="Home"/>
    <Frame x:Name="ContentFrame"/>
</NavigationView>
```

## 窗口打包 / MSIX（交付发布形态）

**打包（MSIX）推荐**：项目加 `Package.appxmanifest`（Identity Name 等），VS 发布向导 / `dotnet publish` + makeappx。未打包模式用 `WindowsAppSDK` 自带的自举（`WindowsAppRuntime`）启动。

`Package.appxmanifest` 最小：
```xml
<Package ...>
  <Identity Name="MyCompany.MyApp" Publisher="CN=你" Version="1.0.0.0"/>
  <Properties><DisplayName>MyApp</DisplayName></Properties>
  <Applications>
    <Application Id="App" Executable="MyApp.exe">
      <uap:VisualElements DisplayName="MyApp"
         Square150x150Logo="Assets\Logo150.png" .../>
    </Application>
  </Applications>
</Package>
```

## 常见坑

- **TargetFramework 匹配**：WinUI3 要 `net8.0-windows10.0.19041.0` 之类。写成纯 `net8.0` 会在 WinUI API 报 `PlatformNotSupported` / 找不到类型
- **Platforms**：默认 AnyCPU 在 WinAppSDK 常崩，显式 `x64;ARM64;x86`
- **Event vs x:Bind**：code-behind 事件简单任务够用；复杂多 View 坚持 MVVM，别绑乱
- **窗口标题/初始尺寸**：WinUI3 初始大小有限（默认 ~1024 上限逻辑），需 `AppWindow.Resize`；菜单用 `MenuBar`
- **app.manifest DPI**：高 DPI 需声明，否则 WinUI3 自带感知可忽略大多
- **只是小工具**：没必要上 WinUI，给 WPF 更快；判断用户是按"要现代化原生"还是"能跑即可"
- 编译必须 Windows：给代码同时说明"在 Windows 用 VS2022 的 .NET8 桌面工作负载打开 / dotnet build"。别在 Linux 声称能编 WinUI

## 原则

**交付语法正确的 C#/XAML + 可一键打开的 csproj；构建运行在 Windows 上由 VS/dotnet 完成。** MVVM 默认、绑定给表达式、坑要写明，不给运行不了的空壳。
