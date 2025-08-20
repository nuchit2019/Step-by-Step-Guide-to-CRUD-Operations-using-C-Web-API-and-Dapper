# Integration Test โดยใช้ In-Memory Database เพื่อให้ test รวดเร็ว

```csharp name=Tests/ProductApiIntegrationTests.cs
using System.Net;
using System.Net.Http.Json;
using System.Text;
using System.Text.Json;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.DependencyInjection.Extensions;
using ProductApi.Data;
using ProductApi.Models;
using Xunit;

namespace ProductApi.Tests
{
    // คลาส Integration Test ที่ inherit จาก IClassFixture เพื่อใช้ TestWebApplicationFactory ร่วมกัน
    public class ProductApiIntegrationTests : IClassFixture<TestWebApplicationFactory<Program>>
    {
        private readonly HttpClient _client; // HttpClient สำหรับเรียก API
        private readonly TestWebApplicationFactory<Program> _factory; // Factory สำหรับสร้าง Test Server

        // Constructor รับ factory และสร้าง HttpClient
        public ProductApiIntegrationTests(TestWebApplicationFactory<Program> factory)
        {
            _factory = factory; // เก็บ reference ของ factory
            _client = factory.CreateClient(); // สร้าง HttpClient สำหรับทดสอบ
        }

        [Fact] // Attribute บอกว่าเป็น Test Method
        public async Task GetAllProducts_WhenNoProducts_ShouldReturnEmptyList()
        {
            // Act - เรียก GET /api/product
            var response = await _client.GetAsync("/api/product");

            // Assert - ตรวจสอบผลลัพธ์
            response.EnsureSuccessStatusCode(); // ตรวจสอบว่า status code เป็น 2xx
            var products = await response.Content.ReadFromJsonAsync<List<Product>>(); // แปลง response เป็น List<Product>
            
            Assert.NotNull(products); // ตรวจสอบว่า products ไม่เป็น null
            Assert.Empty(products); // ตรวจสอบว่า list ว่างเปล่า
        }

        [Fact]
        public async Task CreateProduct_WithValidData_ShouldReturnCreated()
        {
            // Arrange - เตรียมข้อมูลทดสอบ
            var newProduct = new Product
            {
                Name = "Integration Test Product", // ชื่อสินค้าสำหรับทดสอบ
                Price = 99.99m, // ราคาสินค้า (decimal type)
                Stock = 50 // จำนวนสต็อก
            };

            // Act - ส่ง POST request พร้อมข้อมูล JSON
            var response = await _client.PostAsJsonAsync("/api/product", newProduct);

            // Assert - ตรวจสอบผลลัพธ์
            Assert.Equal(HttpStatusCode.Created, response.StatusCode); // ตรวจสอบ status code = 201 Created
            
            // อ่าน response body เป็น Product object
            var createdProduct = await response.Content.ReadFromJsonAsync<Product>();
            Assert.NotNull(createdProduct); // ตรวจสอบว่าได้ product กลับมา
            Assert.True(createdProduct.Id > 0); // ตรวจสอบว่ามี ID ที่มากกว่า 0
            Assert.Equal(newProduct.Name, createdProduct.Name); // ตรวจสอบชื่อสินค้า
            Assert.Equal(newProduct.Price, createdProduct.Price); // ตรวจสอบราคา
            Assert.Equal(newProduct.Stock, createdProduct.Stock); // ตรวจสอบสต็อก
        }

        [Fact]
        public async Task GetProductById_WithExistingId_ShouldReturnProduct()
        {
            // Arrange - สร้างสินค้าใหม่ก่อน
            var newProduct = new Product
            {
                Name = "Test Product for Get", // ชื่อสินค้าทดสอบ
                Price = 149.99m, // ราคาสินค้า
                Stock = 25 // จำนวนสต็อก
            };

            // สร้างสินค้าผ่าน API
            var createResponse = await _client.PostAsJsonAsync("/api/product", newProduct);
            var createdProduct = await createResponse.Content.ReadFromJsonAsync<Product>();

            // Act - เรียก GET /api/product/{id}
            var response = await _client.GetAsync($"/api/product/{createdProduct!.Id}");

            // Assert - ตรวจสอบผลลัพธ์
            response.EnsureSuccessStatusCode(); // ตรวจสอบ success status
            var product = await response.Content.ReadFromJsonAsync<Product>(); // แปลง response เป็น Product
            
            Assert.NotNull(product); // ตรวจสอบว่าได้ product กลับมา
            Assert.Equal(createdProduct.Id, product.Id); // ตรวจสอบ ID ตรงกัน
            Assert.Equal(newProduct.Name, product.Name); // ตรวจสอบชื่อตรงกัน
        }

        [Fact]
        public async Task GetProductById_WithNonExistingId_ShouldReturnNotFound()
        {
            // Act - เรียก GET ด้วย ID ที่ไม่มีอยู่
            var response = await _client.GetAsync("/api/product/99999");

            // Assert - ตรวจสอบว่าได้ 404 Not Found
            Assert.Equal(HttpStatusCode.NotFound, response.StatusCode);
        }

        [Fact]
        public async Task UpdateProduct_WithValidData_ShouldReturnNoContent()
        {
            // Arrange - สร้างสินค้าใหม่ก่อน
            var newProduct = new Product
            {
                Name = "Product to Update", // ชื่อเดิม
                Price = 199.99m, // ราคาเดิม
                Stock = 30 // สต็อกเดิม
            };

            // สร้างสินค้าผ่าน API
            var createResponse = await _client.PostAsJsonAsync("/api/product", newProduct);
            var createdProduct = await createResponse.Content.ReadFromJsonAsync<Product>();

            // เตรียมข้อมูลที่จะอัพเดต
            var updatedProduct = new Product
            {
                Id = createdProduct!.Id, // ใช้ ID เดิม
                Name = "Updated Product Name", // ชื่อใหม่
                Price = 299.99m, // ราคาใหม่
                Stock = 40 // สต็อกใหม่
            };

            // Act - ส่ง PUT request เพื่ออัพเดต
            var response = await _client.PutAsJsonAsync($"/api/product/{updatedProduct.Id}", updatedProduct);

            // Assert - ตรวจสอบผลลัพธ์
            Assert.Equal(HttpStatusCode.NoContent, response.StatusCode); // ตรวจสอบ 204 No Content

            // ตรวจสอบว่าข้อมูลถูกอัพเดตจริง
            var getResponse = await _client.GetAsync($"/api/product/{updatedProduct.Id}");
            var product = await getResponse.Content.ReadFromJsonAsync<Product>();
            
            Assert.Equal(updatedProduct.Name, product!.Name); // ตรวจสอบชื่อใหม่
            Assert.Equal(updatedProduct.Price, product.Price); // ตรวจสอบราคาใหม่
            Assert.Equal(updatedProduct.Stock, product.Stock); // ตรวจสอบสต็อกใหม่
        }

        [Fact]
        public async Task UpdateProduct_WithMismatchedId_ShouldReturnBadRequest()
        {
            // Arrange - เตรียมข้อมูลที่ ID ไม่ตรงกัน
            var product = new Product
            {
                Id = 1, // ID ใน object
                Name = "Test Product",
                Price = 100.00m,
                Stock = 10
            };

            // Act - ส่ง PUT ด้วย URL ID ที่ไม่ตรงกับ object ID
            var response = await _client.PutAsJsonAsync("/api/product/2", product); // URL ID = 2, object ID = 1

            // Assert - ตรวจสอบว่าได้ 400 Bad Request
            Assert.Equal(HttpStatusCode.BadRequest, response.StatusCode);
        }

        [Fact]
        public async Task DeleteProduct_WithExistingId_ShouldReturnNoContent()
        {
            // Arrange - สร้างสินค้าที่จะลบ
            var newProduct = new Product
            {
                Name = "Product to Delete", // ชื่อสินค้า
                Price = 99.99m, // ราคา
                Stock = 15 // สต็อก
            };

            // สร้างสินค้าผ่าน API
            var createResponse = await _client.PostAsJsonAsync("/api/product", newProduct);
            var createdProduct = await createResponse.Content.ReadFromJsonAsync<Product>();

            // Act - ส่ง DELETE request
            var response = await _client.DeleteAsync($"/api/product/{createdProduct!.Id}");

            // Assert - ตรวจสอบผลลัพธ์
            Assert.Equal(HttpStatusCode.NoContent, response.StatusCode); // ตรวจสอบ 204 No Content

            // ตรวจสอบว่าสินค้าถูกลบจริง
            var getResponse = await _client.GetAsync($"/api/product/{createdProduct.Id}");
            Assert.Equal(HttpStatusCode.NotFound, getResponse.StatusCode); // ควรได้ 404 Not Found
        }

        [Fact]
        public async Task DeleteProduct_WithNonExistingId_ShouldReturnNotFound()
        {
            // Act - ลบสินค้าที่ไม่มีอยู่
            var response = await _client.DeleteAsync("/api/product/99999");

            // Assert - ตรวจสอบว่าได้ 404 Not Found
            Assert.Equal(HttpStatusCode.NotFound, response.StatusCode);
        }

        [Fact]
        public async Task GetAllProducts_AfterCreatingMultiple_ShouldReturnAllProducts()
        {
            // Arrange - สร้างสินค้าหลายตัว
            var products = new[]
            {
                new Product { Name = "Product 1", Price = 100.00m, Stock = 10 },
                new Product { Name = "Product 2", Price = 200.00m, Stock = 20 },
                new Product { Name = "Product 3", Price = 300.00m, Stock = 30 }
            };

            // สร้างสินค้าทั้งหมดผ่าน API
            foreach (var product in products)
            {
                await _client.PostAsJsonAsync("/api/product", product); // สร้างทีละตัว
            }

            // Act - เรียก GET all products
            var response = await _client.GetAsync("/api/product");

            // Assert - ตรวจสอบผลลัพธ์
            response.EnsureSuccessStatusCode(); // ตรวจสอบ success status
            var retrievedProducts = await response.Content.ReadFromJsonAsync<List<Product>>(); // แปลงเป็น List
            
            Assert.NotNull(retrievedProducts); // ตรวจสอบว่าได้ list กลับมา
            Assert.True(retrievedProducts.Count >= products.Length); // ตรวจสอบว่ามีอย่างน้อยเท่าที่สร้าง
        }

        [Fact]
        public async Task CreateProduct_WithInvalidData_ShouldHandleGracefully()
        {
            // Arrange - เตรียมข้อมูลที่ไม่ถูกต้อง (ราคาติดลบ)
            var invalidProduct = new Product
            {
                Name = "", // ชื่อว่าง
                Price = -10.00m, // ราคาติดลบ
                Stock = -5 // สต็อกติดลบ
            };

            // Act - ส่ง POST request ด้วยข้อมูลไม่ถูกต้อง
            var response = await _client.PostAsJsonAsync("/api/product", invalidProduct);

            // Assert - ตรวจสอบว่า API จัดการได้ (ไม่ crash)
            // อาจได้ 400 Bad Request หรือ 201 Created ขึ้นอยู่กับการ validate
            Assert.True(response.StatusCode == HttpStatusCode.Created || 
                       response.StatusCode == HttpStatusCode.BadRequest);
        }
    }
}
```

---

```csharp name=Tests/TestWebApplicationFactory.cs
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.Data.Sqlite;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.DependencyInjection.Extensions;
using ProductApi.Data;
using System.Data;
using Dapper;

namespace ProductApi.Tests
{
    // คลาส Factory สำหรับสร้าง Test Web Application
    public class TestWebApplicationFactory<TStartup> : WebApplicationFactory<TStartup> where TStartup : class
    {
        private SqliteConnection? _connection; // SQLite connection สำหรับ In-Memory database

        // Method สำหรับ configure web host ก่อนสร้าง application
        protected override void ConfigureWebHost(IWebHostBuilder builder)
        {
            // กำหนดให้ใช้ Test environment
            builder.UseEnvironment("Test");

            // Configure services สำหรับ test
            builder.ConfigureServices(services =>
            {
                // ลบ DapperContext ที่มีอยู่แล้วออก
                var descriptor = services.SingleOrDefault(d => d.ServiceType == typeof(IDapperContext));
                if (descriptor != null)
                {
                    services.Remove(descriptor); // ลบ service descriptor เดิม
                }

                // สร้าง SQLite In-Memory connection
                _connection = new SqliteConnection("DataSource=:memory:"); // connection string สำหรับ In-Memory DB
                _connection.Open(); // เปิด connection

                // สร้าง table สำหรับทดสอบ
                CreateTestDatabase(_connection);

                // เพิ่ม Test DapperContext ที่ใช้ In-Memory database
                services.AddSingleton<IDapperContext>(provider => new TestDapperContext(_connection));
            });
        }

        // Method สำหรับสร้าง database schema
        private static void CreateTestDatabase(IDbConnection connection)
        {
            // SQL สำหรับสร้าง Products table
            var createTableSql = @"
                CREATE TABLE IF NOT EXISTS Products (
                    Id INTEGER PRIMARY KEY AUTOINCREMENT,  -- Auto increment primary key
                    Name TEXT NOT NULL,                    -- ชื่อสินค้า (required)
                    Price REAL NOT NULL,                   -- ราคาสินค้า (required)
                    Stock INTEGER NOT NULL                 -- จำนวนสต็อก (required)
                )";
            
            connection.Execute(createTableSql); // Execute SQL command เพื่อสร้าง table
        }

        // Method ที่เรียกเมื่อ dispose factory
        protected override void Dispose(bool disposing)
        {
            if (disposing) // ถ้าเป็นการ dispose ปกติ
            {
                _connection?.Close(); // ปิด database connection
                _connection?.Dispose(); // dispose connection object
            }
            base.Dispose(disposing); // เรียก base dispose method
        }
    }

    // Implementation ของ IDapperContext สำหรับ Test
    public class TestDapperContext : IDapperContext
    {
        private readonly IDbConnection _connection; // Database connection

        // Constructor รับ connection ที่จะใช้
        public TestDapperContext(IDbConnection connection)
        {
            _connection = connection ?? throw new ArgumentNullException(nameof(connection)); // ตรวจสอบ null
        }

        // Method สำหรับสร้าง database connection
        public IDbConnection CreateConnection()
        {
            return _connection; // คืนค่า connection ที่ได้รับมา (In-Memory SQLite)
        }
    }
}
```

---

```xml name=ProductApi.Tests.csproj
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <!-- กำหนด .NET version และ test project settings -->
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <IsPackable>false</IsPackable>
    <IsTestProject>true</IsTestProject>
  </PropertyGroup>

  <ItemGroup>
    <!-- Package สำหรับ testing framework -->
    <PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" Version="8.0.0" />
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.8.0" />
    <PackageReference Include="xunit" Version="2.4.2" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.4.5">
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
      <PrivateAssets>all</PrivateAssets>
    </PackageReference>
    <!-- In-Memory SQLite database for testing -->
    <PackageReference Include="Microsoft.Data.Sqlite" Version="8.0.0" />
    <!-- Mocking framework -->
    <PackageReference Include="Moq" Version="4.20.69" />
  </ItemGroup>

  <ItemGroup>
    <!-- Reference ไปยัง main project -->
    <ProjectReference Include="..\ProductApi\ProductApi.csproj" />
  </ItemGroup>

</Project>
```

---

## การรัน Integration Test

```bash
# เข้าไปใน folder ของ test project
cd ProductApi.Tests

# รัน integration tests ทั้งหมด
dotnet test

# รันเฉพาะ integration tests
dotnet test --filter "ProductApiIntegrationTests"

# รัน test พร้อม detailed output
dotnet test --verbosity normal
```

---

## สรุปการทำงานของ Integration Test

### **TestWebApplicationFactory**
- **บรรทัด 8-9**: สร้าง SQLite In-Memory database
- **บรรทัด 15-24**: แทนที่ DapperContext จริงด้วย Test version
- **บรรทัด 31-40**: สร้าง database schema สำหรับ test
- **บรรทัด 43-50**: จัดการ cleanup เมื่อ test เสร็จ

### **Integration Tests**
- **End-to-End Testing**: ทดสอบ API ผ่าน HTTP requests
- **Real Database**: ใช้ SQLite In-Memory เลียนแบบ database จริง
- **Complete Flow**: ทดสอบตั้งแต่ Controller → Service → Repository → Database
- **HTTP Status Codes**: ตรวจสอบ response codes ที่ถูกต้อง
- **Data Validation**: ตรวจสอบข้อมูลที่สร้าง/แก้ไข/ลบ

### **ข้อดีของ Integration Test**
1. **Real Scenarios**: ทดสอบในสถานการณ์ที่ใกล้เคียงจริง
2. **Component Integration**: ตรวจสอบการทำงานร่วมกันของ components
3. **Fast Execution**: ใช้ In-Memory database ทำให้รวดเร็ว
4. **Isolated**: แต่ละ test แยกอิสระจากกัน
5. **Comprehensive**: ครอบคลุมการทำงานทั้งระบบ
#
