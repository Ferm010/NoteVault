db.cs
```cs
using MySql.Data.MySqlClient;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

namespace project
{
    internal static class Db
    {
        public const string ConnectionString =
            "server=127.0.0.1;port=3306;uid=root;pwd=;database=mydb";

        public static MySqlConnection GetConnection()
        {
            return new MySqlConnection(ConnectionString);
        }
    }
}
```


Session.cs
```cs
namespace project
{
    internal static class Session
    {
        public static int UserId { get; set; } = 0;
        public static string UserName { get; set; } = "";
        public static string RoleName { get; set; } = "";
        public static bool IsGuest { get; set; } = false;
    }
}
```

MainWindow.xaml
```cs
<Window x:Class="project.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
        xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
        xmlns:local="clr-namespace:project"
        mc:Ignorable="d"
        Title="Авторизация" Height="547" Width="817"
        Icon="import/icon.ico"
        WindowStartupLocation="CenterScreen">
    <Grid>
        <Border Background="White" CornerRadius="10" HorizontalAlignment="Center" VerticalAlignment="Center" 
                Width="360" Height="462" BorderBrush="#CCCCCC" BorderThickness="1">
            <Border.Effect>
                <DropShadowEffect BlurRadius="15" ShadowDepth="5" Opacity="0.3"/>
            </Border.Effect>
            <StackPanel Margin="30">
                <TextBlock Text="ООО СтройМатериалы" FontSize="20" FontWeight="Bold" 
                           HorizontalAlignment="Center" Margin="0,0,0,5" Foreground="#333333"/>
                <Image Source="/import/icon.png" Height="70" Margin="0,10,0,20" Stretch="Uniform"/>

                <Label Content="Логин" FontSize="14" Margin="0,0,0,3"/>
                <TextBox x:Name="login" FontSize="14" Margin="0,0,0,15" Padding="8,6"/>

                <Label Content="Пароль" FontSize="14" Margin="0,0,0,3"/>
                <PasswordBox x:Name="passwordBox" FontSize="14" Margin="0,0,0,25" Padding="8,6"/>

                <Button x:Name="Vxod" Content="Вход" FontSize="14" Height="40" 
                        Background="#1976D2" Foreground="White" BorderThickness="0"
                        Cursor="Hand" Margin="0,0,0,10"/>

                <Button x:Name="Guest" Content="Гость" FontSize="14" Height="40"
                        Background="#EEEEEE" Foreground="#333333" BorderThickness="1"
                        Cursor="Hand"/>
            </StackPanel>
        </Border>
    </Grid>
</Window>
```


```cs
using System;
using System.Windows;
using MySql.Data.MySqlClient;

namespace project
{
    public partial class MainWindow : Window
    {
        public MainWindow()
        {
            InitializeComponent();
            Vxod.Click += Vxod_Click;
            Guest.Click += Guest_Click;
        }

        private void Vxod_Click(object sender, RoutedEventArgs e)
        {
            string loginText = login.Text.Trim();
            string passwordText = passwordBox.Password;

            if (string.IsNullOrEmpty(loginText) || string.IsNullOrEmpty(passwordText))
            {
                MessageBox.Show("Заполните все поля", "Ошибка", MessageBoxButton.OK, MessageBoxImage.Warning);
                return;
            }

            MySqlConnection conn = Db.GetConnection();
            conn.Open();
            string sql = "SELECT u.id_user, u.name, r.name FROM users u INNER JOIN role r ON u.role_id = r.id_role WHERE u.login = @login AND u.password = @password";
            MySqlCommand cmd = new MySqlCommand(sql, conn);
            cmd.Parameters.AddWithValue("@login", loginText);
            cmd.Parameters.AddWithValue("@password", passwordText);

            MySqlDataReader reader = cmd.ExecuteReader();
            if (reader.Read())
            {
                Session.UserId = reader.GetInt32(0);
                Session.UserName = reader.GetString(1);
                Session.RoleName = reader.GetString(2);

                Session.IsGuest = false;

                MainMenu mainMenu = new MainMenu();
                mainMenu.Show();
                this.Close();
            }
            else
            {
                MessageBox.Show("Неверный логин или пароль", "Ошибка", MessageBoxButton.OK, MessageBoxImage.Error);
            }
            reader.Close();
        }

        private void Guest_Click(object sender, RoutedEventArgs e)
        {
            Session.IsGuest = true;
            Session.UserName = "Гость";
            Session.RoleName = "Гость";

            MainMenu mainMenu = new MainMenu();
            mainMenu.Show();
            this.Close();
        }
    }
}
```

MainMenu.xaml


```cs
<Window x:Class="project.MainMenu"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
        xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
        xmlns:local="clr-namespace:project"
        mc:Ignorable="d"
        Title="Список товаров" Height="650" Width="1200"
        Icon="import/icon.ico"
        WindowStartupLocation="CenterScreen">
    <Window.Resources>
        <local:DiscountToColorConverter x:Key="DiscountToColor"/>
    </Window.Resources>

    <Grid Margin="15">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
        </Grid.RowDefinitions>

        <!-- Toolbar -->
        <Border Background="#F5F5F5" CornerRadius="6" Padding="12" Margin="0,0,0,12">
            <DockPanel LastChildFill="False">
                <StackPanel Orientation="Horizontal" DockPanel.Dock="Left">
                    <Button x:Name="BtnEdit" Content="Редактировать" Width="130" Margin="0,0,8,0" Height="34"/>
                    <Button x:Name="BtnAdd" Content="Добавить" Width="100" Margin="0,0,8,0" Height="34"/>
                    <Button x:Name="BtnOrders" Content="Заказы" Width="100" Height="34"/>
                </StackPanel>

                <StackPanel Orientation="Horizontal" DockPanel.Dock="Right">
                    <TextBlock Text="{Binding UserNameDisplay}" FontWeight="Bold" Foreground="#333" VerticalAlignment="Center" Margin="0,0,5,0"/>
                    <TextBlock Text="|" Foreground="#999" Margin="5,0,5,0" VerticalAlignment="Center"/>
                    <TextBlock Text="{Binding RoleDisplay}" Foreground="#666" VerticalAlignment="Center" Margin="0,0,10,0"/>
                    <Button x:Name="BtnLogout" Content="Выйти" Width="80" Height="30" Background="#E53935" Foreground="White" BorderThickness="0"/>
                </StackPanel>
            </DockPanel>
        </Border>

        <!-- Product Cards Grid -->
        <ScrollViewer Grid.Row="1" VerticalScrollBarVisibility="Auto">
            <ItemsControl x:Name="ProductsGrid" ItemsSource="{Binding Products}">
                <ItemsControl.ItemsPanel>
                    <ItemsPanelTemplate>
                        <WrapPanel Orientation="Horizontal"/>
                    </ItemsPanelTemplate>
                </ItemsControl.ItemsPanel>

                <ItemsControl.ItemTemplate>
                    <DataTemplate>
                        <Border BorderBrush="#CCCCCC" BorderThickness="1.5" CornerRadius="6" Margin="8" Padding="8">
                            <Border.Style>
                                <Style TargetType="Border">
                                    <Setter Property="Background" Value="White"/>
                                    <Setter Property="BorderBrush" Value="#CCCCCC"/>
                                    <EventSetter Event="Border.MouseLeftButtonDown" Handler="Card_Click"/>
                                    <Style.Triggers>
                                        <DataTrigger Binding="{Binding IsSelected}" Value="True">
                                            <Setter Property="BorderBrush" Value="#DAA520"/>
                                            <Setter Property="Background" Value="#E3F0FF"/>
                                        </DataTrigger>
                                        <Trigger Property="IsMouseOver" Value="True">
                                            <Setter Property="BorderBrush" Value="#DAA520"/>
                                            <Setter Property="Background" Value="#F0F7FF"/>
                                        </Trigger>
                                    </Style.Triggers>
                                </Style>
                            </Border.Style>

                            <Grid Margin="3">
                                <Grid.ColumnDefinitions>
                                    <ColumnDefinition Width="100" />
                                    <ColumnDefinition Width="*" />   
                                </Grid.ColumnDefinitions>

                                <!-- Photo -->
                                <Border Grid.Column="0" BorderBrush="#DDDDDD" BorderThickness="1" Background="#F5F5F5">
                                    <Image Source="{Binding PhotoPath}" Stretch="UniformToFill"/>
                                </Border>

                                <!-- Details -->
                                <StackPanel Grid.Column="1" Margin="8,2">
                                    <TextBlock Text="{Binding CategoryAndName}" FontWeight="Bold" FontSize="12" Foreground="#333" TextTrimming="CharacterEllipsis"/>

                                    <Grid Margin="0,4,0,0">
                                        <Grid.RowDefinitions>
                                            <RowDefinition Height="Auto"/>
                                            <RowDefinition Height="Auto"/>
                                            <RowDefinition Height="Auto"/>
                                            <RowDefinition Height="Auto"/>
                                        </Grid.RowDefinitions>

                                        <!-- Article + Manufacturer -->
                                        <TextBlock Grid.Row="0" Text="{Binding ArticleDisplay}" Foreground="#777" FontSize="10" Margin="0,1"/>

                                        <!-- Description -->
                                        <TextBlock Grid.Row="1" Text="{Binding Desc}" TextWrapping="NoWrap" TextTrimming="CharacterEllipsis" Foreground="#666" FontSize="10" Margin="0,2"/>

                                        <!-- Price + Unit -->
                                        <StackPanel Grid.Row="2" Orientation="Horizontal" Margin="0,3">
                                            <TextBlock Text="{Binding PriceFormatted}" FontWeight="SemiBold" FontSize="14" Foreground="#1976D2"/>
                                            <TextBlock Text="{Binding UnitName}" Foreground="#888" FontSize="10" Margin="6,0,0,0"/>
                                        </StackPanel>

                                        <!-- Amount -->
                                        <TextBlock Grid.Row="3" Text="{Binding amountText}" Foreground="#666" FontSize="10" Margin="0,2"/>
                                    </Grid>
                                </StackPanel>

                                <!-- Discount Badge -->
                                <Border Background="{Binding discount, Converter={StaticResource DiscountToColor}}" CornerRadius="4" HorizontalAlignment="Left" VerticalAlignment="Top" Margin="-2,-8,0,0">
                                    <TextBlock Text="{Binding DiscountText}" 
                                               Foreground="White" FontWeight="Bold" FontSize="11"
                                               TextAlignment="Center" VerticalAlignment="Center" Padding="5,2" MinWidth="36"/>
                                </Border>
                            </Grid>
                        </Border>
                    </DataTemplate>
                </ItemsControl.ItemTemplate>
            </ItemsControl>
        </ScrollViewer>
    </Grid>
</Window>

```


```cs

```
