# DapperContext refcator ให้ unitTest ง่าย
Refactor `DapperContext` ให้ Unit Test ง่ายขึ้น โดยใช้ Interface และ Dependency Injection

```csharp name=Data/IDapperContext.cs
using System.Data;

namespace ProductApi.Data
{
    public interface IDapperContext
    {
        IDbConnection CreateConnection();
    }
}
```

---

```csharp name=Data/DapperContext.cs
using System.Data;
using Microsoft.Data.SqlClient;

namespace ProductApi.Data
{
    public class DapperContext : IDapperContext
    {
        private readonly string _connectionString;

        public DapperContext(IConfiguration config)
        {
            _connectionString = config.GetConnectionString("DefaultConnection") 
                ?? throw new ArgumentNullException(nameof(config), "Connection string cannot be null");
        }

        // Constructor สำหรับ Unit Test
        public DapperContext(string connectionString)
        {
            _connectionString = connectionString ?? throw new ArgumentNullException(nameof(connectionString));
        }

        public IDbConnection CreateConnection()
            => new SqlConnection(_connectionString);
    }
}
```

---

```csharp name=Repositories/ProductRepository.cs
using Dapper;
using ProductApi.Data;
using ProductApi.Models;

namespace ProductApi.Repositories
{
    public class ProductRepository : IProductRepository
    {
        private readonly IDapperContext _context;

        public ProductRepository(IDapperContext context)
        {
            _context = context;
        }

        public async Task<IEnumerable<Product>> GetAllAsync()
        {
            var query = "SELECT * FROM Products";
            using var connection = _context.CreateConnection();
            return await connection.QueryAsync<Product>(query);
        }

        public async Task<Product?> GetByIdAsync(int id)
        {
            var query = "SELECT * FROM Products WHERE Id = @Id";
            using var connection = _context.CreateConnection();
            return await connection.QuerySingleOrDefaultAsync<Product>(query, new { Id = id });
        }

        public async Task<int> AddAsync(Product product)
        {
            var query = "INSERT INTO Products (Name, Price, Stock) VALUES (@Name, @Price, @Stock); SELECT CAST(SCOPE_IDENTITY() as int);";
            using var connection = _context.CreateConnection();
            return await connection.QuerySingleAsync<int>(query, product);
        }

        public async Task<bool> UpdateAsync(Product product)
        {
            var query = "UPDATE Products SET Name = @Name, Price = @Price, Stock = @Stock WHERE Id = @Id";
            using var connection = _context.CreateConnection();
            var affectedRows = await connection.ExecuteAsync(query, product);
            return affectedRows > 0;
        }

        public async Task<bool> DeleteAsync(int id)
        {
            var query = "DELETE FROM Products WHERE Id = @Id";
            using var connection = _context.CreateConnection();
            var affectedRows = await connection.ExecuteAsync(query, new { Id = id });
            return affectedRows > 0;
        }
    }
}
```

---

```csharp name=Program.cs
using ProductApi.Data;
using ProductApi.Repositories;
using ProductApi.Services;

var builder = WebApplication.CreateBuilder(args);

// Add services to DI
builder.Services.AddSingleton<IDapperContext, DapperContext>();
builder.Services.AddScoped<IProductRepository, ProductRepository>();
builder.Services.AddScoped<IProductService, ProductService>();
builder.Services.AddControllers();

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

app.UseSwagger();
app.UseSwaggerUI();

app.UseAuthorization();
app.MapControllers();

app.Run();
```

---

## ตัวอย่าง Unit Test

```csharp name=Tests/ProductRepositoryTests.cs
using System.Data;
using Microsoft.Data.Sqlite;
using ProductApi.Data;
using ProductApi.Models;
using ProductApi.Repositories;
using Moq;
using Xunit;
using Dapper;

namespace ProductApi.Tests
{
    public class ProductRepositoryTests : IDisposable
    {
        private readonly IDbConnection _connection;
        private readonly IDapperContext _context;
        private readonly ProductRepository _repository;

        public ProductRepositoryTests()
        {
            // ใช้ SQLite In-Memory สำหรับ Unit Test
            _connection = new SqliteConnection("DataSource=:memory:");
            _connection.Open();
            
            // สร้าง Mock Context
            var mockContext = new Mock<IDapperContext>();
            mockContext.Setup(x => x.CreateConnection()).Returns(_connection);
            _context = mockContext.Object;
            
            _repository = new ProductRepository(_context);
            
            // สร้าง Table สำหรับ Test
            CreateTestTable();
        }

        private void CreateTestTable()
        {
            var createTableSql = @"
                CREATE TABLE Products (
                    Id INTEGER PRIMARY KEY AUTOINCREMENT,
                    Name TEXT NOT NULL,
                    Price DECIMAL(18,2) NOT NULL,
                    Stock INTEGER NOT NULL
                )";
            _connection.Execute(createTableSql);
        }

        [Fact]
        public async Task GetAllAsync_ShouldReturnAllProducts()
        {
            // Arrange
            var product1 = new Product { Name = "Product1", Price = 100.00m, Stock = 10 };
            var product2 = new Product { Name = "Product2", Price = 200.00m, Stock = 20 };
            
            await _repository.AddAsync(product1);
            await _repository.AddAsync(product2);

            // Act
            var result = await _repository.GetAllAsync();

            // Assert
            Assert.Equal(2, result.Count());
        }

        [Fact]
        public async Task GetByIdAsync_WithValidId_ShouldReturnProduct()
        {
            // Arrange
            var product = new Product { Name = "Test Product", Price = 100.00m, Stock = 10 };
            var id = await _repository.AddAsync(product);

            // Act
            var result = await _repository.GetByIdAsync(id);

            // Assert
            Assert.NotNull(result);
            Assert.Equal("Test Product", result.Name);
            Assert.Equal(100.00m, result.Price);
        }

        [Fact]
        public async Task AddAsync_ShouldReturnNewId()
        {
            // Arrange
            var product = new Product { Name = "New Product", Price = 150.00m, Stock = 5 };

            // Act
            var id = await _repository.AddAsync(product);

            // Assert
            Assert.True(id > 0);
        }

        [Fact]
        public async Task UpdateAsync_WithValidProduct_ShouldReturnTrue()
        {
            // Arrange
            var product = new Product { Name = "Original", Price = 100.00m, Stock = 10 };
            var id = await _repository.AddAsync(product);
            
            product.Id = id;
            product.Name = "Updated";
            product.Price = 200.00m;

            // Act
            var result = await _repository.UpdateAsync(product);

            // Assert
            Assert.True(result);
            
            var updatedProduct = await _repository.GetByIdAsync(id);
            Assert.Equal("Updated", updatedProduct?.Name);
            Assert.Equal(200.00m, updatedProduct?.Price);
        }

        [Fact]
        public async Task DeleteAsync_WithValidId_ShouldReturnTrue()
        {
            // Arrange
            var product = new Product { Name = "To Delete", Price = 100.00m, Stock = 10 };
            var id = await _repository.AddAsync(product);

            // Act
            var result = await _repository.DeleteAsync(id);

            // Assert
            Assert.True(result);
            
            var deletedProduct = await _repository.GetByIdAsync(id);
            Assert.Null(deletedProduct);
        }

        public void Dispose()
        {
            _connection?.Dispose();
        }
    }
}
```

---

## Mock Context สำหรับ Unit Test อื่นๆ

```csharp name=Tests/MockDapperContext.cs
using System.Data;
using Microsoft.Data.Sqlite;
using ProductApi.Data;

namespace ProductApi.Tests
{
    public class MockDapperContext : IDapperContext, IDisposable
    {
        private readonly IDbConnection _connection;

        public MockDapperContext()
        {
            _connection = new SqliteConnection("DataSource=:memory:");
            _connection.Open();
        }

        public IDbConnection CreateConnection()
            => _connection;

        public void Dispose()
        {
            _connection?.Dispose();
        }
    }
}
```

---

## การเปลี่ยนแปลงหลัก

1. **เพิ่ม Interface `IDapperContext`** - ทำให้สามารถ Mock ได้ง่าย
2. **Constructor Overload** - รับ connection string โดยตรงสำหรับ Test
3. **Dependency Injection** - ใช้ Interface แทน concrete class
4. **Unit Tests** - ใช้ SQLite In-Memory สำหรับ Test
5. **Mock Implementation** - สำหรับ Test scenarios ที่ซับซ้อน

## ประโยชน์

- **Testable**: สามารถ Mock database connection ได้
- **Flexible**: เปลี่ยน database provider ได้ง่าย
- **Isolated**: Unit test ไม่ต้องพึ่งพา actual database
- **Fast**: ใช้ In-Memory database ทำให้ test รวดเร็ว
#
