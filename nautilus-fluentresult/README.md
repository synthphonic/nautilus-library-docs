# Nautilus.FluentResult

[![Build Status](https://github.com/shahzshafie/nautilus-fluentresult/actions/workflows/ci.yml/badge.svg)](https://github.com/shahzshafie/nautilus-fluentresult/actions/workflows/ci.yml)
[![NuGet Version](https://img.shields.io/nuget/v/Nautilus.FluentResult)](https://www.nuget.org/packages/Nautilus.FluentResult/)
[![License: BSL-1.1](https://img.shields.io/badge/License-BSL--1.1-blue.svg)](LICENSE)

A type-safe, fluent Result Pattern implementation for .NET applications. This library provides structured error handling that avoids exceptions for expected failures, making your code more predictable and easier to test.

## Features

- **Type-Safe**: Strongly-typed success/failure handling
- **Structured Errors**: Errors contain both code and description
- **Implicit Conversions**: Easy conversion between values and results
- **Null Safety**: Proper null handling for values and errors
- **.NET 9.0**: Built on the latest .NET platform
- **Zero Dependencies**: No external dependencies

## Installation

Install the package via [NuGet](https://www.nuget.org/packages/Nautilus.FluentResult/):

```bash
dotnet add package Nautilus.FluentResult
```

Or via the Package Manager Console:

```powershell
Install-Package Nautilus.FluentResult
```

## Quick Start

### Basic Usage

```csharp
using Nautilus.FluentResult;

// Create a success result
Result result = Result.Success();

if (result.IsSuccess)
{
    Console.WriteLine("Operation succeeded!");
}

// Create a failure result
Error error = new Error("VALIDATION_ERROR", "Invalid input");
Result failure = Result.Failure(error);

if (failure.IsFailure)
{
    Console.WriteLine($"Error: {failure.Error.Code} - {failure.Error.Description}");
}
```

### Working with Values

```csharp
// Create a successful result with a value
Result<User> userResult = Result<User>.Success(new User { Name = "John" });

if (userResult.IsSuccess)
{
    User user = userResult.Value;
    Console.WriteLine($"User: {user.Name}");
}

// Create a failure result
Error error = new Error("USER_NOT_FOUND", "User does not exist");
Result<User> notFound = Result<User>.Failure(error);

// Implicit conversions make it even easier
Result<User> success = new User { Name = "Jane" }; // Automatically wraps in Success
Result<User> failure = new Error("SERVER_ERROR", "Database unavailable"); // Automatically wraps in Failure
```

### Real-World Example

```csharp
public Result<User> GetUserById(int userId)
{
    if (userId <= 0)
    {
        return Result<User>.Failure(
            new Error("INVALID_ID", "User ID must be positive")
        );
    }

    User? user = _repository.Find(userId);

    if (user is null)
    {
        return Result<User>.Failure(
            new Error("USER_NOT_FOUND", $"User with ID {userId} not found")
        );
    }

    return user; // Implicit conversion to Result<User>.Success(user)
}

// Usage
Result<User> result = GetUserById(123);

if (result.IsFailure)
{
    Console.WriteLine($"Failed to get user: {result.Error.Description}");
    return;
}

User user = result.Value;
Console.WriteLine($"Found user: {user.Name}");
```

## API Reference

### Result

The base class for operation results.

#### Properties

| Property | Type | Description |
|----------|------|-------------|
| `IsSuccess` | `bool` | Indicates whether the operation was successful |
| `IsFailure` | `bool` | Indicates whether the operation failed |
| `Error` | `Error` | The error associated with a failed result |

#### Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `Success()` | `Result` | Creates a successful result |
| `Failure(Error error)` | `Result` | Creates a failed result with the specified error |

### Result\<T\>

A generic result that returns a value on success.

#### Properties

| Property | Type | Description |
|----------|------|-------------|
| `Value` | `T` | The value of the successful operation |
| *(Inherits all properties from `Result`)* | | |

#### Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `Success(T value)` | `Result<T>` | Creates a successful result with the specified value |
| `Failure(Error error)` | `Result<T>` | Creates a failed result with the specified error |

#### Implicit Conversions

- `T` → `Result<T>`: Creates a successful result containing the value
- `Error` → `Result<T>`: Creates a failed result containing the error

### Error

Represents structured error information.

#### Properties

| Property | Type | Description |
|----------|------|-------------|
| `Code` | `string` | The error code identifier |
| `Description` | `string` | Human-readable error description |
| `None` | `static Error` | Represents no error (success state) |

## Usage Patterns

### Chaining Operations

```csharp
Result<User> GetUser(int id) => /* ... */;
Result<Address> GetAddress(User user) => /* ... */;
Result<Location> GetLocation(Address address) => /* ... */

Result<Location> result = GetUser(userId)
    .Bind(user => GetAddress(user))
    .Bind(address => GetLocation(address));
```

### Matching Results

```csharp
string message = result.Match(
    onSuccess: value => $"Success: {value}",
    onFailure: error => $"Error: {error.Description}"
);
```

### Error Aggregation

```csharp
List<Error> errors = new();

// Collect errors from multiple operations
foreach (var item in items)
{
    Result<ProcessedItem> result = ProcessItem(item);
    if (result.IsFailure)
    {
        errors.Add(result.Error);
    }
}

if (errors.Count > 0)
{
    return Result.Failure(
        new Error("MULTIPLE_ERRORS", $"{errors.Count} items failed to process")
    );
}
```

## License

This project is licensed under the [Business Source License 1.1](LICENSE).

The license allows free commercial use while protecting the source code. The code will automatically convert to the MIT License on **[Change Date]**, 3 years after the first publication.

## Contributing

Contributions are welcome! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## Support

- **Issues**: [GitHub Issues](https://github.com/shahzshafie/nautilus-fluentresult/issues)
- **Discussions**: [GitHub Discussions](https://github.com/shahzshafie/nautilus-fluentresult/discussions)

## Acknowledgments

Inspired by the [Result Pattern](https://martinfowler.com/articles/replaceThrowWithNotification.html) popularized by Martin Fowler and the functional programming community.

---

**Note**: This library is in active development. API changes may occur before version 1.0.0.