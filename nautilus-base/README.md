# Nautilus.Base

[![Build Status](https://github.com/synthphonic/nautilus-base/actions/workflows/ci.yml/badge.svg)](https://github.com/synthphonic/nautilus-base/actions/workflows/ci.yml)
[![NuGet Version](https://img.shields.io/nuget/v/Nautilus.Base)](https://www.nuget.org/packages/Nautilus.Base/)
[![License: BSL-1.1](https://img.shields.io/badge/License-BSL--1.1-blue.svg)](LICENSE)

A foundational .NET library providing data access abstraction and configuration management for the Nautilus ecosystem. This library implements a provider pattern to support multiple database backends through a common, consistent API, enabling developers to build database-agnostic applications with ease.

## Features

- **Provider Pattern Architecture**: Abstract base class for implementing database-specific providers
- **Multi-Framework Support**: Targets `netstandard2.0`, `net6.0`, `net7.0`, and `net8.0`
- **CRUD Operations**: Complete set of Create, Read, Update, and Delete methods
- **Async/Await Support**: Async variants for all data operations
- **Entity Mapping**: Attribute-based mapping for entities to database tables
- **Pagination Support**: Built-in pagination for large datasets
- **Configuration Management**: JSON-based configuration loading with Microsoft.Extensions.Configuration
- **SQL Injection Protection**: Built-in SQL identifier validation for security
- **Reflection Caching**: Performance-optimized property metadata caching
- **Stored Procedure Support**: Execute stored procedures with multiple result sets
- **Bulk Operations**: Support for bulk insert operations

## Installation

Install the package via [NuGet](https://www.nuget.org/packages/Nautilus.Base/):

```bash
dotnet add package Nautilus.Base
```

Or via the Package Manager Console:

```powershell
Install-Package Nautilus.Base
```

## Quick Start

### Define an Entity with Attributes

```csharp
using Nautilus.Base.Data.ModelBase;
using Nautilus.Base.Data.Attributes;

// Basic entity with table and column mapping
[Table("application_users")]
public class User : EntityBase<int>
{
    [PrimaryKey]
    public int Id { get; set; }

    [Column("first_name")]
    public string FirstName { get; set; }

    [Column("last_name")]
    public string LastName { get; set; }

    public string Email { get; set; }
}

// Entity with schema specification
[Table("products", Schema = "inventory")]
public class Product : EntityBase<int>
{
    [PrimaryKey]
    public int Id { get; set; }

    public string Name { get; set; }

    public decimal Price { get; set; }
}
```

### Understanding DataProviderBase

`DataProviderBase` is an abstract class that defines the contract for all database providers. Concrete implementations (such as `SqlDataProvider` for PostgreSQL or `MongoDataProvider` for MongoDB) inherit from this class and provide database-specific functionality.

The base class provides:
- CRUD operations that work with any database
- Automatic SQL generation for common operations
- Parameter handling to prevent SQL injection
- Entity mapping through attributes
- Async variants for all operations

### Basic CRUD Operations

```csharp
// Get a single entity by primary key
User user = dataProvider.Get<User>(123);

// Get all entities
IEnumerable<User> allUsers = dataProvider.GetAll<User>();

// Get with where clause
var whereParams = new Dictionary<string, object>
{
    ["Email"] = "user@example.com"
};
User user = dataProvider.Get<User>(whereParams);

// Insert/Update (Save handles both)
var newUser = new User
{
    FirstName = "John",
    LastName = "Doe",
    Email = "john.doe@example.com"
};
object insertedId = dataProvider.Save(newUser);

// Update existing entity
user.FirstName = "Jane";
dataProvider.Update(user);

// Delete by primary key
int deleted = dataProvider.Delete<User>(123);
```

### Async Operations

```csharp
// Async get by primary key
User user = await dataProvider.GetAsync<User>(123);

// Async get all
IEnumerable<User> users = await dataProvider.GetAllAsync<User>();

// Async save
object insertedId = await dataProvider.SaveAsync(newUser);

// Async update
int affected = await dataProvider.UpdateAsync(user);

// Async delete
int deleted = await dataProvider.DeleteAsync<User>(123);
```

### Pagination

```csharp
using Nautilus.Base.Data.Provider;

// Create pagination options (page 2, 20 items per page)
var options = new PaginationOptions(page: 2, pageSize: 20);

// Execute paginated query
IEnumerable<User> page = dataProvider.ExecuteQuery<User>(options);

// Async variant
IEnumerable<User> page = await dataProvider.ExecuteQueryAsync<User>(options);
```

### Configuration Loading

```csharp
using Nautilus.Base.Configuration;
using Microsoft.Extensions.Configuration;

// Load configuration from appsettings.json
IConfigurationRoot config = AppSettingJsonConfiguration.GetConfiguration("appsettings.json");

// Bind to AppSettings model
var appSettings = new AppSettings();
config.Bind(appSettings);

// Access database provider settings
var providerSetting = appSettings.DatabaseProviders["DefaultConnection"];
var connectionString = providerSetting.ConnectionString;
var providerType = providerSetting.ProviderType; // SqlServer, MySql, Sqlite, PostgreSql, Mongo
```

### appsettings.json Structure

```json
{
  "DatabaseProviders": {
    "DefaultConnection": {
      "Key": "DefaultConnection",
      "ProviderType": "PostgreSql",
      "ConnectionString": "Host=localhost;Port=5432;Database=mydb;Username=user;Password=pass",
      "Active": true,
      "Schema": "public",
      "UseExistingDatabase": false
    }
  },
  "Security": {
    "JwtSecret": "your-secret-key",
    "TokenExpirationMinutes": 60
  },
  "WebApi": {
    "CorsAllowedOrigins": ["http://localhost:3000"]
  }
}
```

## API Reference

### DataProviderBase

Abstract base class for all data provider implementations.

#### Core Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `Get<T>(primaryKey)` | `T` | Gets a single entity by primary key |
| `Get<T>(where)` | `T` | Gets a single entity matching the where clause |
| `GetAll<T>()` | `IEnumerable<T>` | Gets all entities of type T |
| `GetAll<T>(where)` | `IEnumerable<T>` | Gets all entities matching the where clause |
| `Save(model)` | `object` | Inserts or updates an entity |
| `Update(model)` | `int` | Updates an existing entity (returns affected rows) |
| `Delete<T>(primaryKey)` | `int` | Deletes an entity by primary key |
| `Delete<T>(where)` | `int` | Deletes entities matching the where clause |
| `GetRecordsCount<T>()` | `long` | Gets the total record count for type T |

#### Async Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `GetAsync<T>(primaryKey)` | `Task<T>` | Async variant of Get by primary key |
| `GetAsync<T>(where)` | `Task<T>` | Async variant of Get with where clause |
| `GetAllAsync<T>()` | `Task<IEnumerable<T>>` | Async variant of GetAll |
| `GetAllAsync<T>(where)` | `Task<IEnumerable<T>>` | Async variant of GetAll with where |
| `SaveAsync(model)` | `Task<object>` | Async variant of Save |
| `UpdateAsync(model)` | `Task<int>` | Async variant of Update |
| `DeleteAsync<T>(primaryKey)` | `Task<int>` | Async variant of Delete |
| `DeleteAsync<T>(where)` | `Task<int>` | Async variant of Delete with where |
| `GetRecordsCountAsync<T>()` | `Task<long>` | Async variant of GetRecordsCount |

#### Query Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `ExecuteQuery<T>(options)` | `IEnumerable<T>` | Executes a paginated query |
| `ExecuteQueryAsync<T>(options)` | `Task<IEnumerable<T>>` | Async paginated query |
| `ExecuteQuery(sql)` | `Dictionary<string, object>` | Executes raw SQL query |
| `ExecuteQueryAsync(sql)` | `Task<Dictionary<string, object>>` | Async raw SQL query |
| `ExecuteNonQuery(sql)` | `int` | Executes a non-query SQL statement |
| `ExecuteNonQueryAsync(sql)` | `Task<int>` | Async non-query SQL statement |
| `ExecuteReader(sql)` | `IEnumerable<Dictionary<string, object>>` | Executes SQL and returns multiple rows |
| `ExecuteReaderAsync(sql)` | `Task<Dictionary<string, object>>` | Executes SQL and returns single row |

#### Stored Procedures

| Method | Returns | Description |
|--------|---------|-------------|
| `ExecuteStoredProcedure<T>(spName, parameters)` | `T` | Executes a stored procedure returning single result |
| `ExecuteStoredProcedureList<T>(spName, parameters)` | `IEnumerable<T>` | Executes a stored procedure returning list |
| `ExecuteStoredProcedureList<T1, T2>(...)` | `(IEnumerable<T1>, T2)` | Executes SP returning two result sets |
| `ExecuteStoredProcedureList<T1, T2, T3>(...)` | `(IEnumerable<T1>, T2, T3)` | Executes SP returning three result sets |

### Entity Attributes

#### [Table]

Maps a class to a database table.

```csharp
[Table("table_name")]
[Table("table_name", Schema = "schema_name")]
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `tableName` | `string` | The database table name |
| `Schema` | `string` | Optional schema name (for PostgreSQL, SQL Server) |

#### [PrimaryKey]

Marks a property as the primary key.

```csharp
[PrimaryKey]
[PrimaryKey(AutoGenerateGuid = true)]
[PrimaryKey(Name = "custom_id")]
```

| Property | Type | Default | Description |
|-----------|------|---------|-------------|
| `AutoGenerateGuid` | `bool` | `false` | Whether to auto-generate a GUID |
| `Name` | `string` | `null` | Custom column name for the primary key |

#### [Column]

Maps a property to a database column.

```csharp
[Column("column_name")]
[Column(typeof(Order), nameof(Order.Id), "OrderId")]
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `columnName` | `string` | The database column name |
| `tableType` | `Type` | Type representing the referenced table (for foreign keys) |
| `targetColumn` | `string` | Target column in referenced table |
| `ColumnName` | `string` | Column name to use in current table |

### Configuration Classes

#### DatabaseProviderSetting

Configuration for a database connection.

| Property | Type | Description |
|-----------|------|-------------|
| `Key` | `string` | Unique identifier for the database |
| `ProviderType` | `DbProviderType` | Database type (SqlServer, MySql, Sqlite, PostgreSql, Mongo) |
| `ConnectionString` | `string` | Database connection string |
| `Active` | `bool` | Whether this connection is active |
| `IdentityDb` | `bool` | Whether this is the identity database |
| `Schema` | `string` | Database schema (default: "public" for PostgreSQL) |
| `UseExistingDatabase` | `bool` | Skip database creation if true |
| `CreateSampleData` | `bool` | Create sample data on initialization |

#### AppSettingJsonConfiguration

Static utility for loading configuration.

| Method | Returns | Description |
|--------|---------|-------------|
| `GetConfiguration(jsonFileName)` | `IConfigurationRoot` | Loads configuration from JSON file |

## Usage Patterns

### Entity Mapping with Different Attribute Combinations

```csharp
// Simple mapping - uses class name for table
public class Category : EntityBase<int>
{
    [PrimaryKey]
    public int Id { get; set; }
    public string Name { get; set; }
}
// Maps to table: Category

// Custom table name
[Table("categories")]
public class Category : EntityBase<int>
{
    [PrimaryKey]
    public int Id { get; set; }
    public string Name { get; set; }
}

// Schema + custom table name
[Table("categories", Schema = "catalog")]
public class Category : EntityBase<int>
{
    [PrimaryKey]
    public int Id { get; set; }
    public string Name { get; set; }
}
// Maps to: catalog.categories

// Column name overrides
[Table("users")]
public class User : EntityBase<int>
{
    [PrimaryKey]
    [Column("user_id")]
    public int Id { get; set; }

    [Column("full_name")]
    public string FullName { get; set; }

    // Uses property name "Email" as column name
    public string Email { get; set; }
}
```

### Advanced Query Patterns

```csharp
// Complex where clause
var whereParams = new Dictionary<string, object>
{
    ["Status"] = "Active",
    ["CreatedAt"] = DateTime.Today.AddDays(-30)
};
IEnumerable<Order> orders = dataProvider.GetAll<Order>(whereParams);

// Raw SQL execution
string sql = "SELECT * FROM users WHERE registration_date > @cutoffDate";
var parameters = new Dictionary<string, object>
{
    ["cutoffDate"] = new DateTime(2024, 1, 1)
};
var results = dataProvider.ExecuteQuery(sql, parameters);

// Execute raw SQL with no results
string updateSql = "UPDATE products SET price = price * 1.1 WHERE category = @category";
var parameters = new Dictionary<string, object> { ["category"] = "Electronics" };
int affected = dataProvider.ExecuteNonQuery(updateSql, parameters);
```

### Bulk Operations

```csharp
// Save multiple entities
var users = new List<User>
{
    new User { FirstName = "Alice", Email = "alice@example.com" },
    new User { FirstName = "Bob", Email = "bob@example.com" },
    new User { FirstName = "Charlie", Email = "charlie@example.com" }
};

// Save all (loops internally)
dataProvider.SaveAll(users);

// Async bulk save
await dataProvider.SaveAllAsync(users);

// Bulk insert (provider-specific implementation)
dataProvider.SaveBulk(users);
await dataProvider.SaveBulkAsync(users);
```

### Working with Multiple Result Sets

```csharp
// Stored procedure returning multiple result sets
var parameters = new Dictionary<string, object>
{
    ["UserId"] = 123
};

// Returns two result sets: (orders, summary)
var (orders, summary) = dataProvider.ExecuteStoredProcedureList<Order, OrderSummary>(
    "GetUserOrderSummary",
    parameters
);

// Returns three result sets
var (orders, items, totals) = dataProvider.ExecuteStoredProcedureList<Order, OrderItem, OrderTotal>(
    "GetUserOrderDetails",
    parameters
);
```

### Get Total Record Count

```csharp
// Get total count (sync)
long totalUsers = dataProvider.GetRecordsCount<User>();

// Get total count (async)
long totalProducts = await dataProvider.GetRecordsCountAsync<Product>();

// Use with pagination
var options = new PaginationOptions(page: 1, pageSize: 20);
var users = dataProvider.ExecuteQuery<User>(options);
long totalRecords = dataProvider.GetRecordsCount<User>();
int totalPages = (int)Math.Ceiling((double)totalRecords / options.PageSize);
```

## Architecture

Nautilus.Base follows a **provider pattern** that separates the abstraction from the implementation. This design allows your application code to remain database-agnostic while letting you swap database providers as needed.

### Provider Pattern Overview

```
                              Application Layer
              (Uses DataProviderBase - database-agnostic code)

                                       inherits

                             DataProviderBase
        (Abstract base class with CRUD, Query, Pagination)

                    implements                        implements
                    ────────────────                 ─────────────────

    SqlDataProvider          MongoDataProvider        Future Providers
    (PostgreSQL)             (MongoDB)                 (MySQL, etc.)

         │                        │                         │
         ▼                        ▼                         ▼

  Separate Packages:      nautilus-dataprovider-*     Future Packages
  - postgresql            - mongodb                  - mysql
  - mssql                 - cosmos                   - oracle
```

### Key Design Principles

1. **Abstraction First**: `DataProviderBase` provides the complete interface, keeping application code database-agnostic
2. **Concrete Providers Separate**: Database-specific implementations are in separate NuGet packages
3. **Attribute-Based Mapping**: Entities use attributes for declarative database mapping
4. **Security by Default**: SQL identifier validation prevents injection attacks
5. **Reflection Caching**: Property metadata is cached for performance
6. **Async-First**: All operations have async variants for modern .NET applications

### Project Structure

```
src/Nautilus.Base/
├── Data/
│   ├── Attributes/              # Entity mapping attributes
│   │   ├── TableAttribute.cs
│   │   ├── PrimaryKeyAttribute.cs
│   │   ├── ColumnAttribute.cs
│   │   ├── ForeignKeyAttribute.cs
│   │   └── ...
│   ├── Entity/
│   │   └── EntityBase.cs       # Base entity class
│   ├── Provider/
│   │   ├── Base/
│   │   │   ├── DataProviderBase.cs    # Abstract provider
│   │   │   ├── DataConnection.cs      # Connection management
│   │   │   └── PaginationOptions.cs   # Pagination configuration
│   │   ├── Reflection/
│   │   │   └── PropertyInfoCache.cs   # Reflection caching
│   │   └── Security/
│   │       └── SqlIdentifierValidator.cs  # SQL injection protection
│   └── DbProviderType.cs         # Database type enum
├── Configuration/
│   ├── Models/
│   │   ├── AppSettings.cs               # Root settings model
│   │   ├── AppSettingBase.cs            # Base settings
│   │   ├── DatabaseProviderSetting.cs   # DB configuration
│   │   ├── SecuritySetting.cs           # Security settings
│   │   └── ...
│   └── AppSettingJsonConfiguration.cs   # JSON configuration loader
└── Diagnostics/                 # Diagnostic utilities
```

### How Concrete Providers Work

Concrete data provider implementations are distributed as separate packages following the naming convention `Nautilus.DataProvider.[DatabaseName]`. Each package:

1. **Inherits from DataProviderBase**: Implements the abstract methods
2. **Provides Database-Specific Logic**: Handles connection creation, parameter formatting, and SQL dialect differences
3. **Implements Abstract Methods**:
   - `CreateConnection()`: Creates a database-specific connection
   - `CreateDatabase()`: Creates the database if it doesn't exist
   - `CreateDbParameter()`: Creates database-specific parameters
   - `Save()`, `SaveAsync()`: Implements insert logic
   - `SaveBulk()`, `SaveBulkAsync()`: Implements bulk insert

Example provider packages:
- `Nautilus.DataProvider.PostgreSQL` - PostgreSQL support
- `Nautilus.DataProvider.MongoDB` - MongoDB support
- `Nautilus.DataProvider.MSSQL` - Microsoft SQL Server support

## Security

Nautilus.Base includes built-in protection against SQL injection attacks through the `SqlIdentifierValidator` class.

### SQL Injection Prevention

All SQL identifiers (table names, column names, schema names) are automatically validated before being used in SQL queries:

```csharp
// Automatically validated
[Table("users")]  // "users" is validated
public class User { }

[Column("user_id")]  // "user_id" is validated
public int Id { get; set; }
```

The validator checks for:
- **SQL Keywords**: Prevents use of reserved words (SELECT, DROP, etc.)
- **Comment Markers**: Blocks `--`, `/*`, `*/`
- **Semicolons**: Prevents statement injection
- **Quotes**: Blocks single and double quotes
- **Invalid Characters**: Only allows alphanumeric, underscore, dots, @, #, $

If validation fails, a `NautilusException` is thrown with a descriptive message:

```
Error: Column name 'SELECT' contains a reserved SQL keyword.
Use the [Column] attribute to map to a safe database column name.
```

### Parameterized Queries

All values are passed as parameters, never concatenated into SQL:

```csharp
// Safe - values are parameterized
var whereParams = new Dictionary<string, object>
{
    ["Email"] = "user'; DROP TABLE users; --"
};
var user = dataProvider.Get<User>(whereParams);
```

## Related Repositories

Nautilus.Base is the foundation for database access in the Nautilus ecosystem. Concrete provider implementations are available in separate packages:

| Package | Description | Link |
|---------|-------------|------|
| `Nautilus.DataProvider.PostgreSQL` | PostgreSQL provider implementation | [GitHub](https://github.com/synthphonic/nautilus-dataprovider-postgresql) |
| `Nautilus.DataProvider.MongoDB` | MongoDB provider implementation | [GitHub](https://github.com/synthphonic/nautilus-dataprovider-mongodb) |
| `Nautilus.DataProvider.MSSQL` | SQL Server provider implementation | [GitHub](https://github.com/synthphonic/nautilus-dataprovider-mssql) |

## License

This project is licensed under the [Business Source License 1.1](LICENSE).

The license allows free commercial use while protecting the source code. The code will automatically convert to the MIT License on **[Change Date]**, 3 years after the first publication.

## Contributing

Contributions are welcome! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## Support

- **Issues**: [GitHub Issues](https://github.com/synthphonic/nautilus-base/issues)
- **Discussions**: [GitHub Discussions](https://github.com/synthphonic/nautilus-base/discussions)

## Acknowledgments

Nautilus.Base is part of the [Nautilus](https://github.com/synthphonic) ecosystem - a collection of .NET libraries designed to simplify common development tasks. The provider pattern is inspired by the need for database-agnostic data access in modern applications.

Related Nautilus packages:
- [Nautilus.FluentResult](https://github.com/synthphonic/nautilus-fluentresult) - Type-safe result pattern
- [Nautilus.HttpClient](https://github.com/synthphonic/nautilus-httpclient) - Fluent HTTP client
- [Nautilus.HandlerPattern](https://github.com/synthphonic/nautilus-handlerpattern) - Request/response pattern with pipelines

---

**Note**: This library is in active development. API changes may occur before version 1.0.0.
