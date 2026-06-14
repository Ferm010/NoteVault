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
using System;
using System.Collections.ObjectModel;
using System.ComponentModel;
using System.Windows;
using System.Windows.Controls;
using System.Windows.Data;
using System.Windows.Input;
using System.Windows.Media;
using MySql.Data.MySqlClient;

namespace project
{
    public class DiscountToColorConverter : IValueConverter
    {
        public object Convert(object value, Type targetType, object parameter, System.Globalization.CultureInfo culture)
        {
            int discount = (int)value;
            if (discount > 12) return "#F4A460";
            if (discount > 0) return "#E53935";
            return "Transparent";
        }

        public object ConvertBack(object value, Type targetType, object parameter, System.Globalization.CultureInfo culture)
        {
            throw new NotImplementedException();
        }
    }

    public partial class MainMenu : Window, INotifyPropertyChanged
    {
        public ObservableCollection<ProductItem> Products { get; set; } = new ObservableCollection<ProductItem>();
        private ProductItem _selectedProduct;

        public string UserNameDisplay => Session.IsGuest ? "Гость" : Session.UserName;
        public string RoleDisplay => Session.IsGuest ? "" : "(" + Session.RoleName + ")";

        public ProductItem SelectedProduct
        {
            get { return _selectedProduct; }
            set
            {
                _selectedProduct = value;
                OnPropertyChanged(nameof(SelectedProduct));
            }
        }

        public event PropertyChangedEventHandler PropertyChanged;
        protected void OnPropertyChanged(string name) { PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(name)); }

        public MainMenu()
        {
            InitializeComponent();
            DataContext = this;

            BtnEdit.Click += BtnEdit_Click;
            BtnAdd.Click += BtnAdd_Click;
            BtnOrders.Click += BtnOrders_Click;
            BtnLogout.Click += BtnLogout_Click;
            ProductsGrid.MouseDoubleClick += ProductsGrid_MouseDoubleClick;

            LoadProducts();
        }

        private void LoadProducts()
        {
            try
            {
                Products.Clear();
                MySqlConnection conn = Db.GetConnection();
                conn.Open();

                string sql = @"SELECT i.id_item, i.article, i.name, i.unit_name, i.price, i.discount, i.amount, 
                                      `i`.`desc`, `i`.`photo`,
                                      s.name AS suplier_name, m.name AS manufacturs_name, c.name AS category_name
                               FROM items i
                               LEFT JOIN suplier s ON i.suplier_id = s.id_suplier
                               LEFT JOIN manufacturs m ON i.manufacturs_id = m.id_manufacturs
                               LEFT JOIN category c ON i.category_id = c.id_category
                               ORDER BY i.id_item";

                MySqlCommand cmd = new MySqlCommand(sql, conn);
                MySqlDataReader reader = cmd.ExecuteReader();

            while (reader.Read())
            {
                int id = reader.GetInt32("id_item");
                string article = reader.GetString("article");
                string name = reader.GetString("name");
                string unit_name = reader.GetString("unit_name");
                decimal price = reader.GetDecimal("price");
                int discount = 0;
                if (!reader.IsDBNull(reader.GetOrdinal("discount")))
                    discount = reader.GetInt32("discount");

                int amount = 0;
                if (!reader.IsDBNull(reader.GetOrdinal("amount")))
                    amount = reader.GetInt32("amount");
                string desc = reader.IsDBNull(reader.GetOrdinal("desc")) ? "" : reader.GetString("desc");
                string photo = reader.GetString("photo");

                string suplier_name = "Не указан";
                if (!reader.IsDBNull(reader.GetOrdinal("suplier_name")))
                    suplier_name = reader.GetString("suplier_name");

                string manufacturs_name = "Не указан";
                if (!reader.IsDBNull(reader.GetOrdinal("manufacturs_name")))
                    manufacturs_name = reader.GetString("manufacturs_name");

                string category_name = "Не указан";
                if (!reader.IsDBNull(reader.GetOrdinal("category_name")))
                    category_name = reader.GetString("category_name");

                string priceFormatted = price.ToString("F2") + " руб.";
                string amountText = "В наличии: " + amount + " шт.";

                // Формируем путь к фото
                string photoPath = "";
                if (!string.IsNullOrEmpty(photo) && System.IO.File.Exists(System.IO.Path.Combine(@"C:\Users\Ferm\Desktop\store\project\import", photo)))
                {
                    photoPath = @"pack://application:,,,/import/" + photo;
                }
                else
                {
                    photoPath = @"pack://application:,,,/import/picture.png";
                }

                string discountText = discount > 0 ? "-" + discount + "%" : "";

                Products.Add(new ProductItem
                {
                    id_item = id,
                    article = article,
                    name = name,
                    unit_name = unit_name,
                    price = price,
                    priceFormatted = priceFormatted,
                    discount = discount,
                    amount = amount,
                    amountText = amountText,
                    desc = desc,
                    photo = photo,
                    suplier_name = suplier_name,
                    manufacturs_name = manufacturs_name,
                    category_name = category_name,
                    PhotoPath = photoPath,
                    DiscountText = discountText,
                    ManufactursName = manufacturs_name,
                    SuplierName = suplier_name,
                    UnitName = unit_name,
                    ArticleDisplay = "Арт: " + article + " | " + manufacturs_name
                });
            }

            reader.Close();

            if (Products.Count > 0)
                SelectedProduct = Products[0];
            }
            catch (Exception ex)
            {
                MessageBox.Show("Ошибка загрузки товаров: " + ex.Message, "Ошибка", MessageBoxButton.OK, MessageBoxImage.Error);
            }
        }

        private void Card_Click(object sender, MouseButtonEventArgs e)
        {
            Border border = sender as Border;
            if (border?.DataContext is ProductItem item)
            {
                foreach (var p in Products)
                    p.IsSelected = false;
                item.IsSelected = true;
                SelectedProduct = item;
            }
        }

        private void ProductsGrid_MouseDoubleClick(object sender, MouseButtonEventArgs e)
        {
            if (SelectedProduct != null)
            {
                OpenEditWindow(SelectedProduct);
            }
        }

        private void BtnEdit_Click(object sender, RoutedEventArgs e)
        {
            if (SelectedProduct == null)
            {
                MessageBox.Show("Выберите товар для редактирования", "Внимание", MessageBoxButton.OK, MessageBoxImage.Information);
                return;
            }
            OpenEditWindow(SelectedProduct);
        }

        private void BtnAdd_Click(object sender, RoutedEventArgs e)
        {
            OpenEditWindow(null);
        }

        private void BtnOrders_Click(object sender, RoutedEventArgs e)
        {
            OrdersMenu ordersWindow = new OrdersMenu();
            ordersWindow.ShowDialog();
        }

        private void BtnLogout_Click(object sender, RoutedEventArgs e)
        {
            Session.UserId = 0;
            Session.UserName = "";
            Session.RoleName = "";
            Session.IsGuest = false;

            MainWindow loginWindow = new MainWindow();
            loginWindow.Show();
            this.Close();
        }

        private void OpenEditWindow(ProductItem product)
        {
            if (Session.IsGuest && product != null)
            {
                MessageBox.Show("Гость не может редактировать товары", "Доступ запрещен", MessageBoxButton.OK, MessageBoxImage.Warning);
                return;
            }

            Edit editWindow = new Edit(product);
            bool? result = editWindow.ShowDialog();

            if (result == true)
            {
                LoadProducts();
                if (product != null && Products.Count > 0)
                {
                    ProductItem found = null;
                    foreach (var p in Products)
                    {
                        if (p.id_item == product.id_item)
                        {
                            found = p;
                            break;
                        }
                    }
                    SelectedProduct = found ?? Products[0];
                }
                else if (Products.Count > 0)
                {
                    SelectedProduct = Products[0];
                }
            }
        }

        public ProductItem FindProductById(int id)
        {
            foreach (var p in Products)
            {
                if (p.id_item == id) return p;
            }
            return null;
        }
    }

    public class ProductItem : INotifyPropertyChanged
    {
        public int id_item { get; set; }
        public string article { get; set; } = "";
        public string name { get; set; } = "";
        public string unit_name { get; set; } = "";
        public decimal price { get; set; }
        public string priceFormatted { get; set; } = "";
        public int discount { get; set; }
        public int amount { get; set; }
        public string amountText { get; set; } = "";
        public string desc { get; set; } = "";
        public string photo { get; set; } = "";
        public string suplier_name { get; set; } = "";
        public string manufacturs_name { get; set; } = "";
        public string category_name { get; set; } = "";

        // Дополнительные свойства для отображения
        public string PhotoPath { get; set; } = @"pack://application:,,,/import/picture.png";

        private string _discountText;
        public string DiscountText
        {
            get => _discountText;
            set { _discountText = value; PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(nameof(DiscountText))); }
        }

        public string CategoryAndName => category_name + (string.IsNullOrEmpty(category_name) ? "" : " / ") + name;

        public string DiscountBackgroundColor => discount > 12 ? "#F4A460" : (discount > 0 ? "#E53935" : "Transparent");

        // Свойства с camelCase для корректного биндинга XAML
        private string _manufactursName;
        public string ManufactursName
        {
            get => _manufactursName;
            set { _manufactursName = value; PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(nameof(ManufactursName))); }
        }

        private string _suplierName;
        public string SuplierName
        {
            get => _suplierName;
            set { _suplierName = value; PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(nameof(SuplierName))); }
        }

        private string _unitName;
        public string UnitName
        {
            get => _unitName;
            set { _unitName = value; PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(nameof(UnitName))); }
        }

        private string _articleDisplay;
        public string ArticleDisplay
        {
            get => _articleDisplay;
            set { _articleDisplay = value; PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(nameof(ArticleDisplay))); }
        }

        private bool _isSelected;
        public bool IsSelected
        {
            get => _isSelected;
            set
            {
                if (_isSelected != value)
                {
                    _isSelected = value;
                    PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(nameof(IsSelected)));
                }
            }
        }

        public event PropertyChangedEventHandler PropertyChanged;
    }

    public class OrderItemModel
    {
        public int Id { get; set; }
        public string Article { get; set; } = "";
        public string OrderDate { get; set; } = "";
        public string DeliveryDate { get; set; } = "";
        public string PickupPoint { get; set; } = "";
        public string CreatedBy { get; set; } = "";
        public int Code { get; set; }
        public string Status { get; set; } = "";

        public string ItemsList { get; set; } = "";
        public string Name { get; set; } = "";
        public int Quantity { get; set; }
        public string PriceFormatted { get; set; } = "";
    }
}

```

ordersmenu.xaml


```cs
<Window x:Class="project.OrdersMenu"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
        xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
        mc:Ignorable="d"
        Title="Заказы" Height="700" Width="1100"
        WindowStartupLocation="CenterScreen">
    <Grid Margin="15">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>

        <!-- Header -->
        <Border Background="#F5F5F5" CornerRadius="6" Padding="12" Margin="0,0,0,10">
            <DockPanel LastChildFill="False">
                <TextBlock Text="Список заказов" FontSize="18" FontWeight="Bold" DockPanel.Dock="Left"/>
                <Button x:Name="BtnBack" Content="Назад" Width="90" Height="32" DockPanel.Dock="Right"/>
            </DockPanel>
        </Border>

        <!-- Orders DataGrid -->
        <DataGrid x:Name="OrdersGrid" Grid.Row="1" AutoGenerateColumns="False" 
                  CanUserAddRows="False" IsReadOnly="True" SelectionMode="Single"
                  SelectionChanged="OrdersGrid_SelectionChanged"
                  BorderThickness="1" Background="White" Margin="0,0,0,10">
            <DataGrid.Columns>
                <DataGridTextColumn Header="ID" Binding="{Binding Id}" Width="60"/>
                <DataGridTextColumn Header="Код заказа" Binding="{Binding Code}" Width="90"/>
                <DataGridTextColumn Header="Дата заказа" Binding="{Binding OrderDate}" Width="120"/>
                <DataGridTextColumn Header="Дата доставки" Binding="{Binding DeliveryDate}" Width="120"/>
                <DataGridTextColumn Header="Пункт выдачи" Binding="{Binding PickupPoint}" Width="200"/>
                <DataGridTextColumn Header="Пользователь" Binding="{Binding CreatedBy}" Width="130"/>
                <DataGridTextColumn Header="Статус" Binding="{Binding Status}" Width="120"/>
            </DataGrid.Columns>
        </DataGrid>

        <!-- Order Items Detail -->
        <Border Grid.Row="2" BorderBrush="#DDDDDD" BorderThickness="1" CornerRadius="6" Padding="10">
            <StackPanel>
                <TextBlock Text="Состав заказа:" FontWeight="Bold" Margin="0,0,0,5"/>
                <DataGrid x:Name="OrderItemsGrid" AutoGenerateColumns="False" 
                          CanUserAddRows="False" IsReadOnly="True" SelectionMode="Single"
                          Height="150" Background="White">
                    <DataGrid.Columns>
                        <DataGridTextColumn Header="Артикул" Binding="{Binding Article}" Width="120"/>
                        <DataGridTextColumn Header="Наименование" Binding="{Binding Name}" Width="250"/>
                        <DataGridTextColumn Header="Количество" Binding="{Binding Quantity}" Width="100"/>
                        <DataGridTextColumn Header="Цена" Binding="{Binding PriceFormatted}" Width="100"/>
                    </DataGrid.Columns>
                </DataGrid>
            </StackPanel>
        </Border>
    </Grid>
</Window>
```


```cs
using System;
using System.Collections.ObjectModel;
using System.Windows;
using MySql.Data.MySqlClient;

namespace project
{
    public partial class OrdersMenu : Window
    {
        private ObservableCollection<OrderItemModel> _orders = new ObservableCollection<OrderItemModel>();
        private int? _selectedOrderId = null;

        public OrdersMenu()
        {
            InitializeComponent();
            BtnBack.Click += BtnBack_Click;
            LoadOrders();
        }

        private void LoadOrders()
        {
            _orders.Clear();
            MySqlConnection conn = Db.GetConnection();
            conn.Open();

            string sql = @"SELECT o.id_orders, o.kod, o.date_orders, o.date_delivery, 
                                  p.adress AS point_adress, u.name AS user_name, s.name AS status_name
                           FROM orders o
                           INNER JOIN point p ON o.point_id = p.id_point
                           LEFT JOIN users u ON o.users_id = u.id_user
                           INNER JOIN status s ON o.status_id = s.id_status
                           ORDER BY o.id_orders DESC";

            MySqlCommand cmd = new MySqlCommand(sql, conn);
            MySqlDataReader reader = cmd.ExecuteReader();

            while (reader.Read())
            {
                int id = reader.GetInt32("id_orders");
                string user_name = "Гость";
                if (!reader.IsDBNull(reader.GetOrdinal("user_name")))
                    user_name = reader.GetString("user_name");

                _orders.Add(new OrderItemModel
                {
                    Id = id,
                    Code = reader.GetInt32("kod"),
                    OrderDate = reader.GetString("date_orders"),
                    DeliveryDate = reader.GetString("date_delivery"),
                    PickupPoint = reader.GetString("point_adress"),
                    CreatedBy = user_name,
                    Status = reader.GetString("status_name")
                });
            }

            reader.Close();
            OrdersGrid.ItemsSource = _orders;

            if (_orders.Count > 0)
            {
                _selectedOrderId = _orders[0].Id;
                LoadOrderItems(_selectedOrderId.Value);
            }
        }

        private void LoadOrderItems(int orderId)
        {
            ObservableCollection<OrderItemModel> items = new ObservableCollection<OrderItemModel>();

            MySqlConnection conn = Db.GetConnection();
            conn.Open();

            string sql = @"SELECT oi.quanity, i.article, i.name AS item_name, i.price
                           FROM order_item oi
                           INNER JOIN items i ON oi.items_id = i.id_item
                           WHERE oi.orders_id = @order_id";

            MySqlCommand cmd = new MySqlCommand(sql, conn);
            cmd.Parameters.AddWithValue("@order_id", orderId);

            MySqlDataReader reader = cmd.ExecuteReader();
            while (reader.Read())
            {
                int quantity = 0;
                if (!reader.IsDBNull(reader.GetOrdinal("quanity")))
                    quantity = reader.GetInt32("quanity");

                string article = reader.GetString("article");
                string name = reader.GetString("item_name");
                decimal price = reader.GetDecimal("price");

                items.Add(new OrderItemModel
                {
                    Article = article,
                    Name = name,
                    Quantity = quantity,
                    PriceFormatted = price.ToString("F2") + " руб."
                });
            }

            reader.Close();
            OrderItemsGrid.ItemsSource = items;
        }

        private void OrdersGrid_SelectionChanged(object sender, System.Windows.Controls.SelectionChangedEventArgs e)
        {
            if (OrdersGrid.SelectedItem is OrderItemModel selectedOrder)
            {
                _selectedOrderId = selectedOrder.Id;
                LoadOrderItems(selectedOrder.Id);
            }
        }

        private void BtnBack_Click(object sender, RoutedEventArgs e)
        {
            Close();
        }
    }
}

```

edit.xaml


```cs
<Window x:Class="project.Edit"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
        xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
        mc:Ignorable="d"
        Title="Товар" Height="650" Width="750"
        WindowStartupLocation="CenterScreen">
    <Grid Margin="20">
        <ScrollViewer VerticalScrollBarVisibility="Auto">
            <StackPanel>
                <!-- Photo Section -->
                <TextBlock Text="Фото товара:" FontWeight="Bold" Margin="0,0,0,5"/>
                <Border BorderBrush="#CCCCCC" BorderThickness="1" Height="160" Background="#F5F5F5" 
                        CornerRadius="4" Margin="0,0,0,15" ClipToBounds="True">
                    <Image x:Name="PhotoPreview" Stretch="UniformToFill"/>
                </Border>
                <Button x:Name="BtnLoadPhoto" Content="Загрузить изображение" Width="180" Height="32" Margin="0,0,0,15"/>

                <!-- Article -->
                <DockPanel LastChildFill="True" Margin="0,0,0,10">
                    <TextBlock Text="Артикул:" Width="120" VerticalAlignment="Center"/>
                    <TextBox x:Name="TxtArticle" Height="32"/>
                </DockPanel>

                <!-- Name -->
                <DockPanel LastChildFill="True" Margin="0,0,0,10">
                    <TextBlock Text="Наименование:" Width="120" VerticalAlignment="Center"/>
                    <TextBox x:Name="TxtName" Height="32"/>
                </DockPanel>

                <!-- Category -->
                <DockPanel LastChildFill="True" Margin="0,0,0,10">
                    <TextBlock Text="Категория:" Width="120" VerticalAlignment="Center"/>
                    <ComboBox x:Name="CboCategory" Height="32"/>
                </DockPanel>

                <!-- Manufacturer -->
                <DockPanel LastChildFill="True" Margin="0,0,0,10">
                    <TextBlock Text="Производитель:" Width="120" VerticalAlignment="Center"/>
                    <ComboBox x:Name="CboManufacturs" Height="32"/>
                </DockPanel>

                <!-- Supplier -->
                <DockPanel LastChildFill="True" Margin="0,0,0,10">
                    <TextBlock Text="Поставщик:" Width="120" VerticalAlignment="Center"/>
                    <ComboBox x:Name="CboSuplier" Height="32"/>
                </DockPanel>

                <!-- Price -->
                <DockPanel LastChildFill="True" Margin="0,0,0,10">
                    <TextBlock Text="Цена (руб):" Width="120" VerticalAlignment="Center"/>
                    <TextBox x:Name="TxtPrice" Height="32"/>
                </DockPanel>

                <!-- Discount -->
                <DockPanel LastChildFill="True" Margin="0,0,0,10">
                    <TextBlock Text="Скидка (%):" Width="120" VerticalAlignment="Center"/>
                    <TextBox x:Name="TxtDiscount" Height="32"/>
                </DockPanel>

                <!-- Unit -->
                <DockPanel LastChildFill="True" Margin="0,0,0,10">
                    <TextBlock Text="Ед. измерения:" Width="120" VerticalAlignment="Center"/>
                    <TextBox x:Name="TxtUnit" Height="32"/>
                </DockPanel>

                <!-- Amount -->
                <DockPanel LastChildFill="True" Margin="0,0,0,10">
                    <TextBlock Text="Количество:" Width="120" VerticalAlignment="Center"/>
                    <TextBox x:Name="TxtAmount" Height="32"/>
                </DockPanel>

                <!-- Description -->
                <DockPanel LastChildFill="True" Margin="0,0,0,15">
                    <TextBlock Text="Описание:" Width="120" VerticalAlignment="Top"/>
                    <TextBox x:Name="TxtDesc" Height="80" AcceptsReturn="True" TextWrapping="Wrap" VerticalScrollBarVisibility="Auto"/>
                </DockPanel>

                <!-- Buttons -->
                <StackPanel Orientation="Horizontal" HorizontalAlignment="Right" Margin="0,15,0,0">
                    <Button x:Name="BtnBack" Content="Назад" Width="90" Height="36" Margin="0,0,8,0"/>
                    <Button x:Name="BtnDelete" Content="Удалить" Width="100" Height="36" 
                            Background="#E53935" Foreground="White" BorderThickness="0" Margin="0,0,8,0"/>
                    <Button x:Name="BtnSave" Content="Сохранить" Width="110" Height="36"
                            Background="#43A047" Foreground="White" BorderThickness="0"/>
                </StackPanel>
            </StackPanel>
        </ScrollViewer>
    </Grid>
</Window>
```


```cs
using System;
using System.IO;
using System.Windows;
using System.Windows.Controls;
using System.Windows.Media.Imaging;
using MySql.Data.MySqlClient;
using Microsoft.Win32;

namespace project
{
    public partial class Edit : Window
    {
        private readonly ProductItem _product;
        private int? _editingId = null;
        private string _currentPhotoName = "";
        private bool _isAdding = false;

        public Edit(ProductItem product)
        {
            InitializeComponent();

            if (product == null)
            {
                _isAdding = true;
                Title = "Добавить товар";
            }
            else
            {
                _editingId = product.id_item;
                _product = product;
                Title = "Редактирование: " + product.name;
            }

            BtnSave.Click += BtnSave_Click;
            BtnDelete.Click += BtnDelete_Click;
            BtnBack.Click += BtnBack_Click;
            BtnLoadPhoto.Click += BtnLoadPhoto_Click;

            LoadReferenceData();

            if (!_isAdding && _editingId.HasValue)
            {
                LoadProductData(_editingId.Value);
            }
        }

        private void LoadReferenceData()
        {
            MySqlConnection conn = Db.GetConnection();
            conn.Open();

            MySqlCommand cmdCat = new MySqlCommand("SELECT id_category, name FROM category ORDER BY name", conn);
            MySqlDataReader reader = cmdCat.ExecuteReader();
            while (reader.Read())
                CboCategory.Items.Add(new { Id = reader.GetInt32(0), Name = reader.GetString(1) });
            reader.Close();

            MySqlCommand cmdMan = new MySqlCommand("SELECT id_manufacturs, name FROM manufacturs ORDER BY name", conn);
            MySqlDataReader readerM = cmdMan.ExecuteReader();
            while (readerM.Read())
                CboManufacturs.Items.Add(new { Id = readerM.GetInt32(0), Name = readerM.GetString(1) });
            readerM.Close();

            MySqlCommand cmdSup = new MySqlCommand("SELECT id_suplier, name FROM suplier ORDER BY name", conn);
            MySqlDataReader readerS = cmdSup.ExecuteReader();
            while (readerS.Read())
                CboSuplier.Items.Add(new { Id = readerS.GetInt32(0), Name = readerS.GetString(1) });
            readerS.Close();
        }

        private void LoadProductData(int id)
        {
            MySqlConnection conn = Db.GetConnection();
            conn.Open();

            string sql = @"SELECT i.article, i.name, i.unit_name, i.price, i.discount, i.amount, 
                                  `i`.`desc`, `i`.`photo`,
                                  i.suplier_id, i.manufacturs_id, i.category_id
                           FROM items i WHERE i.id_item = @id";

            MySqlCommand cmd = new MySqlCommand(sql, conn);
            cmd.Parameters.AddWithValue("@id", id);

            MySqlDataReader reader = cmd.ExecuteReader();
            if (reader.Read())
            {
                TxtArticle.Text = reader.GetString("article");
                TxtName.Text = reader.GetString("name");
                TxtUnit.Text = reader.GetString("unit_name");
                TxtPrice.Text = reader.GetDecimal("price").ToString("F2");
                TxtDiscount.Text = reader.GetInt32("discount").ToString();
                TxtAmount.Text = reader.GetInt32("amount").ToString();
                TxtDesc.Text = reader.IsDBNull(reader.GetOrdinal("desc")) ? "" : reader.GetString("desc");

                _currentPhotoName = reader.GetString("photo");

                int suplierId = reader.GetInt32("suplier_id");
                int manufactursId = reader.GetInt32("manufacturs_id");
                int categoryId = reader.GetInt32("category_id");

                foreach (var item in CboSuplier.Items)
                {
                    dynamic obj = item;
                    if (obj.Id == suplierId) { CboSuplier.SelectedItem = item; break; }
                }
                foreach (var item in CboManufacturs.Items)
                {
                    dynamic obj = item;
                    if (obj.Id == manufactursId) { CboManufacturs.SelectedItem = item; break; }
                }
                foreach (var item in CboCategory.Items)
                {
                    dynamic obj = item;
                    if (obj.Id == categoryId) { CboCategory.SelectedItem = item; break; }
                }

                ShowPhoto(_currentPhotoName);
            }
            reader.Close();
        }

        private void ShowPhoto(string photoFileName)
        {
            BitmapImage bitmap = new BitmapImage();

            if (string.IsNullOrEmpty(photoFileName) || photoFileName == "null" || photoFileName == "\0")
            {
                bitmap.BeginInit();
                bitmap.UriSource = new Uri(@"import/picture.png", UriKind.Relative);
                bitmap.EndInit();
                PhotoPreview.Source = bitmap;
                return;
            }

            string fullPath = Path.Combine(AppDomain.CurrentDomain.BaseDirectory, @"..\..\import", photoFileName);
            if (!File.Exists(fullPath))
            {
                fullPath = Path.Combine(Environment.CurrentDirectory, "import", photoFileName);
            }

            if (!File.Exists(fullPath))
            {
                fullPath = Path.Combine(Path.GetDirectoryName(System.Reflection.Assembly.GetExecutingAssembly().Location), "import", photoFileName);
            }

            bitmap.BeginInit();
            if (File.Exists(fullPath))
            {
                bitmap.UriSource = new Uri(fullPath);
            }
            else
            {
                bitmap.UriSource = new Uri(@"import/picture.png", UriKind.Relative);
            }
            bitmap.EndInit();
            PhotoPreview.Source = bitmap;
        }

        private void BtnLoadPhoto_Click(object sender, RoutedEventArgs e)
        {
            OpenFileDialog dlg = new OpenFileDialog
            {
                Filter = "Image files|*.jpg;*.jpeg;*.png;*.bmp|All files|*.*",
                Title = "Выберите изображение"
            };

            if (dlg.ShowDialog() == true)
            {
                _currentPhotoName = Path.GetFileName(dlg.FileName);

                string baseDir = AppDomain.CurrentDomain.BaseDirectory;
                string importDir = Path.Combine(baseDir, "..", "..", "import");
                importDir = Path.GetFullPath(importDir);
                if (!Directory.Exists(importDir))
                    importDir = Path.Combine(Environment.CurrentDirectory, "import");

                string destPath = Path.Combine(importDir, _currentPhotoName);
                File.Copy(dlg.FileName, destPath, true);

                ShowPhoto(_currentPhotoName);
            }
        }

        private void BtnSave_Click(object sender, RoutedEventArgs e)
        {
            string article = TxtArticle.Text.Trim();
            string name = TxtName.Text.Trim();
            string unit_name = TxtUnit.Text.Trim();
            string priceStr = TxtPrice.Text.Trim();
            string discountStr = TxtDiscount.Text.Trim();
            string amountStr = TxtAmount.Text.Trim();
            string desc = TxtDesc.Text.Trim();

            if (string.IsNullOrEmpty(article) || string.IsNullOrEmpty(name))
            {
                MessageBox.Show("Заполните артикул и наименование", "Ошибка", MessageBoxButton.OK, MessageBoxImage.Warning);
                return;
            }

            decimal price;
            if (!decimal.TryParse(priceStr, out price))
            {
                MessageBox.Show("Введите корректную цену", "Ошибка", MessageBoxButton.OK, MessageBoxImage.Warning);
                return;
            }

            int discount = 0;
            if (!int.TryParse(discountStr, out discount))
                discount = 0;

            int amount = 0;
            if (!int.TryParse(amountStr, out amount))
                amount = 0;

            int suplierId = 1;
            if (CboSuplier.SelectedItem != null)
            {
                dynamic obj = CboSuplier.SelectedItem;
                suplierId = obj.Id;
            }

            int manufactursId = 1;
            if (CboManufacturs.SelectedItem != null)
            {
                dynamic obj = CboManufacturs.SelectedItem;
                manufactursId = obj.Id;
            }

            int categoryId = 1;
            if (CboCategory.SelectedItem != null)
            {
                dynamic obj = CboCategory.SelectedItem;
                categoryId = obj.Id;
            }

            string photoName = string.IsNullOrEmpty(_currentPhotoName) ? "picture.png" : _currentPhotoName;

            MySqlConnection conn = Db.GetConnection();
            conn.Open();

            try
            {
                if (_isAdding)
                {
                    string sql = @"INSERT INTO items (article, name, unit_name, price, discount, amount, `desc`, photo, 
                                         suplier_id, manufacturs_id, category_id)
                                   VALUES (@article, @name, @unit_name, @price, @discount, @amount, @desc, @photo, 
                                           @suplier_id, @manufacturs_id, @category_id)";

                    MySqlCommand cmd = new MySqlCommand(sql, conn);
                    cmd.Parameters.AddWithValue("@article", article);
                    cmd.Parameters.AddWithValue("@name", name);
                    cmd.Parameters.AddWithValue("@unit_name", unit_name);
                    cmd.Parameters.AddWithValue("@price", price);
                    cmd.Parameters.AddWithValue("@discount", discount);
                    cmd.Parameters.AddWithValue("@amount", amount);
                    cmd.Parameters.AddWithValue("@desc", desc);
                    cmd.Parameters.AddWithValue("@photo", photoName);
                    cmd.Parameters.AddWithValue("@suplier_id", suplierId);
                    cmd.Parameters.AddWithValue("@manufacturs_id", manufactursId);
                    cmd.Parameters.AddWithValue("@category_id", categoryId);

                    cmd.ExecuteNonQuery();
                }
                else
                {
                    string sql = @"UPDATE items SET article=@article, name=@name, unit_name=@unit_name, price=@price, 
                                         discount=@discount, amount=@amount, `desc`=@desc, photo=@photo,
                                         suplier_id=@suplier_id, manufacturs_id=@manufacturs_id, category_id=@category_id
                                   WHERE id_item=@id";

                    MySqlCommand cmd = new MySqlCommand(sql, conn);
                    cmd.Parameters.AddWithValue("@article", article);
                    cmd.Parameters.AddWithValue("@name", name);
                    cmd.Parameters.AddWithValue("@unit_name", unit_name);
                    cmd.Parameters.AddWithValue("@price", price);
                    cmd.Parameters.AddWithValue("@discount", discount);
                    cmd.Parameters.AddWithValue("@amount", amount);
                    cmd.Parameters.AddWithValue("@desc", desc);
                    cmd.Parameters.AddWithValue("@photo", photoName);
                    cmd.Parameters.AddWithValue("@suplier_id", suplierId);
                    cmd.Parameters.AddWithValue("@manufacturs_id", manufactursId);
                    cmd.Parameters.AddWithValue("@category_id", categoryId);
                    cmd.Parameters.AddWithValue("@id", _editingId.Value);

                    cmd.ExecuteNonQuery();
                }

                MessageBox.Show("Сохранено успешно!", "Готово", MessageBoxButton.OK, MessageBoxImage.Information);
                DialogResult = true;
                Close();
            }
            catch (Exception ex)
            {
                MessageBox.Show("Ошибка сохранения: " + ex.Message, "Ошибка", MessageBoxButton.OK, MessageBoxImage.Error);
            }
        }

        private void BtnDelete_Click(object sender, RoutedEventArgs e)
        {
            if (!_editingId.HasValue)
            {
                MessageBox.Show("Нечего удалять. Это новый товар.", "Внимание", MessageBoxButton.OK, MessageBoxImage.Information);
                return;
            }

            bool result = MessageBox.Show("Удалить товар '" + TxtName.Text + "'?", "Подтверждение",
                                          MessageBoxButton.YesNo, MessageBoxImage.Question) == MessageBoxResult.Yes;

            if (!result) return;

            MySqlConnection conn = Db.GetConnection();
            conn.Open();

            try
            {
                string sql = "DELETE FROM items WHERE id_item = @id";
                MySqlCommand cmd = new MySqlCommand(sql, conn);
                cmd.Parameters.AddWithValue("@id", _editingId.Value);
                cmd.ExecuteNonQuery();

                MessageBox.Show("Товар удалён.", "Готово", MessageBoxButton.OK, MessageBoxImage.Information);
                DialogResult = true;
                Close();
            }
            catch (Exception ex)
            {
                MessageBox.Show("Ошибка удаления: " + ex.Message, "Ошибка", MessageBoxButton.OK, MessageBoxImage.Error);
            }
        }

        private void BtnBack_Click(object sender, RoutedEventArgs e)
        {
            DialogResult = false;
            Close();
        }
    }
}
```
