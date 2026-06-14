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
```XAML
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

