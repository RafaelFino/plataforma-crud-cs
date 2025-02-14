# Criando uma API RESTful em C# com .NET Core e SQLite

Este guia explica passo a passo como criar uma API RESTful em **C# com .NET Core** utilizando **SQLite** como banco de dados, sem o uso de ORM, separando a aplicação em **Controle, Domínio e Repositório**.

## 1. Criando o Projeto

Execute o seguinte comando para criar um novo projeto Web API:

```sh
dotnet new webapi -n ProductApi
cd ProductApi
```

Isso criará um projeto básico de API.

## 2. Adicionando o Pacote do SQLite

Para trabalhar com o banco de dados SQLite, instale o pacote necessário:

```sh
dotnet add package System.Data.SQLite
```

## 3. Estrutura do Projeto

Organizaremos os arquivos conforme a estrutura abaixo:

```
ProductApi/
│── Controllers/
│   ├── ProductController.cs
│── Domain/
│   ├── Product.cs
│── Repositories/
│   ├── ProductRepository.cs
│── Database/
│   ├── DatabaseInitializer.cs
│── Program.cs
│── appsettings.json
```

Isso ajuda a manter o código organizado e modularizado.

## 4. Criando a Tabela de Produtos

Crie o arquivo **`Database/DatabaseInitializer.cs`**:

```csharp
using System.Data.SQLite;
using System.IO;

namespace ProductApi.Database
{
    public static class DatabaseInitializer
    {
        private const string DatabaseFile = "products.db";

        public static void Initialize()
        {
            if (!File.Exists(DatabaseFile))
            {
                SQLiteConnection.CreateFile(DatabaseFile);
            }

            using (var connection = new SQLiteConnection($"Data Source={DatabaseFile};Version=3;"))
            {
                connection.Open();
                string tableCreationQuery = @"
                    CREATE TABLE IF NOT EXISTS Products (
                        Id INTEGER PRIMARY KEY AUTOINCREMENT,
                        Name TEXT NOT NULL,
                        Description TEXT NOT NULL,
                        Price REAL NOT NULL
                    );";
                
                using (var command = new SQLiteCommand(tableCreationQuery, connection))
                {
                    command.ExecuteNonQuery();
                }
            }
        }
    }
}
```

## 5. Criando a Classe de Produto

Crie o arquivo **`Domain/Product.cs`**:

```csharp
namespace ProductApi.Domain
{
    public class Product
    {
        public int Id { get; set; }
        public string Name { get; set; } = string.Empty;
        public string Description { get; set; } = string.Empty;
        public decimal Price { get; set; }
    }
}
```

## 6. Criando o Repositório de Produtos

Crie o arquivo **`Repositories/ProductRepository.cs`**:

```csharp
using System.Collections.Generic;
using System.Data.SQLite;
using ProductApi.Domain;

namespace ProductApi.Repositories
{
    public class ProductRepository
    {
        private const string ConnectionString = "Data Source=products.db;Version=3;";

        public void Add(Product product)
        {
            using (var connection = new SQLiteConnection(ConnectionString))
            {
                connection.Open();
                string query = "INSERT INTO Products (Name, Description, Price) VALUES (@Name, @Description, @Price)";
                
                using (var command = new SQLiteCommand(query, connection))
                {
                    command.Parameters.AddWithValue("@Name", product.Name);
                    command.Parameters.AddWithValue("@Description", product.Description);
                    command.Parameters.AddWithValue("@Price", product.Price);
                    command.ExecuteNonQuery();
                }
            }
        }

        public List<Product> GetAll()
        {
            var products = new List<Product>();
            using (var connection = new SQLiteConnection(ConnectionString))
            {
                connection.Open();
                string query = "SELECT * FROM Products";
                
                using (var command = new SQLiteCommand(query, connection))
                using (var reader = command.ExecuteReader())
                {
                    while (reader.Read())
                    {
                        products.Add(new Product
                        {
                            Id = reader.GetInt32(0),
                            Name = reader.GetString(1),
                            Description = reader.GetString(2),
                            Price = reader.GetDecimal(3)
                        });
                    }
                }
            }
            return products;
        }

        public void Update(Product product)
        {
            using (var connection = new SQLiteConnection(ConnectionString))
            {
                connection.Open();
                string query = "UPDATE Products SET Name=@Name, Description=@Description, Price=@Price WHERE Id=@Id";
                
                using (var command = new SQLiteCommand(query, connection))
                {
                    command.Parameters.AddWithValue("@Id", product.Id);
                    command.Parameters.AddWithValue("@Name", product.Name);
                    command.Parameters.AddWithValue("@Description", product.Description);
                    command.Parameters.AddWithValue("@Price", product.Price);
                    command.ExecuteNonQuery();
                }
            }
        }

        public void Delete(int id)
        {
            using (var connection = new SQLiteConnection(ConnectionString))
            {
                connection.Open();
                string query = "DELETE FROM Products WHERE Id=@Id";

                using (var command = new SQLiteCommand(query, connection))
                {
                    command.Parameters.AddWithValue("@Id", id);
                    command.ExecuteNonQuery();
                }
            }
        }
    }
}
```

## 7. Criando o Controlador da API

Crie o arquivo **`Controllers/ProductController.cs`**:

```csharp
using Microsoft.AspNetCore.Mvc;
using ProductApi.Domain;
using ProductApi.Repositories;
using System.Collections.Generic;

namespace ProductApi.Controllers
{
    [ApiController]
    [Route("api/products")]
    public class ProductController : ControllerBase
    {
        private readonly ProductRepository _repository = new ProductRepository();

        [HttpPost]
        public IActionResult Create(Product product)
        {
            _repository.Add(product);
            return Created("", product);
        }

        [HttpGet]
        public ActionResult<List<Product>> GetAll()
        {
            return _repository.GetAll();
        }

        [HttpPut]
        public IActionResult Update(Product product)
        {
            _repository.Update(product);
            return NoContent();
        }

        [HttpDelete("{id}")]
        public IActionResult Delete(int id)
        {
            _repository.Delete(id);
            return NoContent();
        }
    }
}
```

## 8. Configurando e Executando a API

No arquivo **`Program.cs`**, adicione:

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using ProductApi.Database;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();

var app = builder.Build();
DatabaseInitializer.Initialize();

app.UseRouting();
app.MapControllers();
app.Run();
```

Agora, execute a API:
```sh
dotnet run
```

A API estará disponível em **http://localhost:5000/api/products**.

