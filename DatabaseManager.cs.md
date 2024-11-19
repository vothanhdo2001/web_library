
# Hướng Dẫn Sử Dụng `DatabaseManager`

## Giới Thiệu
Lớp `DatabaseManager` được thiết kế để:
1. Thực hiện các truy vấn cơ sở dữ liệu (query) với kết quả ánh xạ mạnh mẽ (strongly-typed).
2. Xử lý các thao tác bulk insert với hiệu suất cao.
3. Hỗ trợ retry logic trong trường hợp truy vấn gặp lỗi.

---

### Thư viện cần thiết:
- **Dapper**: Thư viện micro ORM để ánh xạ kết quả truy vấn cơ sở dữ liệu thành các đối tượng mạnh mẽ.

#### Cài đặt Dapper:
Sử dụng **NuGet Package Manager Console**:
```bash
Install-Package Dapper
```

Hoặc sử dụng **.NET CLI**:
```bash
dotnet add package Dapper
```

---

## Sử Dụng
### 1. Khởi Tạo `DatabaseManager`
```csharp
var databaseManager = new DatabaseManager(
    connectionString: "your_connection_string",
    timeout: 60,              // Timeout 60 giây
    maxRetries: 5,            // Tối đa 5 lần retry
    retryDelayMilliseconds: 30000 // Delay 30 giây giữa các lần retry
);
```

### 2. Truy Vấn Dữ Liệu (`ExecuteQueryAsync` với Dapper)
Phương thức `ExecuteQueryAsync<T>` cho phép bạn thực hiện truy vấn SQL và ánh xạ kết quả thành danh sách các đối tượng mạnh (`IEnumerable<T>`).

#### Ví dụ:
```csharp
public class User
{
    public int Id { get; set; }
    public string Name { get; set; }
}

var users = await databaseManager.ExecuteQueryAsync<User>("SELECT Id, Name FROM Users");
```

### 3. Truy Vấn Dữ Liệu Tùy Biến (`ExecuteReaderAsync` với SqlDataReader)
Phương thức `ExecuteReaderAsync<T>` cho phép bạn ánh xạ thủ công kết quả từ `SqlDataReader`.

#### Ví dụ:
```csharp
var users = await databaseManager.ExecuteReaderAsync<User>("SELECT Id, Name FROM Users", reader =>
{
    return new User
    {
        Id = reader.GetInt32(reader.GetOrdinal("Id")),
        Name = reader.GetString(reader.GetOrdinal("Name"))
    };
});
```

### 4. Chèn Dữ Liệu Hàng Loạt (`BulkInsertAsync`)
Phương thức `BulkInsertAsync` sử dụng **SqlBulkCopy** để chèn một danh sách lớn các bản ghi vào cơ sở dữ liệu.

#### Ví dụ:
```csharp
var users = new List<User>
{
    new User { Id = 1, Name = "Alice" },
    new User { Id = 2, Name = "Bob" }
};

await databaseManager.BulkInsertAsync("Users", users);
```

---

## Mã Nguồn Hoàn Chỉnh
### Lớp `DatabaseManager`:
```csharp
using System;
using System.Collections.Generic;
using System.Data;
using System.Data.SqlClient;
using System.Threading;
using System.Threading.Tasks;
using Dapper;
using SynchronizeSubscribersWithAttentive.Logger;

namespace SynchronizeSubscribersWithAttentive.DataAccess
{
    public class DatabaseManager
    {
        private readonly string _connectionString;
        private readonly int _timeout;
        private readonly int _maxRetries;
        private readonly int _retryDelayMilliseconds;

        public DatabaseManager(string connectionString, int timeout = 30, int maxRetries = 5, int retryDelayMilliseconds = 30000)
        {
            _connectionString = connectionString ?? throw new ArgumentNullException(nameof(connectionString));
            _timeout = timeout;
            _maxRetries = maxRetries;
            _retryDelayMilliseconds = retryDelayMilliseconds;
        }

        // 1. Phương thức ExecuteQueryAsync với Dapper
        public async Task<IEnumerable<T>> ExecuteQueryAsync<T>(string query, object parameters = null, CancellationToken cancellationToken = default)
        {
            return await RetryAsync(async () =>
            {
                using (var connection = new SqlConnection(_connectionString))
                {
                    Log.Info($"Executing query: {query}");
                    await connection.OpenAsync(cancellationToken);
                    return await connection.QueryAsync<T>(query, parameters, commandTimeout: _timeout);
                }
            }, query, cancellationToken);
        }

        // 2. Phương thức ExecuteReaderAsync với SqlDataReader
        public async Task<List<T>> ExecuteReaderAsync<T>(string query, Func<SqlDataReader, T> mapFunction, CancellationToken cancellationToken = default)
        {
            return await RetryAsync(async () =>
            {
                var results = new List<T>();

                using (var connection = new SqlConnection(_connectionString))
                using (var command = new SqlCommand(query, connection))
                {
                    command.CommandTimeout = _timeout;

                    Log.Info($"Executing query with SqlDataReader: {query}");
                    await connection.OpenAsync(cancellationToken);

                    using (var reader = await command.ExecuteReaderAsync(cancellationToken))
                    {
                        while (await reader.ReadAsync(cancellationToken))
                        {
                            results.Add(mapFunction(reader));
                        }
                    }
                }

                return results;
            }, query, cancellationToken);
        }

        // 3. Phương thức BulkInsertAsync
        public async Task BulkInsertAsync<T>(string tableName, IEnumerable<T> data)
        {
            if (data == null) throw new ArgumentNullException(nameof(data));
            if (string.IsNullOrWhiteSpace(tableName)) throw new ArgumentNullException(nameof(tableName));

            using (var connection = new SqlConnection(_connectionString))
            using (var bulkCopy = new SqlBulkCopy(connection)
            {
                DestinationTableName = tableName,
                BatchSize = 1000, // Số lượng bản ghi mỗi batch
                BulkCopyTimeout = _timeout
            })
            {
                var dataTable = ToDataTable(data);

                Log.Info($"Bulk inserting {dataTable.Rows.Count} rows into {tableName}");
                await connection.OpenAsync();
                await bulkCopy.WriteToServerAsync(dataTable);
            }
        }

        // Helper: Chuyển đổi IEnumerable<T> thành DataTable
        private DataTable ToDataTable<T>(IEnumerable<T> data)
        {
            var dataTable = new DataTable(typeof(T).Name);
            var properties = typeof(T).GetProperties();

            foreach (var prop in properties)
            {
                dataTable.Columns.Add(prop.Name, Nullable.GetUnderlyingType(prop.PropertyType) ?? prop.PropertyType);
            }

            foreach (var item in data)
            {
                var row = dataTable.NewRow();
                foreach (var prop in properties)
                {
                    row[prop.Name] = prop.GetValue(item) ?? DBNull.Value;
                }
                dataTable.Rows.Add(row);
            }

            return dataTable;
        }

        // Retry Logic
        private async Task<T> RetryAsync<T>(Func<Task<T>> operation, string operationDescription, CancellationToken cancellationToken)
        {
            for (int attempt = 1; attempt <= _maxRetries; attempt++)
            {
                try
                {
                    return await operation();
                }
                catch (Exception ex)
                {
                    Log.Error($"Error during operation '{operationDescription}': {ex.Message}");
                    if (attempt >= _maxRetries)
                    {
                        throw;
                    }

                    Log.Info($"Retrying operation '{operationDescription}' in {_retryDelayMilliseconds / 1000} seconds... (Attempt {attempt})");
                    await Task.Delay(_retryDelayMilliseconds, cancellationToken);
                }
            }
            throw new InvalidOperationException("Retry logic failed unexpectedly.");
        }
    }
}
```

---

## Ghi Chú
- **Đảm bảo tên các cột trong bảng SQL Server khớp với tên các thuộc tính trong lớp C# (`T`).**
- **Nếu cần tùy chỉnh thêm các logic xử lý đặc thù, bạn có thể mở rộng lớp `DatabaseManager`.**

---

## Liên Hệ
Nếu gặp vấn đề, bạn có thể liên hệ qua [GitHub Issues](https://github.com).
