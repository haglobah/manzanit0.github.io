---
layout: post
title: Bootstraping a Xamaring Application
author: Javier Garcia
category: xamarin
tags: mobile, xamarin, c#, .net
---

The point of this blogpost is simply to showcase how simple it is to scaffold a
side menu with Xamarin Forms. Also serves me personally as a reference every
time I need to copy-paste the code.

For this tutorial I'm using Jetbrains Rider 2020.3 and .NET Core 3.1. The
reason why I'm saying this is because I'm not sure if projects scaffolding is
the same in Visual Studio or not, so I'd rather mention it. The default project
that gets created with Rider is a single Xamarin Form which says Hello World.
So let's jump straight in and create a new project, mine will be called
`CookbookMobile`.

## Creating a side menu for a cooking app

Creating a side menu is dead simple, there's just not too many to-the-point
tutorials out there. It's a matter of three steps:

1. Creating the `MainMenu` Xamarin Form as a MasterDetail type which I'll scaffold below
2. Start the form in the `App`
3. Import your burguer icon if you wish one and add any default styles

The `MainMenu` Xamarin form will look like the below. Notice that it already
has some styles in the TextColor and Background, which belong to the default
styles we'll provide in the last step. Feel free to simply remove those lines

#### MainMenu.xaml

```xml
<?xml version="1.0" encoding="UTF-8"?>

<MasterDetailPage
    xmlns="http://xamarin.com/schemas/2014/forms"
    xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
    x:Class="CookbookMobile.MainMenu">
    <MasterDetailPage.Master>
        <ContentPage Icon="hamburger_menu.png" Title="Menu" BackgroundColor="LightGray">
            <Grid VerticalOptions="FillAndExpand"  RowSpacing="50">
                <Grid.RowDefinitions>
                    <RowDefinition Height="56" />
                    <RowDefinition Height="*" />
                </Grid.RowDefinitions>
                <Grid Grid.Row="0" Padding="16, 30, 0, -30">
                    <Grid.RowDefinitions>
                        <RowDefinition Height="36" />
                        <RowDefinition Height="20" />
                    </Grid.RowDefinitions>
                    <Label Grid.Row="0"
			               Text="Cookbook"
		                   TextColor="Black"
		                   FontSize="20"
		                   VerticalOptions="Center"
		                   HorizontalOptions="Start" />
                    <Label Grid.Row="1"
                           Text="Save, share and revisit your favourite recipes"
                           TextColor="DarkSlateGray"
                           FontSize="14"
                           VerticalOptions="Center"
                           HorizontalOptions="Start" />
                </Grid>
                <ListView Grid.Row="1"
                          x:Name="MenuListView"
                          ItemsSource="{Binding MainMenuItems}"
                          ItemSelected="MainMenuItem_Selected"
                          VerticalOptions="FillAndExpand"
                          SeparatorVisibility="None"
                          RowHeight="48"
                          BackgroundColor="{StaticResource GhostColour}">
                    <ListView.ItemTemplate>
                        <DataTemplate>
                            <ViewCell>
                                <StackLayout VerticalOptions="FillAndExpand"
                                             Orientation="Horizontal"
                                             Padding="16,10,0,10"
                                             Spacing="20">
                                    <Label Text="{Binding Title}"
                                           FontSize="14"
                                           VerticalOptions="Center"
                                           TextColor="Black" />
                                </StackLayout>
                            </ViewCell>
                        </DataTemplate>
                    </ListView.ItemTemplate>
                </ListView>
            </Grid>
        </ContentPage>
    </MasterDetailPage.Master>
</MasterDetailPage>
```

#### MainMenu.xaml.cs

```cs
namespace CookbookMobile
{

    public class MainMenuItem
    {
        public string Title { get; set; }
        public Type TargetType { get; set; }
    }

    [XamlCompilation(XamlCompilationOptions.Compile)]
    public partial class MainMenu : MasterDetailPage
    {
        public List<MainMenuItem> MainMenuItems { get; set; }

        public MainMenu()
        {
            BindingContext = this;

            MainMenuItems = new List<MainMenuItem>()
            {
                new MainMenuItem {Title = "Recipes", TargetType = typeof(RecipesPage)},
                new MainMenuItem {Title = "Settings", TargetType = typeof(RecipesPage)},
                new MainMenuItem {Title = "About", TargetType = typeof(RecipesPage)},
            };

            Detail = new NavigationPage(new RecipesPage());

            InitializeComponent();
        }

        public void MainMenuItem_Selected(object sender, SelectedItemChangedEventArgs e)
        {
            if (!(e.SelectedItem is MainMenuItem item)) return;

            Detail = new NavigationPage((Page) Activator.CreateInstance(item.TargetType));
            MenuListView.SelectedItem = null;
            IsPresented = false;
        }
    }
}
```

Once we've creating the `MainMenu` form, let's make sure to set the menu as the
main page:

```cs
public partial class App : Application
{
    public App()
    {
        InitializeComponent();
        // Set the menu as the main page.
        MainPage = new MainMenu();
    }

    // ...
}
```

Add some default styles in the `App.xaml` if you wish (notice how these colours,
like the GhostColour, are used in the `MainMenu.xaml`):

```xml
<?xml version="1.0" encoding="utf-8" ?>
<Application xmlns="http://xamarin.com/schemas/2014/forms"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             x:Class="CookbookMobile.App">
    <Application.Resources>
        <ResourceDictionary>
            <Thickness x:Key="PageMargin">20</Thickness>
            <!-- Colors -->
            <Color x:Key="NotSoLightBlueColour">#2A7886</Color>
            <Color x:Key="LightBlueColour">#79BAC1</Color>
            <Color x:Key="GhostColour">AliceBlue</Color>
            <Color x:Key="AubergineColour">#512B58</Color>

            <Color x:Key="AppBackgroundColor">AliceBlue</Color>
            <Color x:Key="NavigationBarColor">#1976D2</Color>
            <Color x:Key="NavigationBarTextColor">White</Color>

            <!-- Implicit styles -->
            <Style TargetType="{x:Type NavigationPage}">
                <Setter Property="BarBackgroundColor" Value="{StaticResource NotSoLightBlueColour}" />
                <Setter Property="BarTextColor" Value="{StaticResource GhostColour}" />
            </Style>

            <Style TargetType="{x:Type ContentPage}" ApplyToDerivedTypes="True">
                <Setter Property="BackgroundColor" Value="{StaticResource GhostColour}" />
            </Style>
        </ResourceDictionary>

    </Application.Resources>
</Application>
```

Lastly, add a `hamburguer_menu.png` of your choice under
`CookbookMobile/CookbookMobile.iOS/Resources` and
`CookbookMobile/CookbookMobile.Android/Resources/drawable`... and we're done.
That's literally it :)

Well, you might be wondering what the `RecipesPage` class which appears in the
`MainMenu.xaml.cs` file is. It's a simple Form which says _"Hello world"_. Feel
free to create your own forms and point to them from the menu.
