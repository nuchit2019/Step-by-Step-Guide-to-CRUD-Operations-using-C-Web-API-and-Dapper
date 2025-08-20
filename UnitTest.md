# Unit Test ตาม Best Practices ที่จะช่วยเพิ่ม Productivity ได้จริง...

## 1. Service Layer Unit Tests (Business Logic Testing)

```csharp name=Tests/Unit/ProductServiceTests.cs
using Moq;
using ProductApi.Models;
using ProductApi.Repositories;
using ProductApi.Services;
using Xunit;

namespace ProductApi.Tests.Unit
{
    /// <summary>
    /// Unit Test สำหรับ ProductService - หัวใจของ Business Logic
    /// ช่วยให้มั่นใจว่า Business Rules ทำงานถูกต้อง
    /// </summary>
    public class ProductServiceTests
    {
        private readonly Mock<IProductRepository> _mockRepository; // Mock Repository
        private readonly ProductService _service; // Service ที่จะทดสอบ

        public ProductServiceTests()
        {
            _mockRepository = new Mock<IProductRepository>(); // สร้าง Mock Repository
            _service = new ProductService(_mockRepository.Object); // Inject Mock เข้า Service
        }

        #region GetAllAsync Tests
        /// <summary>
        /// Test ที่ช่วย Productivity: ตรวจสอบว่า Service ส่งต่อข้อมูลจาก Repository ถูกต้อง
        /// ป้องกันข้อผิดพลาดในการ map/transform ข้อมูล
        /// </summary>
        [Fact]
        public async Task GetAllAsync_ShouldReturnAllProductsFromRepository()
        {
            // Arrange - เตรียมข้อมูลทดสอบ
            var expectedProducts = new List<Product>
            {
                new() { Id = 1, Name = "Product 1", Price = 100.00m, Stock = 10 },
                new() { Id = 2, Name = "Product 2", Price = 200.00m, Stock = 20 }
            };

            // Setup Mock Repository ให้คืนค่าข้อมูลที่เตรียมไว้
            _mockRepository.Setup(repo => repo.GetAllAsync())
                          .ReturnsAsync(expectedProducts);

            // Act - เรียกใช้ method ที่จะทดสอบ
            var result = await _service.GetAllAsync();

            // Assert - ตรวจสอบผลลัพธ์
            Assert.NotNull(result); // ตรวจสอบว่าผลลัพธ์ไม่เป็น null
            Assert.Equal(expectedProducts.Count, result.Count()); // ตรวจสอบจำนวน
            Assert.Equal(expectedProducts, result); // ตรวจสอบข้อมูล

            // Verify - ตรวจสอบว่า Repository method ถูกเรียกครั้งเดียว
            _mockRepository.Verify(repo => repo.GetAllAsync(), Times.Once);
        }

        /// <summary>
        /// Test Edge Case: ตรวจสอบเมื่อไม่มีข้อมูล
        /// ป้องกัน NullReferenceException ใน Production
        /// </summary>
        [Fact]
        public async Task GetAllAsync_WhenNoProducts_ShouldReturnEmptyList()
        {
            // Arrange
            _mockRepository.Setup(repo => repo.GetAllAsync())
                          .ReturnsAsync(new List<Product>());

            // Act
            var result = await _service.GetAllAsync();

            // Assert
            Assert.NotNull(result);
            Assert.Empty(result);
        }
        #endregion

        #region GetByIdAsync Tests
        /// <summary>
        /// Test Happy Path: ช่วยให้มั่นใจว่าการค้นหาด้วย ID ทำงานถูกต้อง
        /// </summary>
        [Fact]
        public async Task GetByIdAsync_WithValidId_ShouldReturnProduct()
        {
            // Arrange
            var productId = 1;
            var expectedProduct = new Product 
            { 
                Id = productId, 
                Name = "Test Product", 
                Price = 99.99m, 
                Stock = 15 
            };

            _mockRepository.Setup(repo => repo.GetByIdAsync(productId))
                          .ReturnsAsync(expectedProduct);

            // Act
            var result = await _service.GetByIdAsync(productId);

            // Assert
            Assert.NotNull(result);
            Assert.Equal(expectedProduct.Id, result.Id);
            Assert.Equal(expectedProduct.Name, result.Name);
            
            // Verify Repository เรียกด้วย parameter ที่ถูกต้อง
            _mockRepository.Verify(repo => repo.GetByIdAsync(productId), Times.Once);
        }

        /// <summary>
        /// Test Critical Edge Case: ป้องกัน Application Crash เมื่อไม่พบข้อมูล
        /// </summary>
        [Fact]
        public async Task GetByIdAsync_WithNonExistentId_ShouldReturnNull()
        {
            // Arrange
            var nonExistentId = 999;
            _mockRepository.Setup(repo => repo.GetByIdAsync(nonExistentId))
                          .ReturnsAsync((Product?)null);

            // Act
            var result = await _service.GetByIdAsync(nonExistentId);

            // Assert
            Assert.Null(result);
        }
        #endregion

        #region AddAsync Tests
        /// <summary>
        /// Test Business Rule: ตรวจสอบการสร้างข้อมูลใหม่
        /// ช่วยให้มั่นใจว่า validation และ data flow ถูกต้อง
        /// </summary>
        [Fact]
        public async Task AddAsync_WithValidProduct_ShouldReturnNewId()
        {
            // Arrange
            var newProduct = new Product 
            { 
                Name = "New Product", 
                Price = 150.00m, 
                Stock = 25 
            };
            var expectedId = 5;

            _mockRepository.Setup(repo => repo.AddAsync(newProduct))
                          .ReturnsAsync(expectedId);

            // Act
            var result = await _service.AddAsync(newProduct);

            // Assert
            Assert.Equal(expectedId, result);
            
            // Verify ว่า Repository ได้รับ object ที่ถูกต้อง
            _mockRepository.Verify(repo => repo.AddAsync(It.Is<Product>(p => 
                p.Name == newProduct.Name && 
                p.Price == newProduct.Price && 
                p.Stock == newProduct.Stock)), Times.Once);
        }
        #endregion

        #region UpdateAsync Tests
        /// <summary>
        /// Test Update Operation: ช่วยให้มั่นใจว่าการอัพเดททำงานถูกต้อง
        /// ป้องกันการ overwrite ข้อมูลผิด
        /// </summary>
        [Fact]
        public async Task UpdateAsync_WithValidProduct_ShouldReturnTrue()
        {
            // Arrange
            var productToUpdate = new Product 
            { 
                Id = 1, 
                Name = "Updated Product", 
                Price = 199.99m, 
                Stock = 30 
            };

            _mockRepository.Setup(repo => repo.UpdateAsync(productToUpdate))
                          .ReturnsAsync(true);

            // Act
            var result = await _service.UpdateAsync(productToUpdate);

            // Assert
            Assert.True(result);
        }

        /// <summary>
        /// Test Failure Scenario: ตรวจสอบเมื่ออัพเดทไม่สำเร็จ
        /// ช่วยให้ระบบจัดการ error ได้ถูกต้อง
        /// </summary>
        [Fact]
        public async Task UpdateAsync_WithNonExistentProduct_ShouldReturnFalse()
        {
            // Arrange
            var nonExistentProduct = new Product 
            { 
                Id = 999, 
                Name = "Non-existent", 
                Price = 100.00m, 
                Stock = 10 
            };

            _mockRepository.Setup(repo => repo.UpdateAsync(nonExistentProduct))
                          .ReturnsAsync(false);

            // Act
            var result = await _service.UpdateAsync(nonExistentProduct);

            // Assert
            Assert.False(result);
        }
        #endregion

        #region DeleteAsync Tests
        /// <summary>
        /// Test Delete Operation: ช่วยป้องกันการลบข้อมูลผิด
        /// </summary>
        [Theory] // ใช้ Theory สำหรับ test หลาย scenarios
        [InlineData(1, true)]   // Case: ลบสำเร็จ
        [InlineData(999, false)] // Case: ไม่พบข้อมูลที่จะลบ
        public async Task DeleteAsync_ShouldReturnExpectedResult(int productId, bool expectedResult)
        {
            // Arrange
            _mockRepository.Setup(repo => repo.DeleteAsync(productId))
                          .ReturnsAsync(expectedResult);

            // Act
            var result = await _service.DeleteAsync(productId);

            // Assert
            Assert.Equal(expectedResult, result);
            _mockRepository.Verify(repo => repo.DeleteAsync(productId), Times.Once);
        }
        #endregion
    }
}
```

---

## 2. Repository Layer Unit Tests (Data Access Testing)

```csharp name=Tests/Unit/ProductRepositoryTests.cs
using System.Data;
using Dapper;
using Microsoft.Data.Sqlite;
using ProductApi.Data;
using ProductApi.Models;
using ProductApi.Repositories;
using Moq;
using Xunit;

namespace ProductApi.Tests.Unit
{
    /// <summary>
    /// Unit Test สำหรับ ProductRepository - Data Access Layer
    /// ช่วยให้มั่นใจว่า SQL queries และ data mapping ถูกต้อง
    /// </summary>
    public class ProductRepositoryTests : IDisposable
    {
        private readonly IDbConnection _connection;
        private readonly Mock<IDapperContext> _mockContext;
        private readonly ProductRepository _repository;

        public ProductRepositoryTests()
        {
            // ใช้ SQLite In-Memory สำหรับ isolated testing
            _connection = new SqliteConnection("DataSource=:memory:");
            _connection.Open();
            
            // Setup Mock Context
            _mockContext = new Mock<IDapperContext>();
            _mockContext.Setup(x => x.CreateConnection()).Returns(_connection);
            
            _repository = new ProductRepository(_mockContext.Object);
            
            CreateTestTable();
        }

        /// <summary>
        /// สร้าง table structure สำหรับ test
        /// ช่วยให้ test database schema ตรงกับ production
        /// </summary>
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

        #region GetAllAsync Tests
        /// <summary>
        /// Test SQL Query: ตรวจสอบว่า SELECT query ทำงานถูกต้อง
        /// ช่วยป้องกัน SQL injection และ performance issues
        /// </summary>
        [Fact]
        public async Task GetAllAsync_WithMultipleProducts_ShouldReturnAllProducts()
        {
            // Arrange - Insert test data
            var testProducts = new[]
            {
                new { Name = "Product A", Price = 100.50m, Stock = 15 },
                new { Name = "Product B", Price = 200.75m, Stock = 25 },
                new { Name = "Product C", Price = 300.25m, Stock = 35 }
            };

            var insertSql = "INSERT INTO Products (Name, Price, Stock) VALUES (@Name, @Price, @Stock)";
            foreach (var product in testProducts)
            {
                await _connection.ExecuteAsync(insertSql, product);
            }

            // Act
            var result = await _repository.GetAllAsync();
            var products = result.ToList();

            // Assert
            Assert.Equal(testProducts.Length, products.Count);
            
            // ตรวจสอบ data mapping ถูกต้อง
            for (int i = 0; i < testProducts.Length; i++)
            {
                Assert.Equal(testProducts[i].Name, products[i].Name);
                Assert.Equal(testProducts[i].Price, products[i].Price);
                Assert.Equal(testProducts[i].Stock, products[i].Stock);
            }

            // Verify Context ถูกเรียกใช้
            _mockContext.Verify(x => x.CreateConnection(), Times.Once);
        }

        /// <summary>
        /// Test Empty Result: ป้องกัน null reference exceptions
        /// </summary>
        [Fact]
        public async Task GetAllAsync_WithNoProducts_ShouldReturnEmptyEnumerable()
        {
            // Act
            var result = await _repository.GetAllAsync();

            // Assert
            Assert.NotNull(result);
            Assert.Empty(result);
        }
        #endregion

        #region GetByIdAsync Tests
        /// <summary>
        /// Test Parameterized Query: ป้องกัน SQL injection
        /// ตรวจสอบการ bind parameters ถูกต้อง
        /// </summary>
        [Fact]
        public async Task GetByIdAsync_WithExistingId_ShouldReturnCorrectProduct()
        {
            // Arrange
            var expectedProduct = new Product
            {
                Name = "Test Product",
                Price = 99.99m,
                Stock = 20
            };

            var insertSql = "INSERT INTO Products (Name, Price, Stock) VALUES (@Name, @Price, @Stock); SELECT last_insert_rowid();";
            var insertedId = await _connection.QuerySingleAsync<int>(insertSql, expectedProduct);

            // Act
            var result = await _repository.GetByIdAsync(insertedId);

            // Assert
            Assert.NotNull(result);
            Assert.Equal(insertedId, result.Id);
            Assert.Equal(expectedProduct.Name, result.Name);
            Assert.Equal(expectedProduct.Price, result.Price);
            Assert.Equal(expectedProduct.Stock, result.Stock);
        }

        /// <summary>
        /// Test Invalid ID: ตรวจสอบการจัดการเมื่อไม่พบข้อมูล
        /// </summary>
        [Theory]
        [InlineData(-1)]     // Negative ID
        [InlineData(0)]      // Zero ID
        [InlineData(99999)]  // Non-existent ID
        public async Task GetByIdAsync_WithInvalidId_ShouldReturnNull(int invalidId)
        {
            // Act
            var result = await _repository.GetByIdAsync(invalidId);

            // Assert
            Assert.Null(result);
        }
        #endregion

        #region AddAsync Tests
        /// <summary>
        /// Test Insert Operation: ตรวจสอบการสร้างข้อมูลใหม่
        /// ช่วยให้มั่นใจว่า auto-increment ID ทำงานถูกต้อง
        /// </summary>
        [Fact]
        public async Task AddAsync_WithValidProduct_ShouldReturnPositiveId()
        {
            // Arrange
            var newProduct = new Product
            {
                Name = "New Product Test",
                Price = 149.99m,
                Stock = 30
            };

            // Act
            var newId = await _repository.AddAsync(newProduct);

            // Assert
            Assert.True(newId > 0, "New ID should be positive");

            // Verify data was actually inserted
            var insertedProduct = await _repository.GetByIdAsync(newId);
            Assert.NotNull(insertedProduct);
            Assert.Equal(newProduct.Name, insertedProduct.Name);
            Assert.Equal(newProduct.Price, insertedProduct.Price);
            Assert.Equal(newProduct.Stock, insertedProduct.Stock);
        }

        /// <summary>
        /// Test Data Validation: ตรวจสอบการ handle ข้อมูลที่มี special characters
        /// ช่วยป้องกัน encoding issues
        /// </summary>
        [Fact]
        public async Task AddAsync_WithSpecialCharacters_ShouldHandleCorrectly()
        {
            // Arrange
            var productWithSpecialChars = new Product
            {
                Name = "Product with 'quotes' & symbols $#@",
                Price = 199.99m,
                Stock = 10
            };

            // Act
            var newId = await _repository.AddAsync(productWithSpecialChars);

            // Assert
            Assert.True(newId > 0);
            var retrievedProduct = await _repository.GetByIdAsync(newId);
            Assert.Equal(productWithSpecialChars.Name, retrievedProduct?.Name);
        }
        #endregion

        #region UpdateAsync Tests
        /// <summary>
        /// Test Update Operation: ตรวจสอบการแก้ไขข้อมูล
        /// ช่วยป้องกันการ overwrite ข้อมูลผิด
        /// </summary>
        [Fact]
        public async Task UpdateAsync_WithExistingProduct_ShouldReturnTrueAndUpdateData()
        {
            // Arrange - สร้างข้อมูลเดิม
            var originalProduct = new Product
            {
                Name = "Original Name",
                Price = 100.00m,
                Stock = 15
            };
            var productId = await _repository.AddAsync(originalProduct);

            // เตรียมข้อมูลที่จะอัพเดท
            var updatedProduct = new Product
            {
                Id = productId,
                Name = "Updated Name",
                Price = 150.00m,
                Stock = 25
            };

            // Act
            var result = await _repository.UpdateAsync(updatedProduct);

            // Assert
            Assert.True(result);

            // Verify ข้อมูลถูกอัพเดทจริง
            var retrievedProduct = await _repository.GetByIdAsync(productId);
            Assert.NotNull(retrievedProduct);
            Assert.Equal(updatedProduct.Name, retrievedProduct.Name);
            Assert.Equal(updatedProduct.Price, retrievedProduct.Price);
            Assert.Equal(updatedProduct.Stock, retrievedProduct.Stock);
        }

        /// <summary>
        /// Test Update Non-existent: ตรวจสอบเมื่ออัพเดท record ที่ไม่มีอยู่
        /// </summary>
        [Fact]
        public async Task UpdateAsync_WithNonExistentProduct_ShouldReturnFalse()
        {
            // Arrange
            var nonExistentProduct = new Product
            {
                Id = 99999,
                Name = "Non-existent",
                Price = 100.00m,
                Stock = 10
            };

            // Act
            var result = await _repository.UpdateAsync(nonExistentProduct);

            // Assert
            Assert.False(result);
        }
        #endregion

        #region DeleteAsync Tests
        /// <summary>
        /// Test Delete Operation: ตรวจสอบการลบข้อมูล
        /// ช่วยป้องกันการลบข้อมูลผิด
        /// </summary>
        [Fact]
        public async Task DeleteAsync_WithExistingId_ShouldReturnTrueAndDeleteData()
        {
            // Arrange
            var productToDelete = new Product
            {
                Name = "To Delete",
                Price = 50.00m,
                Stock = 5
            };
            var productId = await _repository.AddAsync(productToDelete);

            // Act
            var result = await _repository.DeleteAsync(productId);

            // Assert
            Assert.True(result);

            // Verify ข้อมูลถูกลบจริง
            var deletedProduct = await _repository.GetByIdAsync(productId);
            Assert.Null(deletedProduct);
        }

        /// <summary>
        /// Test Delete Non-existent: ตรวจสอบเมื่อลบ record ที่ไม่มีอยู่
        /// </summary>
        [Fact]
        public async Task DeleteAsync_WithNonExistentId_ShouldReturnFalse()
        {
            // Act
            var result = await _repository.DeleteAsync(99999);

            // Assert
            Assert.False(result);
        }
        #endregion

        #region Performance Tests
        /// <summary>
        /// Test Performance: ตรวจสอบประสิทธิภาพเมื่อมีข้อมูลจำนวนมาก
        /// ช่วยป้องกัน performance bottlenecks
        /// </summary>
        [Fact]
        public async Task GetAllAsync_WithLargeDataSet_ShouldCompleteInReasonableTime()
        {
            // Arrange - สร้างข้อมูลจำนวนมาก
            var products = Enumerable.Range(1, 1000).Select(i => new
            {
                Name = $"Product {i}",
                Price = (decimal)(i * 10.5),
                Stock = i % 100
            });

            var insertSql = "INSERT INTO Products (Name, Price, Stock) VALUES (@Name, @Price, @Stock)";
            await _connection.ExecuteAsync(insertSql, products);

            // Act & Assert - วัดเวลาการทำงาน
            var stopwatch = System.Diagnostics.Stopwatch.StartNew();
            var result = await _repository.GetAllAsync();
            stopwatch.Stop();

            Assert.Equal(1000, result.Count());
            Assert.True(stopwatch.ElapsedMilliseconds < 1000, 
                       $"Query took too long: {stopwatch.ElapsedMilliseconds}ms");
        }
        #endregion

        public void Dispose()
        {
            _connection?.Dispose();
        }
    }
}
```

---

## 3. Controller Unit Tests (API Layer Testing)

```csharp name=Tests/Unit/ProductControllerTests.cs
using Microsoft.AspNetCore.Mvc;
using Moq;
using ProductApi.Controllers;
using ProductApi.Models;
using ProductApi.Services;
using Xunit;

namespace ProductApi.Tests.Unit
{
    /// <summary>
    /// Unit Test สำหรับ ProductController - API Layer
    /// ช่วยให้มั่นใจว่า HTTP responses และ status codes ถูกต้อง
    /// ป้องกัน API breaking changes
    /// </summary>
    public class ProductControllerTests
    {
        private readonly Mock<IProductService> _mockService;
        private readonly ProductController _controller;

        public ProductControllerTests()
        {
            _mockService = new Mock<IProductService>();
            _controller = new ProductController(_mockService.Object);
        }

        #region GetAll Tests
        /// <summary>
        /// Test API Response: ตรวจสอบ HTTP 200 OK response
        /// ช่วยป้องกัน API contract breaking
        /// </summary>
        [Fact]
        public async Task GetAll_WithProducts_ShouldReturnOkWithProducts()
        {
            // Arrange
            var products = new List<Product>
            {
                new() { Id = 1, Name = "Product 1", Price = 100.00m, Stock = 10 },
                new() { Id = 2, Name = "Product 2", Price = 200.00m, Stock = 20 }
            };

            _mockService.Setup(service => service.GetAllAsync())
                       .ReturnsAsync(products);

            // Act
            var result = await _controller.GetAll();

            // Assert
            var okResult = Assert.IsType<OkObjectResult>(result);
            var returnedProducts = Assert.IsAssignableFrom<IEnumerable<Product>>(okResult.Value);
            Assert.Equal(products.Count, returnedProducts.Count());
        }

        /// <summary>
        /// Test Empty Response: ตรวจสอบ empty list response
        /// </summary>
        [Fact]
        public async Task GetAll_WithNoProducts_ShouldReturnOkWithEmptyList()
        {
            // Arrange
            _mockService.Setup(service => service.GetAllAsync())
                       .ReturnsAsync(new List<Product>());

            // Act
            var result = await _controller.GetAll();

            // Assert
            var okResult = Assert.IsType<OkObjectResult>(result);
            var returnedProducts = Assert.IsAssignableFrom<IEnumerable<Product>>(okResult.Value);
            Assert.Empty(returnedProducts);
        }
        #endregion

        #region GetById Tests
        /// <summary>
        /// Test Success Response: ตรวจสอบ HTTP 200 OK
        /// </summary>
        [Fact]
        public async Task GetById_WithExistingId_ShouldReturnOkWithProduct()
        {
            // Arrange
            var productId = 1;
            var expectedProduct = new Product 
            { 
                Id = productId, 
                Name = "Test Product", 
                Price = 99.99m, 
                Stock = 15 
            };

            _mockService.Setup(service => service.GetByIdAsync(productId))
                       .ReturnsAsync(expectedProduct);

            // Act
            var result = await _controller.GetById(productId);

            // Assert
            var okResult = Assert.IsType<OkObjectResult>(result);
            var returnedProduct = Assert.IsType<Product>(okResult.Value);
            Assert.Equal(expectedProduct.Id, returnedProduct.Id);
            Assert.Equal(expectedProduct.Name, returnedProduct.Name);
        }

        /// <summary>
        /// Test Not Found Response: ตรวจสอบ HTTP 404 Not Found
        /// Critical for API consumers
        /// </summary>
        [Fact]
        public async Task GetById_WithNonExistentId_ShouldReturnNotFound()
        {
            // Arrange
            var nonExistentId = 999;
            _mockService.Setup(service => service.GetByIdAsync(nonExistentId))
                       .ReturnsAsync((Product?)null);

            // Act
            var result = await _controller.GetById(nonExistentId);

            // Assert
            Assert.IsType<NotFoundResult>(result);
        }
        #endregion

        #region Create Tests
        /// <summary>
        /// Test Create Success: ตรวจสอบ HTTP 201 Created
        /// รวมถึง Location header สำหรับ RESTful API
        /// </summary>
        [Fact]
        public async Task Create_WithValidProduct_ShouldReturnCreatedAtAction()
        {
            // Arrange
            var newProduct = new Product 
            { 
                Name = "New Product", 
                Price = 150.00m, 
                Stock = 25 
            };
            var expectedId = 5;

            _mockService.Setup(service => service.AddAsync(newProduct))
                       .ReturnsAsync(expectedId);

            // Act
            var result = await _controller.Create(newProduct);

            // Assert
            var createdResult = Assert.IsType<CreatedAtActionResult>(result);
            Assert.Equal(nameof(_controller.GetById), createdResult.ActionName);
            Assert.Equal(expectedId, createdResult.RouteValues?["id"]);
            Assert.Equal(newProduct, createdResult.Value);
        }
        #endregion

        #region Update Tests
        /// <summary>
        /// Test Update Success: ตรวจสอบ HTTP 204 No Content
        /// </summary>
        [Fact]
        public async Task Update_WithValidProduct_ShouldReturnNoContent()
        {
            // Arrange
            var productId = 1;
            var productToUpdate = new Product 
            { 
                Id = productId, 
                Name = "Updated Product", 
                Price = 199.99m, 
                Stock = 30 
            };

            _mockService.Setup(service => service.UpdateAsync(productToUpdate))
                       .ReturnsAsync(true);

            // Act
            var result = await _controller.Update(productId, productToUpdate);

            // Assert
            Assert.IsType<NoContentResult>(result);
        }

        /// <summary>
        /// Test ID Mismatch: ตรวจสอบ HTTP 400 Bad Request
        /// ช่วยป้องกัน data corruption
        /// </summary>
        [Fact]
        public async Task Update_WithMismatchedId_ShouldReturnBadRequest()
        {
            // Arrange
            var urlId = 1;
            var productWithDifferentId = new Product 
            { 
                Id = 2, // ID ไม่ตรงกับ URL
                Name = "Product", 
                Price = 100.00m, 
                Stock = 10 
            };

            // Act
            var result = await _controller.Update(urlId, productWithDifferentId);

            // Assert
            Assert.IsType<BadRequestResult>(result);
        }

        /// <summary>
        /// Test Update Not Found: ตรวจสอบ HTTP 404 Not Found
        /// </summary>
        [Fact]
        public async Task Update_WithNonExistentProduct_ShouldReturnNotFound()
        {
            // Arrange
            var productId = 999;
            var nonExistentProduct = new Product 
            { 
                Id = productId, 
                Name = "Non-existent", 
                Price = 100.00m, 
                Stock = 10 
            };

            _mockService.Setup(service => service.UpdateAsync(nonExistentProduct))
                       .ReturnsAsync(false);

            // Act
            var result = await _controller.Update(productId, nonExistentProduct);

            // Assert
            Assert.IsType<NotFoundResult>(result);
        }
        #endregion

        #region Delete Tests
        /// <summary>
        /// Test Delete Success: ตรวจสอบ HTTP 204 No Content
        /// </summary>
        [Fact]
        public async Task Delete_WithExistingId_ShouldReturnNoContent()
        {
            // Arrange
            var productId = 1;
            _mockService.Setup(service => service.DeleteAsync(productId))
                       .ReturnsAsync(true);

            // Act
            var result = await _controller.Delete(productId);

            // Assert
            Assert.IsType<NoContentResult>(result);
        }

        /// <summary>
        /// Test Delete Not Found: ตรวจสอบ HTTP 404 Not Found
        /// </summary>
        [Fact]
        public async Task Delete_WithNonExistentId_ShouldReturnNotFound()
        {
            // Arrange
            var nonExistentId = 999;
            _mockService.Setup(service => service.DeleteAsync(nonExistentId))
                       .ReturnsAsync(false);

            // Act
            var result = await _controller.Delete(nonExistentId);

            // Assert
            Assert.IsType<NotFoundResult>(result);
        }
        #endregion
    }
}
```

---

## 4. Test Utilities และ Best Practices

```csharp name=Tests/Utilities/TestDataBuilder.cs
using ProductApi.Models;

namespace ProductApi.Tests.Utilities
{
    /// <summary>
    /// Builder Pattern สำหรับสร้าง test data
    /// ช่วยให้ test code สะอาดและ maintainable
    /// เพิ่ม productivity ในการเขียน test
    /// </summary>
    public class ProductTestDataBuilder
    {
        private int _id = 0;
        private string _name = "Default Product";
        private decimal _price = 100.00m;
        private int _stock = 10;

        /// <summary>
        /// สร้าง valid product สำหรับ happy path tests
        /// </summary>
        public static ProductTestDataBuilder CreateValid()
        {
            return new ProductTestDataBuilder();
        }

        /// <summary>
        /// สร้าง product ที่มี boundary values สำหรับ edge case tests
        /// </summary>
        public static ProductTestDataBuilder CreateBoundary()
        {
            return new ProductTestDataBuilder()
                .WithName("A") // Minimum length
                .WithPrice(0.01m) // Minimum price
                .WithStock(0); // Minimum stock
        }

        /// <summary>
        /// สร้าง product สำหรับ performance tests
        /// </summary>
        public static ProductTestDataBuilder CreateLarge()
        {
            return new ProductTestDataBuilder()
                .WithName(new string('X', 100)) // Maximum reasonable length
                .WithPrice(999999.99m) // Large price
                .WithStock(int.MaxValue); // Maximum stock
        }

        public ProductTestDataBuilder WithId(int id)
        {
            _id = id;
            return this;
        }

        public ProductTestDataBuilder WithName(string name)
        {
            _name = name;
            return this;
        }

        public ProductTestDataBuilder WithPrice(decimal price)
        {
            _price = price;
            return this;
        }

        public ProductTestDataBuilder WithStock(int stock)
        {
            _stock = stock;
            return this;
        }

        public Product Build()
        {
            return new Product
            {
                Id = _id,
                Name = _name,
                Price = _price,
                Stock = _stock
            };
        }

        /// <summary>
        /// สร้าง collection ของ products สำหรับ bulk testing
        /// </summary>
        public List<Product> BuildMany(int count)
        {
            return Enumerable.Range(1, count)
                           .Select(i => new ProductTestDataBuilder()
                               .WithId(i)
                               .WithName($"{_name} {i}")
                               .WithPrice(_price + i)
                               .WithStock(_stock + i)
                               .Build())
                           .ToList();
        }
    }
}
```

---

```csharp name=Tests/Utilities/MockExtensions.cs
using Moq;
using ProductApi.Models;
using ProductApi.Repositories;
using ProductApi.Services;

namespace ProductApi.Tests.Utilities
{
    /// <summary>
    /// Extension methods สำหรับ Mock setup
    /// ช่วยลด boilerplate code และเพิ่ม readability
    /// </summary>
    public static class MockExtensions
    {
        /// <summary>
        /// Setup Mock Repository สำหรับ success scenarios
        /// </summary>
        public static Mock<IProductRepository> SetupSuccessfulRepository(this Mock<IProductRepository> mock, List<Product> products)
        {
            // Setup GetAll
            mock.Setup(r => r.GetAllAsync()).ReturnsAsync(products);

            // Setup GetById for existing products
            foreach (var product in products)
            {
                mock.Setup(r => r.GetByIdAsync(product.Id)).ReturnsAsync(product);
            }

            // Setup GetById for non-existing products
            mock.Setup(r => r.GetByIdAsync(It.Is<int>(id => !products.Any(p => p.Id == id))))
                .ReturnsAsync((Product?)null);

            // Setup Add
            mock.Setup(r => r.AddAsync(It.IsAny<Product>()))
                .ReturnsAsync((Product p) => products.Count + 1);

            // Setup Update
            mock.Setup(r => r.UpdateAsync(It.Is<Product>(p => products.Any(existing => existing.Id == p.Id))))
                .ReturnsAsync(true);
            mock.Setup(r => r.UpdateAsync(It.Is<Product>(p => !products.Any(existing => existing.Id == p.Id))))
                .ReturnsAsync(false);

            // Setup Delete
            mock.Setup(r => r.DeleteAsync(It.Is<int>(id => products.Any(p => p.Id == id))))
                .ReturnsAsync(true);
            mock.Setup(r => r.DeleteAsync(It.Is<int>(id => !products.Any(p => p.Id == id))))
                .ReturnsAsync(false);

            return mock;
        }

        /// <summary>
        /// Setup Mock Service สำหรับ Controller tests
        /// </summary>
        public static Mock<IProductService> SetupSuccessfulService(this Mock<IProductService> mock, List<Product> products)
        {
            // เรียกใช้ setup เดียวกันกับ Repository
            return mock.SetupSuccessfulRepository(products).As<IProductService>();
        }
    }
}
```

---

## 5. Test Configuration และ Collection

```csharp name=Tests/TestCollections.cs
using Xunit;

namespace ProductApi.Tests
{
    /// <summary>
    /// Test Collections สำหรับจัดการ test execution order
    /// และ shared resources
    /// </summary>
    
    [CollectionDefinition("Database Collection")]
    public class DatabaseCollection : ICollectionFixture<DatabaseFixture>
    {
        // This class has no code, and is never created. Its purpose is simply
        // to be the place to apply [CollectionDefinition] and all the
        // ICollectionFixture<> interfaces.
    }

    /// <summary>
    /// Shared fixture สำหรับ database tests
    /// ช่วยลดเวลาใน test setup
    /// </summary>
    public class DatabaseFixture : IDisposable
    {
        public string ConnectionString { get; private set; }

        public DatabaseFixture()
        {
            ConnectionString = "DataSource=:memory:";
            // Additional setup if needed
        }

        public void Dispose()
        {
            // Cleanup code
        }
    }
}
```

---

## สรุป Unit Tests ที่เพิ่ม Productivity จริง

### **1. Service Layer Tests** 
- **ป้องกัน Business Logic Bugs** ที่ยากต่อการ debug
- **Fast Feedback Loop** เมื่อแก้ไข business rules
- **Documentation** ของ business behavior

### **2. Repository Layer Tests**
- **ป้องกัน SQL Injection** และ data corruption
- **ตรวจสอบ Performance** ของ queries
- **Database Schema Validation**

### **3. Controller Layer Tests**
- **API Contract Testing** ป้องกัน breaking changes
- **HTTP Status Code Validation** สำหรับ API consumers
- **Request/Response Validation**

### **4. Test Utilities**
- **Builder Pattern** ลด boilerplate code
- **Mock Extensions** เพิ่ม readability
- **Shared Fixtures** ลดเวลา setup

### **5. ประโยชน์ที่เพิ่ม Productivity**

#### **Development Time:**
- **Fast Feedback**: รู้ทันทีเมื่อโค้ดพัง
- **Regression Prevention**: ป้องกันบัคเก่าเกิดขึ้นใหม่
- **Refactoring Confidence**: แก้โค้ดได้อย่างมั่นใจ

#### **Maintenance:**
- **Living Documentation**: test cases เป็น documentation
- **Easy Debugging**: isolate ปัญหาได้ง่าย
- **Change Impact Analysis**: เห็นผลกระทบของการเปลี่ยนแปลง

#### **Team Productivity:**
- **Code Review Efficiency**: reviewer มั่นใจมากขึ้น  
- **Onboarding**: developer ใหม่เข้าใจระบบเร็วขึ้น
- **Feature Development**: เพิ่มฟีเจอร์ใหม่ได้อย่างมั่นใจ

#
