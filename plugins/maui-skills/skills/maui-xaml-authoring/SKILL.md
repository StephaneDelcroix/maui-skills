---
name: maui-xaml-authoring
description: >-
  Guardrails and reference for writing correct .NET MAUI XAML. Prevents the most
  common AI-generated XAML mistakes: wrong namespace URIs, missing x:DataType,
  bad binding paths, incorrect OnPlatform syntax, Grid definition errors, obsolete
  controls, and resource/style misuse. Derived from ~305 XAML regression tests and
  38+ diagnostic codes in the dotnet/maui repository.
  USE FOR: "XAML", "xaml page", "ContentPage", "binding", "x:DataType",
  "DataTemplate", "Grid", "StackLayout", "ResourceDictionary", "Style",
  "OnPlatform", "OnIdiom", "markup extension", "StaticResource",
  "CollectionView ItemTemplate", "Shell XAML", "xmlns", "namespace".
  DO NOT USE FOR: compiled C# bindings with SetBinding lambdas (use maui-data-binding),
  XAML hot reload diagnostics (use maui-hot-reload-diagnostics),
  Xamarin.Forms migration (use xamarin-forms-migration).
---

# .NET MAUI XAML Authoring

## Root Element & Namespace Declarations

Every XAML file must start with the XML processing instruction and the correct namespace URIs.

```xml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             x:Class="MyApp.Views.MyPage">
```

| Prefix | URI | Includes |
|--------|-----|----------|
| *(default)* | `http://schemas.microsoft.com/dotnet/2021/maui` | Controls, Shapes, `Microsoft.Maui`, `Microsoft.Maui.Graphics` |
| `x:` | `http://schemas.microsoft.com/winfx/2009/xaml` | XAML directives and `System` primitives |

```xml
<!-- ❌ Xamarin.Forms namespace — outdated -->
xmlns="http://xamarin.com/schemas/2014/forms"

<!-- ❌ Wrong year for x: prefix -->
xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"

<!-- ❌ Missing x:Class when a code-behind file exists -->
<ContentPage xmlns="...">
```

### Importing Custom Types

```xml
<!-- Same assembly -->
xmlns:local="clr-namespace:MyApp.ViewModels"

<!-- External assembly -->
xmlns:toolkit="clr-namespace:CommunityToolkit.Maui.Views;assembly=CommunityToolkit.Maui"
```

Do **not** invent XML namespace URIs for your own types. Only framework assemblies register `XmlnsDefinition` URIs.

---

## Compiled Bindings & x:DataType

Always set `x:DataType` on the root element (or any element introducing a new `BindingContext`) to enable compiled bindings. Without it bindings use reflection, and property-name typos are silent.

```xml
<ContentPage xmlns:vm="clr-namespace:MyApp.ViewModels"
             x:DataType="vm:MainViewModel">
    <Label Text="{Binding Title}" />
</ContentPage>
```

### DataTemplate Requires Its Own x:DataType

`DataTemplate` creates a new binding scope — always redeclare:

```xml
<CollectionView ItemsSource="{Binding People}">
    <CollectionView.ItemTemplate>
        <DataTemplate x:DataType="model:Person">
            <Label Text="{Binding FullName}" />
        </DataTemplate>
    </CollectionView.ItemTemplate>
</CollectionView>
```

### Opting Out for a Single Binding

```xml
<Label Text="{Binding DynamicPath}" x:DataType="{x:Null}" />
```

### Common Mistakes

```xml
<!-- ❌ No x:DataType — bindings are reflection-based, no compile-time checking -->
<ContentPage>
    <Label Text="{Binding Title}" />
</ContentPage>

<!-- ❌ x:DataType on wrong type — compiles, crashes at runtime -->
<ContentPage x:DataType="model:Person">
    <Label Text="{Binding Title}" />  <!-- Person has no Title -->
</ContentPage>

<!-- ❌ x:DataType on Picker for ItemDisplayBinding — should be item type, not page VM -->
<Picker x:DataType="vm:MainViewModel"
        ItemDisplayBinding="{Binding Name}" />
```

---

## Binding Expression Syntax

```xml
<!-- Basic -->
<Entry Text="{Binding Email, Mode=TwoWay}" />

<!-- Property path with dot notation -->
<Label Text="{Binding User.Address.City}" />

<!-- StringFormat — wrap in single quotes when it contains commas or braces -->
<Label Text="{Binding Price, StringFormat='Total: {0:C2}'}" />

<!-- Escape leading brace in StringFormat with {} -->
<Label Text="{Binding Count, StringFormat='{}{0} items'}" />
```

Valid `Mode` values: `Default`, `OneWay`, `OneWayToSource`, `TwoWay`, `OneTime`.

| Mistake | Example | Fix |
|---------|---------|-----|
| Unclosed expression | `{Binding Name` | `{Binding Name}` |
| Empty indexer | `{Binding Items[]}` | `{Binding Items[0]}` |
| Wrong property path | `{Binding Nmae}` | Match ViewModel property name exactly |
| ConverterParameter binding | `ConverterParameter={Binding X}` | Not supported — use a static value or `x:Reference` |

---

## Grid

### Compact Syntax (preferred for simple grids)

```xml
<Grid RowDefinitions="Auto, *, 50"
      ColumnDefinitions="*, 2*, Auto"
      ColumnSpacing="10" RowSpacing="5">

    <Label Grid.Row="0" Grid.Column="0" Text="Name" />
    <Entry Grid.Row="0" Grid.Column="1" Grid.ColumnSpan="2" />
    <Button Grid.Row="2" Grid.Column="0" Grid.ColumnSpan="3" Text="Submit" />
</Grid>
```

### Verbose Syntax

```xml
<Grid>
    <Grid.RowDefinitions>
        <RowDefinition Height="Auto" />
        <RowDefinition Height="*" />
    </Grid.RowDefinitions>
    <Grid.ColumnDefinitions>
        <ColumnDefinition Width="200" />
        <ColumnDefinition Width="*" />
    </Grid.ColumnDefinitions>
</Grid>
```

### GridLength Values

| Value | Meaning |
|-------|---------|
| `Auto` | Size to content |
| `*` | Proportional (same as `1*`) |
| `2*` | Twice the proportion of `*` |
| `100` | Fixed 100 device-independent pixels |

### Common Grid Mistakes

```xml
<!-- ❌ Children without Grid.Row / Grid.Column all stack in cell (0,0) -->
<Grid RowDefinitions="Auto, Auto">
    <Label Text="A" />
    <Label Text="B" />   <!-- overlaps A at (0,0) -->
</Grid>

<!-- ✅ Fix -->
<Grid RowDefinitions="Auto, Auto">
    <Label Grid.Row="0" Text="A" />
    <Label Grid.Row="1" Text="B" />
</Grid>

<!-- ❌ Grid.Row="1" with only one RowDefinition — row does not exist -->
<Grid RowDefinitions="Auto">
    <Label Grid.Row="1" Text="Oops" />
</Grid>
```

---

## StackLayout vs VerticalStackLayout / HorizontalStackLayout

Always prefer the specific variants — they are faster and lighter:

```xml
<!-- ✅ Preferred -->
<VerticalStackLayout Spacing="10">
    <Label Text="Title" />
    <Label Text="Subtitle" />
</VerticalStackLayout>

<!-- ❌ Slower legacy -->
<StackLayout Orientation="Vertical" Spacing="10">
    <Label Text="Title" />
    <Label Text="Subtitle" />
</StackLayout>
```

---

## Resources & Styles

### Declaring Resources

Every resource in a `ResourceDictionary` must have an `x:Key`:

```xml
<ContentPage.Resources>
    <Color x:Key="Primary">#512BD4</Color>
    <Style x:Key="TitleStyle" TargetType="Label">
        <Setter Property="FontSize" Value="24" />
        <Setter Property="FontAttributes" Value="Bold" />
    </Style>
</ContentPage.Resources>
```

### Implicit Styles

Omit `x:Key` — applies to all controls of that `TargetType` in scope:

```xml
<Style TargetType="Button">
    <Setter Property="BackgroundColor" Value="{StaticResource Primary}" />
</Style>
```

### StaticResource vs DynamicResource

| Extension | Resolved | Use when |
|-----------|----------|----------|
| `{StaticResource Key}` | Once at load | Value never changes |
| `{DynamicResource Key}` | On every change | Theme colors, runtime-switchable values |

### Common Resource Mistakes

```xml
<!-- ❌ Missing x:Key -->
<Color>#512BD4</Color>

<!-- ❌ Referencing resource before it is defined -->
<Label TextColor="{StaticResource Primary}" />
<ContentPage.Resources>
    <Color x:Key="Primary">#512BD4</Color>
</ContentPage.Resources>

<!-- ❌ Duplicate x:Key — ArgumentException at runtime -->
<Color x:Key="Primary">#512BD4</Color>
<Color x:Key="Primary">#FF0000</Color>

<!-- ❌ StaticResource for a theme color that changes at runtime -->
<Label TextColor="{StaticResource ThemeText}" />
<!-- ✅ Use DynamicResource -->
<Label TextColor="{DynamicResource ThemeText}" />
```

---

## OnPlatform & OnIdiom

### Inline Markup Extension

```xml
<Label FontSize="{OnPlatform iOS=14, Android=16, WinUI=15}"
       Margin="{OnIdiom Phone='10,5', Tablet='20,10'}" />
```

### Element Syntax (for complex values)

```xml
<Label.FontSize>
    <OnPlatform x:TypeArguments="x:Double">
        <On Platform="iOS">14</On>
        <On Platform="Android">16</On>
    </OnPlatform>
</Label.FontSize>
```

### Platform Names

| Platform | Name in XAML |
|----------|-------------|
| Android | `Android` |
| iOS | `iOS` |
| Mac Catalyst | `MacCatalyst` |
| Windows | `WinUI` |

### Common Mistakes

```xml
<!-- ❌ Missing x:TypeArguments in element syntax -->
<OnPlatform>
    <On Platform="iOS">14</On>
</OnPlatform>

<!-- ❌ Wrong platform name -->
<Label FontSize="{OnPlatform Windows=16}" />   <!-- Should be WinUI -->
<Label FontSize="{OnPlatform macOS=16}" />      <!-- Should be MacCatalyst -->

<!-- ❌ No Default fallback — value is default(T) on unlisted platforms -->
<Label FontSize="{OnPlatform iOS=14}" />
<!-- ✅ Provide Default -->
<Label FontSize="{OnPlatform Default=14, iOS=16}" />
```

---

## Common Control Patterns

### CollectionView with DataTemplate

```xml
<CollectionView ItemsSource="{Binding Items}"
                SelectionMode="Single"
                SelectionChangedCommand="{Binding SelectCommand}">
    <CollectionView.ItemTemplate>
        <DataTemplate x:DataType="model:Item">
            <Grid Padding="10" ColumnDefinitions="Auto, *">
                <Image Grid.Column="0" Source="{Binding Icon}" HeightRequest="40" />
                <Label Grid.Column="1" Text="{Binding Name}" VerticalOptions="Center" />
            </Grid>
        </DataTemplate>
    </CollectionView.ItemTemplate>
</CollectionView>
```

```xml
<!-- ❌ ScrollView wrapping CollectionView — CollectionView scrolls natively -->
<ScrollView>
    <CollectionView ItemsSource="{Binding Items}" />
</ScrollView>

<!-- ❌ DataTemplate without x:DataType — uncompiled bindings -->
<DataTemplate>
    <Label Text="{Binding Name}" />
</DataTemplate>
```

### Shell Routing

```xml
<Shell xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
       xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
       xmlns:views="clr-namespace:MyApp.Views"
       x:Class="MyApp.AppShell">

    <FlyoutItem Title="Home" Icon="home.png">
        <ShellContent ContentTemplate="{DataTemplate views:HomePage}" />
    </FlyoutItem>
</Shell>
```

```xml
<!-- ❌ Page instance instead of DataTemplate — no lazy creation -->
<ShellContent>
    <views:HomePage />
</ShellContent>

<!-- ✅ Lazy creation with DataTemplate -->
<ShellContent ContentTemplate="{DataTemplate views:HomePage}" />
```

---

## Event Handlers & Commands

```xml
<!-- Event handler (code-behind) -->
<Button Text="Click Me" Clicked="OnButtonClicked" />

<!-- Command binding (MVVM) -->
<Button Text="Submit" Command="{Binding SubmitCommand}" CommandParameter="42" />
```

```xml
<!-- ❌ Binding both Clicked and Command — pick one pattern -->
<Button Clicked="OnClick" Command="{Binding ClickCommand}" />

<!-- ❌ Event handler name typo — XamlParseException at runtime -->
<Button Clicked="OnButonClicked" />
```

The code-behind handler must match the expected signature:

```csharp
void OnButtonClicked(object sender, EventArgs e) { }
```

---

## Value Converters

```xml
<ContentPage.Resources>
    <local:BoolToColorConverter x:Key="BoolToColor" />
</ContentPage.Resources>

<Label TextColor="{Binding IsActive, Converter={StaticResource BoolToColor}}" />
```

```xml
<!-- ❌ Converter not registered in Resources -->
<Label TextColor="{Binding IsActive, Converter={StaticResource BoolToColor}}" />
<!-- throws StaticResourceNotFoundException -->

<!-- ❌ ConverterParameter cannot be a Binding -->
ConverterParameter={Binding Threshold}   <!-- not supported -->
<!-- ✅ Use a literal or x:Static -->
ConverterParameter=50
```

---

## Attached Properties

Use dot-notation with the owner type:

```xml
<Label Grid.Row="1" Grid.Column="0"
       SemanticProperties.HeadingLevel="Level1"
       Shell.NavBarIsVisible="False" />
```

```xml
<!-- ❌ Bare property name — does not resolve -->
<Label Row="1" />
<!-- ✅ Attached property syntax -->
<Label Grid.Row="1" />
```

---

## Obsolete APIs — Do Not Use

| ❌ Obsolete | ✅ Modern Replacement |
|------------|----------------------|
| `StackLayout` | `VerticalStackLayout` / `HorizontalStackLayout` |
| `Frame` | `Border` |
| `Device.BeginInvokeOnMainThread` | `Dispatcher.Dispatch()` |
| `Application.MainPage = …` | `Window.Page = …` |
| `RelativeLayout` | `Grid` with proportional sizing |

---

## x: Directives Quick Reference

| Directive | Purpose | Example |
|-----------|---------|---------|
| `x:Class` | Code-behind class | `x:Class="MyApp.MainPage"` |
| `x:Name` | Element name for code-behind | `x:Name="myButton"` |
| `x:Key` | Dictionary key (resources) | `x:Key="Primary"` |
| `x:DataType` | Compiled binding type | `x:DataType="vm:MainViewModel"` |
| `x:TypeArguments` | Generic type args | `x:TypeArguments="x:Double"` |
| `x:Static` | Static member | `{x:Static local:Colors.Primary}` |
| `x:Type` | Type reference | `{x:Type Button}` |
| `x:Reference` | Named element | `{x:Reference slider1}` |
| `x:Null` | Null value | `{x:Null}` |
| `x:Array` | Inline array | `<x:Array Type="{x:Type x:String}">` |
| `x:Arguments` | Constructor args | `<x:Arguments><x:Int32>5</x:Int32></x:Arguments>` |
| `x:FieldModifier` | Field visibility | `x:FieldModifier="public"` |

### Common x: Mistakes

```xml
<!-- ❌ x:Name inside DataTemplate — names are template-scoped, not accessible in code-behind -->
<DataTemplate>
    <Label x:Name="itemLabel" />
</DataTemplate>

<!-- ❌ x:DataType without the CLR namespace import -->
<ContentPage x:DataType="MainViewModel" />
<!-- ✅ Import the namespace -->
<ContentPage xmlns:vm="clr-namespace:MyApp.ViewModels"
             x:DataType="vm:MainViewModel" />
```

---

## Markup Extension Cheat Sheet

| Extension | Purpose | Example |
|-----------|---------|---------|
| `{Binding}` | Data binding | `{Binding Name, Mode=TwoWay}` |
| `{StaticResource}` | Static resource lookup | `{StaticResource MyStyle}` |
| `{DynamicResource}` | Dynamic resource lookup | `{DynamicResource ThemeColor}` |
| `{x:Static}` | Static member | `{x:Static local:App.Version}` |
| `{x:Type}` | Type reference | `{x:Type views:HomePage}` |
| `{x:Reference}` | Named element | `{x:Reference slider1}` |
| `{x:Null}` | Null value | `{x:Null}` |
| `{TemplateBinding}` | ControlTemplate binding | `{TemplateBinding Content}` |
| `{OnPlatform}` | Platform-specific | `{OnPlatform iOS=14, Android=16}` |
| `{OnIdiom}` | Idiom-specific | `{OnIdiom Phone=12, Tablet=18}` |
| `{AppThemeBinding}` | Light/Dark theme | `{AppThemeBinding Light=White, Dark=Black}` |
| `{RelativeSource}` | Relative binding | `{RelativeSource AncestorType={x:Type Grid}}` |
| `{DataTemplate}` | Lazy page creation | `{DataTemplate views:MyPage}` |
