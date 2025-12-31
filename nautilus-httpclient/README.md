# Nautilus.HttpClient

[![Build Status](https://github.com/synthphonic/nautilus-httpclient/actions/workflows/ci.yml/badge.svg)](https://github.com/synthphonic/nautilus-httpclient/actions/workflows/ci.yml)
[![NuGet Version](https://img.shields.io/nuget/vpre/Nautilus.HttpClient)](https://www.nuget.org/packages/Nautilus.HttpClient/)
[![License: BSL-1.1](https://img.shields.io/badge/License-BSL--1.1-blue.svg)](LICENSE)

A simple .NET 8 HttpClient wrapper that provides a fluent API for building HTTP requests. This library abstracts away the boilerplate of creating `HttpRequestMessage` objects with a clean, chainable interface.

## Features

- **Fluent API**: Chain methods together to build HTTP requests naturally
- **Type-Safe**: Interface-based design guides you through valid request building combinations
- **Zero Dependencies**: Pure .NET 8.0 - no external NuGet packages required
- **Multiple Body Types**: Support for form-url-encoded, multipart/form-data, and raw content
- **HTTP Methods**: GET, POST, PUT, DELETE
- **JWT Decoder**: Built-in utility for decoding JWT payloads
- **Async-First**: Fully async/await based API
- **.NET 8.0**: Built on the latest .NET platform with modern C# 12 features

## Installation

Install the package via [NuGet](https://www.nuget.org/packages/Nautilus.HttpClient/):

```bash
dotnet add package Nautilus.HttpClient
```

Or via the Package Manager Console:

```powershell
Install-Package Nautilus.HttpClient
```

## Quick Start

### Basic GET Request

```csharp
using System.Net.Http;
using Nautilus.HttpClient;

var httpClient = new HttpClient { BaseAddress = new Uri("https://api.example.com") };

var response = await NautilusHttpClient
    .Get(httpClient, "/users/123")
    .ExecuteAsync();

var content = await response.Content.ReadAsStringAsync();
```

### POST with Form-Url-Encoded Body

```csharp
var response = await NautilusHttpClient
    .Post(httpClient, "/token")
    .UseFormUrlEncodedBody()
    .AddParameter("grant_type", "password")
    .AddParameter("username", "user@example.com")
    .AddParameter("password", "secret")
    .ExecuteAsync();
```

### POST with JSON Body

```csharp
var jsonContent = JsonSerializer.Serialize(new
{
    name = "John Doe",
    email = "john@example.com"
});

var response = await NautilusHttpClient
    .Post(httpClient, "/users")
    .UseRawBody(jsonContent, "application/json")
    .EnsureStatusCode()  // Throws on non-2xx status
    .ExecuteAsync();
```

### POST with Multipart Form Data

```csharp
var response = await NautilusHttpClient
    .Post(httpClient, "/upload")
    .UseFormDataBody()
    .AddParameter("file", fileContent)
    .AddParameter("filename", "document.pdf")
    .AddParameter("description", "Annual report")
    .ExecuteAsync();
```

### Adding Custom Headers

```csharp
var response = await NautilusHttpClient
    .Get(httpClient, "/protected")
    .AddHeader("Authorization", "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...")
    .AddHeader("Accept", "application/json")
    .ExecuteAsync();
```

### PUT Request

```csharp
var jsonContent = JsonSerializer.Serialize(new { name = "Updated Name" });

var response = await NautilusHttpClient
    .Put(httpClient, "/users/123")
    .UseRawBody(jsonContent, "application/json")
    .ExecuteAsync();
```

### DELETE Request

```csharp
var response = await NautilusHttpClient
    .Delete(httpClient, "/users/123")
    .ExecuteAsync();
```

## JWT Decoder

The library includes a utility for decoding JWT payloads (no signature verification):

```csharp
using Nautilus.HttpClient.Jwt;

// Decode to dictionary
var payload = JWTDecoder.Decode(token);
var userId = payload["sub"];
var expiration = payload["exp"];

// Decode to JSON string
var json = JWTDecoder.DecodeAsJson(token);
```

## API Reference

### Static Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `Get(HttpClient, string)` | `IHeader` | Starts building a GET request |
| `Post(HttpClient, string)` | `IHeader` | Starts building a POST request |
| `Put(HttpClient, string)` | `IHeader` | Starts building a PUT request |
| `Delete(HttpClient, string)` | `IHeader` | Starts building a DELETE request |

### IHeader Interface

Entry point after HTTP method selection.

| Method | Returns | Description |
|--------|---------|-------------|
| `AddHeader(key, value)` | `IHeader` | Adds an HTTP header to the request |
| `UseFormUrlEncodedBody()` | `IFormUrlEncodedBody` | Selects form-url-encoded content type |
| `UseFormDataBody()` | `IFormDataBody` | Selects multipart/form-data content type |
| `UseRawBody(content, mediaType?)` | `IRawBody` | Sets raw string content with optional media type |
| `UseRawBody(content, encoding, mediaType)` | `IRawBody` | Sets raw string content with encoding |
| `EnsureStatusCode()` | `IExecute` | Enables automatic status code checking (throws on non-2xx) |
| `ExecuteAsync()` | `Task<HttpResponseMessage>` | Executes the HTTP request |

### IFormUrlEncodedBody Interface

For `application/x-www-form-urlencoded` content.

| Method | Returns | Description |
|--------|---------|-------------|
| `AddParameter(key, value)` | `IFormUrlEncodedBody` | Adds a form parameter |
| `EnsureStatusCode()` | `IExecute` | Enables automatic status code checking |
| `ExecuteAsync()` | `Task<HttpResponseMessage>` | Executes the HTTP request |

### IFormDataBody Interface

For `multipart/form-data` content.

| Method | Returns | Description |
|--------|---------|-------------|
| `AddParameter(key, value)` | `IFormDataBody` | Adds a form part |
| `EnsureStatusCode()` | `IExecute` | Enables automatic status code checking |
| `ExecuteAsync()` | `Task<HttpResponseMessage>` | Executes the HTTP request |

### IRawBody Interface

For raw string content (JSON, XML, text, etc.).

| Method | Returns | Description |
|--------|---------|-------------|
| `EnsureStatusCode()` | `IExecute` | Enables automatic status code checking |
| `ExecuteAsync()` | `Task<HttpResponseMessage>` | Executes the HTTP request |

### JWTDecoder

| Method | Returns | Description |
|--------|---------|-------------|
| `Decode(token)` | `Dictionary<string, object>` | Decodes JWT payload to dictionary |
| `DecodeAsJson(token)` | `string` | Decodes JWT payload to JSON string |

## Usage Patterns

### Chaining Multiple Headers

```csharp
var response = await NautilusHttpClient
    .Get(httpClient, "/api/data")
    .AddHeader("Authorization", $"Bearer {accessToken}")
    .AddHeader("Accept", "application/json")
    .AddHeader("User-Agent", "MyApp/1.0")
    .ExecuteAsync();
```

### Error Handling with EnsureStatusCode

```csharp
try
{
    var response = await NautilusHttpClient
        .Post(httpClient, "/users")
        .UseRawBody(jsonContent, "application/json")
        .EnsureStatusCode()  // Throws HttpRequestException on non-2xx
        .ExecuteAsync();

    // Process successful response
    var data = await response.Content.ReadAsStringAsync();
}
catch (HttpRequestException ex)
{
    // Handle HTTP error (4xx, 5xx)
    Console.WriteLine($"Request failed: {ex.Message}");
}
```

### Manual Status Code Checking

```csharp
var response = await NautilusHttpClient
    .Post(httpClient, "/users")
    .UseRawBody(jsonContent, "application/json")
    .ExecuteAsync();

if (response.StatusCode == HttpStatusCode.Created)
{
    // Handle success
}
else if (response.StatusCode == HttpStatusCode.Conflict)
{
    // Handle conflict
}
```

### Processing Response Content

```csharp
var response = await NautilusHttpClient
    .Get(httpClient, "/users/123")
    .ExecuteAsync();

// As string
var content = await response.Content.ReadAsStringAsync();

// As JSON
var user = await response.Content.ReadFromJsonAsync<User>();

// As stream
await using var stream = await response.Content.ReadAsStreamAsync();
```

### Using with Dependency Injection

```csharp
// Register HttpClient in DI container
builder.Services.AddScoped<HttpClient>(sp =>
    new HttpClient { BaseAddress = new Uri("https://api.example.com") });

// Inject into service
public class UserService
{
    private readonly HttpClient _httpClient;

    public UserService(HttpClient httpClient)
    {
        _httpClient = httpClient;
    }

    public async Task<User> GetUserAsync(int id)
    {
        var response = await NautilusHttpClient
            .Get(_httpClient, $"/users/{id}")
            .ExecuteAsync();

        return await response.Content.ReadFromJsonAsync<User>()
            ?? throw new InvalidOperationException("User not found");
    }
}
```

## Architecture

The library uses a **fluent interface pattern** where `NautilusHttpClient` implements multiple interfaces to guide the developer through building an HTTP request step-by-step.

### Request Building Flow

```
Static Entry Point
   ↓ (Get/Post/Put/Delete)
IHeader
   ↓ (Select body type or execute)
IFormUrlEncodedBody / IFormDataBody / IRawBody
   ↓ (Add parameters or execute)
IExecute
   ↓
ExecuteAsync() → HttpResponseMessage
```

### Design Principles

1. **Interface Segregation**: Each interface represents a specific stage in request building
2. **Fluent Chaining**: All methods return `this` to enable method chaining
3. **Type Safety**: Interfaces prevent invalid method combinations
4. **Zero Dependencies**: Uses only .NET standard library types
5. **Immutability**: Request building is additive and state is maintained internally

### Project Structure

```
src/Nautilus.HttpClient/
├── NautilusHttpClient.cs          # Main implementation (implements all interfaces)
├── HttpBodyType.cs                # Enum for tracking body type
├── Abstractions/                  # Interface definitions
│   ├── IHeader.cs                 # Entry point interface
│   ├── IFormUrlEncodedBody.cs     # Form-url-encoded body interface
│   ├── IFormDataBody.cs           # Multipart form data interface
│   ├── IRawBody.cs                # Raw body interface
│   └── IExecute.cs                # Execution interface
└── Jwt/
    └── JWTDecoder.cs              # JWT payload decoder
```

## Testing

The library includes comprehensive unit and integration tests:

```bash
# Run unit tests
dotnet test src/Nautilus.HttpClient.UnitTests

# Run integration tests
dotnet test src/Nautilus.HttpClient.IntegrationTests

# Run with coverage
dotnet test src/NautilusHttpClientSolution.sln --collect:"XPlat Code Coverage"
```

## License

This project is licensed under the [Business Source License 1.1](LICENSE).

The license allows free commercial use while protecting the source code. The code will automatically convert to the MIT License on **December 31, 2028**, 3 years after the first publication.

## Contributing

Contributions are welcome! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## Support

- **Issues**: [GitHub Issues](https://github.com/synthphonic/nautilus-httpclient/issues)
- **Discussions**: [GitHub Discussions](https://github.com/synthphonic/nautilus-httpclient/discussions)

## Acknowledgments

Inspired by modern fluent API design patterns and the need for cleaner HTTP client abstractions in .NET applications.

---

**Note**: This library is in active development. API changes may occur before version 1.0.0.
