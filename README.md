# Step by Step Guide to CRUD Operations using Web API and Dapper mini ORM
ตัวอย่างโค้ด CRUD สำหรับ Product โดยใช้ **C# WebAPI**, **Dapper**, **SQL Server (local)**, และแบ่ง Layer ตาม **Clean Architecture** และ **SOLID** Design Principles

## โครงสร้างโฟลเดอร์
```
ProductApi/
├── Controllers/
│   └── ProductController.cs
├── Services/
│   └── IProductService.cs
│   └── ProductService.cs
├── Repositories/
│   └── IProductRepository.cs
│   └── ProductRepository.cs
├── Models/
│   └── Product.cs
├── Data/
│   └── DapperContext.cs
└── Program.cs
└── appsettings.json
```

---

```csharp name=Models/Product.cs
namespace ProductApi.Models
{
    public class Product
    {
        public int Id { get; set; }
        public string Name { get; set; } = string.Empty;
        public decimal Price { get; set; }
        public int Stock { get; set; }
    }
}
```

---

```csharp name=Data/DapperContext.cs
using System.Data;
using Microsoft.Data.SqlClient;

namespace ProductApi.Data
{
    public class DapperContext
    {
        private readonly IConfiguration _config;
        private readonly string _connectionString;

        public DapperContext(IConfiguration config)
        {
            _config = config;
            _connectionString = _config.GetConnectionString("DefaultConnection");
        }

        public IDbConnection CreateConnection()
            => new SqlConnection(_connectionString);
    }
}
```

---

```csharp name=Repositories/IProductRepository.cs
using ProductApi.Models;

namespace ProductApi.Repositories
{
    public interface IProductRepository
    {
        Task<IEnumerable<Product>> GetAllAsync();
        Task<Product?> GetByIdAsync(int id);
        Task<int> AddAsync(Product product);
        Task<bool> UpdateAsync(Product product);
        Task<bool> DeleteAsync(int id);
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
        private readonly DapperContext _context;

        public ProductRepository(DapperContext context)
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

```csharp name=Services/IProductService.cs
using ProductApi.Models;

namespace ProductApi.Services
{
    public interface IProductService
    {
        Task<IEnumerable<Product>> GetAllAsync();
        Task<Product?> GetByIdAsync(int id);
        Task<int> AddAsync(Product product);
        Task<bool> UpdateAsync(Product product);
        Task<bool> DeleteAsync(int id);
    }
}
```

---

```csharp name=Services/ProductService.cs
using ProductApi.Models;
using ProductApi.Repositories;

namespace ProductApi.Services
{
    public class ProductService : IProductService
    {
        private readonly IProductRepository _repo;

        public ProductService(IProductRepository repo)
        {
            _repo = repo;
        }

        public Task<IEnumerable<Product>> GetAllAsync()
            => _repo.GetAllAsync();

        public Task<Product?> GetByIdAsync(int id)
            => _repo.GetByIdAsync(id);

        public Task<int> AddAsync(Product product)
            => _repo.AddAsync(product);

        public Task<bool> UpdateAsync(Product product)
            => _repo.UpdateAsync(product);

        public Task<bool> DeleteAsync(int id)
            => _repo.DeleteAsync(id);
    }
}
```

---

```csharp name=Controllers/ProductController.cs
using Microsoft.AspNetCore.Mvc;
using ProductApi.Models;
using ProductApi.Services;

namespace ProductApi.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public class ProductController : ControllerBase
    {
        private readonly IProductService _service;

        public ProductController(IProductService service)
        {
            _service = service;
        }

        [HttpGet]
        public async Task<IActionResult> GetAll()
        {
            var products = await _service.GetAllAsync();
            return Ok(products);
        }

        [HttpGet("{id}")]
        public async Task<IActionResult> GetById(int id)
        {
            var product = await _service.GetByIdAsync(id);
            if (product == null) return NotFound();
            return Ok(product);
        }

        [HttpPost]
        public async Task<IActionResult> Create(Product product)
        {
            var id = await _service.AddAsync(product);
            return CreatedAtAction(nameof(GetById), new { id }, product);
        }

        [HttpPut("{id}")]
        public async Task<IActionResult> Update(int id, Product product)
        {
            if (id != product.Id) return BadRequest();
            var result = await _service.UpdateAsync(product);
            if (!result) return NotFound();
            return NoContent();
        }

        [HttpDelete("{id}")]
        public async Task<IActionResult> Delete(int id)
        {
            var result = await _service.DeleteAsync(id);
            if (!result) return NotFound();
            return NoContent();
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
builder.Services.AddSingleton<DapperContext>();
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

```json name=appsettings.json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=ProductDb;Trusted_Connection=True;"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning"
    }
  },
  "AllowedHosts": "*"
}
```

---

## โค้ดสำหรับสร้าง Table MSSQL

```sql
CREATE TABLE Products
(
    Id INT PRIMARY KEY IDENTITY,
    Name NVARCHAR(100) NOT NULL,
    Price DECIMAL(18,2) NOT NULL,
    Stock INT NOT NULL
)
```

---

## สรุป

- แต่ละ Layer แยกบทบาทชัดเจน: Controller รับ request, Service ทำ logic, Repository จัดการ DB
- ใช้ Dapper สำหรับ SQL
- DI ตาม Clean Architecture
- SOLID: Interface Segregation, Dependency Inversion, Single Responsibility
- รองรับ Swagger ใน WebAPI
#
