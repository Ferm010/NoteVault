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



```cs

```
