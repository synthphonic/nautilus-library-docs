# Nautilus.DataProvider.PostgreSql

[![Build Status](https://github.com/synthphonic/nautilus-dataprovider-postgresql/actions/workflows/ci.yml/badge.svg)](https://github.com/synthphonic/nautilus-dataprovider-postgresql/actions/workflows/ci.yml)
[![NuGet Version](https://img.shields.io/nuget/v/Nautilus.DataProvider.PostgreSql)](https://www.nuget.org/packages/Nautilus.DataProvider.PostgreSql/)
[![License: BSL-1.1](https://img.shields.io/badge/License-BSL--1.1-blue.svg)](LICENSE)

A PostgreSQL data provider implementation for the Nautilus ecosystem. This library provides a concrete implementation of `DataProviderBase` from `Nautilus.Base`, enabling seamless database operations with PostgreSQL through a consistent, database-agnostic API.

## Features

- **PostgreSQL-Specific Implementation**: Full support for PostgreSQL including schemas, UUIDs, and native data types
- **Automatic Migrations**: `PostgreSqlMigrator` for creating tables with foreign key dependency resolution
- **Type Mapping**: Automatic .NET to PostgreSQL type mapping via `PostgreSqlDataTypeMapper`
- **CRUD Operations**: Complete Create, Read, Update, Delete support (inherited from DataProviderBase)
- **Async/Await**: All operations have async variants
- **Schema Support**: Configurable schema (defaults to "public")
- **Parameterized Queries**: SQL injection protection through parameterized commands
- **Stored Procedures**: Full support for PostgreSQL functions/procedures with multiple result sets
- **Dapper Integration**: Leverages Dapper for efficient micro-ORM operations

## Installation

Install the package via [NuGet](https://www.nuget.org/packages/Nautilus.DataProvider.PostgreSql/):

```bash
dotnet add package Nautilus.DataProvider.PostgreSql
```

Or via the Package Manager Console:

```powershell
Install-Package Nautilus.DataProvider.PostgreSql
```

## Quick Start

### Define an Entity with Attributes

```csharp
using Nautilus.Base.Data.ModelBase;
using Nautilus.Base.Data.Attributes;

// Basic entity with table and column mapping
[NautilusTableAttribute("users")]
public class User : EntityBase<int>
{
    [NautilusPrimaryKeyAttribute]
    [NautilusAutoIncrementAttribute]
    public int Id { get; set; }

    [NautilusColumnAttribute("first_name")]
    public string FirstName { get; set; }

    [NautilusColumnAttribute("last_name")]
    public string LastName { get; set; }

    public string Email { get; set; }
}

// Entity with schema and foreign key
[NautilusTableAttribute("orders", Schema = "sales")]
public class Order : EntityBase<int>
{
    [NautilusPrimaryKeyAttribute]
    [NautilusAutoIncrementAttribute]
    public int Id { get; set; }

    public string OrderNumber { get; set; }

    public decimal TotalAmount { get; set; }

    [NautilusForeignKeyAttribute]
    [ConstraintReferenceForeignKey(typeof(User), "Id", true, true)]
    public int UserId { get; set; }
}
```

### Initialize the PostgreSQL Database

```csharp
using Nautilus.DataProvider.PostgreSql;
using Nautilus.Base.Configuration;

// Load configuration from appsettings.json
IConfigurationRoot config = AppSettingJsonConfiguration.GetConfiguration("appsettings.json");
var appSettings = new AppSettings();
config.Bind(appSettings);

var providerSetting = appSettings.DatabaseProviders["DefaultConnection"];

// Create the data provider
var dataProvider = new PostgreSqlDatabase(providerSetting);
```

### Basic CRUD Operations

```csharp
// Insert a new user
var newUser = new User
{
    FirstName = "John",
    LastName = "Doe",
    Email = "john.doe@example.com"
};
object insertedId = dataProvider.Save(newUser);

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

// Update existing entity
user.FirstName = "Jane";
int affected = dataProvider.Update(user);

// Delete by primary key
int deleted = dataProvider.Delete<User>(123);
```

### Async Operations

```csharp
// Async insert
object insertedId = await dataProvider.SaveAsync(newUser);

// Async get by primary key
User user = await dataProvider.GetAsync<User>(123);

// Async get all
IEnumerable<User> users = await dataProvider.GetAllAsync<User>();

// Async update
int affected = await dataProvider.UpdateAsync(user);

// Async delete
int deleted = await dataProvider.DeleteAsync<User>(123);
```

### Bulk Operations

```csharp
// Bulk insert/update multiple entities efficiently
var users = new List<User>
{
    new User { FirstName = "John", LastName = "Doe", Email = "john@example.com" },
    new User { FirstName = "Jane", LastName = "Smith", Email = "jane@example.com" },
    new User { FirstName = "Bob", LastName = "Johnson", Email = "bob@example.com" }
};
IEnumerable<object> insertedIds = dataProvider.SaveBulk(users);

// Async bulk operations
IEnumerable<object> insertedIds = await dataProvider.SaveBulkAsync(users);
```

### Database Management

```csharp
// Check if database exists
bool exists = dataProvider.DatabaseExists();

// Create database (if not exists)
dataProvider.CreateDatabase();

// Check database connection
bool isConnected = dataProvider.CheckConnection();
```

### Database Migration

```csharp
using Nautilus.DataProvider.PostgreSql;

// Create a migrator
var migrator = new PostgreSqlMigrator(
    providerSetting,
    () => dataProvider.CreateConnection(),
    () => dataProvider.CreateDatabase()
);

// Add tables for migration
migrator.AddTable<User>()
        .AddTable<Order>()
        .AddTable<Product>();

// Run the migration (creates tables with FK dependency resolution)
migrator.Run();
```

## API Reference

### PostgreSqlDatabase

Concrete implementation of `DataProviderBase` for PostgreSQL.

#### Constructors

| Constructor | Description |
|-------------|-------------|
| `PostgreSqlDatabase(DatabaseProviderSetting providerSetting)` | Creates a new instance with full configuration |
| `PostgreSqlDatabase(string key, string connectionString)` | Creates a new instance with simple connection string |

#### Properties

| Property | Type | Description |
|----------|------|-------------|
| `Schema` | `string` | PostgreSQL schema name (defaults to "public") |

#### Key Methods (inherited from DataProviderBase)

| Method | Returns | Description |
|--------|---------|-------------|
| `Get<T>(primaryKey)` | `T` | Gets a single entity by primary key |
| `Get<T>(where)` | `T` | Gets a single entity matching the where clause |
| `GetAll<T>()` | `IEnumerable<T>` | Gets all entities of type T |
| `GetAll<T>(where)` | `IEnumerable<T>` | Gets all entities matching the where clause |
| `Save(model)` | `object` | Inserts or updates an entity |
| `SaveBulk(models)` | `IEnumerable<object>` | Bulk insert/update multiple entities |
| `Update(model)` | `int` | Updates an existing entity (returns affected rows) |
| `Delete<T>(primaryKey)` | `int` | Deletes an entity by primary key |
| `GetRecordsCount<T>()` | `long` | Gets the total record count for type T |

#### Async Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `GetAsync<T>(primaryKey)` | `Task<T>` | Async variant of Get by primary key |
| `GetAsync<T>(where)` | `Task<T>` | Async variant of Get with where clause |
| `GetAllAsync<T>()` | `Task<IEnumerable<T>>` | Async variant of GetAll |
| `SaveAsync(model)` | `Task<object>` | Async variant of Save |
| `SaveBulkAsync(models)` | `Task<IEnumerable<object>>` | Async variant of SaveBulk |
| `UpdateAsync(model)` | `Task<int>` | Async variant of Update |
| `DeleteAsync<T>(primaryKey)` | `Task<int>` | Async variant of Delete |
| `GetRecordsCountAsync<T>()` | `Task<long>` | Async variant of GetRecordsCount |

#### Database Management Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `DatabaseExists()` | `bool` | Checks if the database exists |
| `CreateDatabase()` | `void` | Creates the database if it doesn't exist |
| `CheckConnection()` | `bool` | Checks if the database connection is valid |
| `CreateConnection()` | `DbConnection` | Creates and opens a new PostgreSQL connection (overrides base) |
| `GetAllDatabaseNameAsync()` | `Task<IList<string>>` | Gets all database names from PostgreSQL asynchronously |

#### Database Introspection Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `GetTables()` | `IEnumerable<TableInfo>` | Gets table information (schema, name, owner) for current schema |
| `GetTablesAsync()` | `Task<IEnumerable<TableInfo>>` | Async variant of GetTables |

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

#### Stored Procedures

| Method | Returns | Description |
|--------|---------|-------------|
| `ExecuteStoredProcedure<T>(spName, parameters)` | `T` | Executes a stored procedure returning single result |
| `ExecuteStoredProcedureList<T>(spName, parameters)` | `IEnumerable<T>` | Executes a stored procedure returning list |
| `ExecuteStoredProcedureList<T1, T2>(...)` | `(IEnumerable<T1>, T2)` | Executes SP returning two result sets |
| `ExecuteStoredProcedureList<T1, T2, T3>(...)` | `(IEnumerable<T1>, T2, T3)` | Executes SP returning three result sets |

#### Utility Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `CreateDbParameter(name, value)` | `DbParameter` | Creates a PostgreSQL parameter with automatic type conversion (enums → int) |

### PostgreSqlMigrator

Database migration tool for creating PostgreSQL tables.

#### Constructor

| Parameter | Type | Description |
|-----------|------|-------------|
| `databaseProviderSetting` | `DatabaseProviderSetting` | Database configuration |
| `createConnectionFunc` | `Func<DbConnection>` | Function to create database connections |
| `createDatabaseFunc` | `Action` | Optional action to create the database |

#### Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `AddTable<TModel>()` | `IDbMigrator` | Adds a model type for migration (generic) |
| `AddTable(Type modelType)` | `IDbMigrator` | Adds a model type for migration (reflection) |
| `Run()` | `void` | Executes the migration |

The migrator automatically:
- Resolves foreign key dependencies before creating tables
- Creates tables with the correct PostgreSQL data types
- Handles primary keys, auto-increment, and constraints
- Supports ON DELETE CASCADE and ON UPDATE CASCADE

### PostgreSqlDataTypeMapper

Maps .NET types to PostgreSQL data types.

#### Properties

| Property | Type | Description |
|----------|------|-------------|
| `SimpleTypeMappings` | `Dictionary<string, string>` (static) | Maps .NET type full names to PostgreSQL type names for types not requiring custom attributes |

#### Type Mappings

| .NET Type | PostgreSQL Type | Notes |
|-----------|----------------|-------|
| `Int16` | `smallint` | 2-byte integer |
| `Int32` | `integer` | 4-byte integer |
| `Int64` | `bigint` | 8-byte integer |
| `String` | `text` or `varchar(n)` | Use `[NautilusVarcharAttribute]` for varchar |
| `Guid` | `uuid` | UUID with optional auto-generation |
| `Decimal` | `decimal` or `money` | Use `[NautilusDecimalAttribute]` to specify |
| `Double` | `float8` | Double precision |
| `DateTime` | `time` | Time without time zone |
| `DateTimeOffset` | `timestamptz` | Time with time zone |
| `Boolean` | `bool` | Boolean |
| `Byte[]` | `bytea` | Binary data |
| `DateOnly` | `date` | Date without time |
| `Enum` | `integer` | Stored as integer |

#### Simple Type Mappings Reference

| .NET Type | PostgreSQL Type |
|-----------|-----------------|
| `System.Int16` | `smallint` |
| `System.Int64` | `bigint` |
| `System.Guid` | `uuid` |
| `System.Double` | `float8` |
| `System.DateTime` | `time` |
| `System.DateTimeOffset` | `timestamptz` |
| `System.Boolean` | `bool` |
| `System.DateOnly` | `date` |

## Usage Patterns

### Entity Mapping with Attributes

```csharp
// Simple mapping - uses class name for table
public class Category : EntityBase<int>
{
    [NautilusPrimaryKeyAttribute]
    public int Id { get; set; }
    public string Name { get; set; }
}
// Maps to table: Category

// Custom table name with schema
[NautilusTableAttribute("categories", Schema = "inventory")]
public class Category : EntityBase<int>
{
    [NautilusPrimaryKeyAttribute]
    public int Id { get; set; }
    public string Name { get; set; }
}
// Maps to: inventory.categories

// Column name overrides with data type attributes
[NautilusTableAttribute("users")]
public class User : EntityBase<int>
{
    [NautilusPrimaryKeyAttribute]
    [NautilusAutoIncrementAttribute]
    [NautilusColumnAttribute("user_id")]
    public int Id { get; set; }

    [NautilusVarcharAttribute(100)]
    [NautilusColumnAttribute("full_name")]
    public string FullName { get; set; }

    [NautilusNotNullAttribute]
    public string Email { get; set; }
}
```

### Migration with Foreign Keys

```csharp
// Define related entities
[NautilusTableAttribute("authors")]
public class Author : EntityBase<int>
{
    [NautilusPrimaryKeyAttribute]
    [NautilusAutoIncrementAttribute]
    public int Id { get; set; }

    [NautilusVarcharAttribute(200)]
    public string Name { get; set; }
}

[NautilusTableAttribute("books")]
public class Book : EntityBase<int>
{
    [NautilusPrimaryKeyAttribute]
    [NautilusAutoIncrementAttribute]
    public int Id { get; set; }

    [NautilusVarcharAttribute(200)]
    public string Title { get; set; }

    // Foreign key with cascade actions
    [NautilusForeignKeyAttribute]
    [ConstraintReferenceForeignKey(typeof(Author), "Id", OnDeleteCascade: true, OnUpdateCascade: true)]
    public int AuthorId { get; set; }
}

// Run migration - tables are created in dependency order
var migrator = new PostgreSqlMigrator(providerSetting, () => connection, () => database.CreateDatabase());
migrator.AddTable<Author>()
        .AddTable<Book>()
        .Run();
```

### Schema Configuration

```csharp
// Using appsettings.json
{
  "DatabaseProviders": {
    "DefaultConnection": {
      "Key": "DefaultConnection",
      "ProviderType": "PostgreSql",
      "ConnectionString": "Host=localhost;Port=5432;Database=mydb;Username=user;Password=pass",
      "Schema": "myschema",
      "Active": true,
      "UseExistingDatabase": false
    }
  }
}

// The provider will use the configured schema
var provider = new PostgreSqlDatabase(providerSetting);
// provider.Schema == "myschema"

// Or set schema directly
provider.Schema = "public";
```

### Raw SQL Execution

```csharp
// Execute raw SQL with parameters
string sql = "SELECT * FROM users WHERE created_at > @cutoffDate";
var parameters = new Dictionary<string, object>
{
    ["cutoffDate"] = new DateTime(2024, 1, 1)
};
var results = dataProvider.ExecuteQuery(sql, parameters);

// Execute non-query SQL
string updateSql = "UPDATE products SET price = price * 1.1 WHERE category = @category";
var parameters = new Dictionary<string, object> { ["category"] = "Electronics" };
int affected = dataProvider.ExecuteNonQuery(updateSql, parameters);
```

### Stored Procedure Execution

```csharp
// Execute stored procedure returning single result
var parameters = new Dictionary<string, object>
{
    ["UserId"] = 123
};
User user = dataProvider.ExecuteStoredProcedure<User>("get_user_by_id", parameters);

// Execute stored procedure returning list
IEnumerable<Order> orders = dataProvider.ExecuteStoredProcedureList<Order>(
    "get_user_orders",
    parameters
);

// Execute stored procedure returning multiple result sets
var (ordersList, summary) = dataProvider.ExecuteStoredProcedureList<Order, OrderSummary>(
    "get_user_order_summary",
    parameters
);
```

### Working with UUID Primary Keys

```csharp
[NautilusTableAttribute("products")]
public class Product : EntityBase<Guid>
{
    [NautilusPrimaryKeyAttribute(AutoGenerateGuid = true)]
    public Guid Id { get; set; }

    [NautilusVarcharAttribute(200)]
    public string Name { get; set; }

    public decimal Price { get; set; }
}

// On insert, PostgreSQL's gen_random_uuid() is used
var product = new Product
{
    Name = "New Product",
    Price = 29.99m
};
dataProvider.Save(product);
// product.Id is now set to the generated UUID
```

### Database Introspection

```csharp
// Get all tables in the current schema
IEnumerable<TableInfo> tables = dataProvider.GetTables();
foreach (var table in tables)
{
    Console.WriteLine($"Schema: {table.SchemaName}, Table: {table.TableName}, Owner: {table.TableOwner}");
}

// Async variant
IEnumerable<TableInfo> tables = await dataProvider.GetTablesAsync();

// Get all database names
IList<string> databases = await dataProvider.GetAllDatabaseNameAsync();
foreach (var db in databases)
{
    Console.WriteLine($"Database: {db}");
}
```

## Architecture

Nautilus.DataProvider.PostgreSql follows the **Provider Pattern** established by `Nautilus.Base`, providing a concrete PostgreSQL implementation while maintaining a consistent API.

```
                         Application Layer
           (Uses DataProviderBase - database-agnostic code)

                                    inherits

                             DataProviderBase
      (Abstract base class with CRUD, Query, Pagination from Nautilus.Base)

                              implements

                     PostgreSqlDatabase (This Library)
        - CreateConnection() -> NpgsqlConnection
        - CreateDatabase() -> PostgreSQL-specific database creation
        - Save/SaveAsync() -> PostgreSQL INSERT with RETURNING clause
        - UpdateAsync() -> PostgreSQL UPDATE with lowercase column names
        - CreateDbParameter() -> NpgsqlParameter
```

### Key Design Principles

1. **PostgreSQL-Specific**: Leverages Npgsql and PostgreSQL features (UUIDs, schemas, RETURNING clause)
2. **Dependency Resolution**: Migrator automatically resolves foreign key dependencies
3. **Type Mapping**: Automatic .NET to PostgreSQL type mapping
4. **SQL Injection Protection**: All values are parameterized
5. **Async-First**: All operations have async variants

### Project Structure

```
src/Nautilus.DataProvider.PostgreSql/
├── PostgreSqlDatabase.cs           # Main data provider implementation
├── PostgreSqlMigrator.cs           # Database migration tool
├── PostgreSqlDataTypeMapper.cs     # .NET to PostgreSQL type mapper
└── _globalusings.cs                # Global usings
```

## Security

Nautilus.DataProvider.PostgreSql includes built-in protection against SQL injection attacks:

### SQL Injection Prevention

All values are passed as parameters, never concatenated into SQL:

```csharp
// Safe - values are parameterized
var whereParams = new Dictionary<string, object>
{
    ["Email"] = "user'; DROP TABLE users; --"
};
var user = dataProvider.Get<User>(whereParams);
// The malicious input is treated as a literal string, not executable SQL
```

The library uses `NpgsqlParameter` for all parameter values, ensuring proper escaping and type handling.

## Related Repositories

Nautilus.DataProvider.PostgreSql is part of the Nautilus ecosystem:

| Package | Description | Link |
|---------|-------------|------|
| `Nautilus.Base` | Base abstractions and attributes for all data providers | [GitHub](https://github.com/synthphonic/nautilus-base) |

Other data provider implementations:
- `Nautilus.DataProvider.MSSQL` - Microsoft SQL Server support
- `Nautilus.DataProvider.MongoDB` - MongoDB support

## License

This project is licensed under the [Business Source License 1.1](LICENSE).

The license allows free commercial use while protecting the source code. The code will automatically convert to the MIT License on **[Change Date]**, 3 years after the first publication.

## Contributing

Contributions are welcome! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## Support

- **Issues**: [GitHub Issues](https://github.com/synthphonic/nautilus-dataprovider-postgresql/issues)
- **Discussions**: [GitHub Discussions](https://github.com/synthphonic/nautilus-dataprovider-postgresql/discussions)

## Acknowledgments

Nautilus.DataProvider.PostgreSql is part of the [Nautilus](https://github.com/synthphonic) ecosystem - a collection of .NET libraries designed to simplify common development tasks.

Related Nautilus packages:
- [Nautilus.Base](https://github.com/synthphonic/nautilus-base) - Foundation for data access abstraction
- [Nautilus.FluentResult](https://github.com/synthphonic/nautilus-fluentresult) - Type-safe result pattern
- [Nautilus.HttpClient](https://github.com/synthphonic/nautilus-httpclient) - Fluent HTTP client
- [Nautilus.HandlerPattern](https://github.com/synthphonic/nautilus-handlerpattern) - Request/response pattern with pipelines

Built with:
- [Npgsql](https://www.npgsql.org/) - PostgreSQL .NET data provider
- [Dapper](https://github.com/DapperLib/Dapper) - Micro-ORM for efficient data access

---

**Note**: This library is in active development. API changes may occur before version 1.0.0.
