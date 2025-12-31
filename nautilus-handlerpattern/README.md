# Nautilus.HandlerPattern

[![Build Status](https://github.com/synthphonic/nautilus-handlerpattern/actions/workflows/ci.yml/badge.svg)](https://github.com/synthphonic/nautilus-handlerpattern/actions/workflows/ci.yml)
[![NuGet Version](https://img.shields.io/nuget/vpre/Nautilus.HandlerPattern)](https://www.nuget.org/packages/Nautilus.HandlerPattern/)
[![License: BSL-1.1](https://img.shields.io/badge/License-BSL--1.1-blue.svg)](LICENSE)

A flexible .NET library implementing the Handler Pattern (Mediator/CQRS variant) with pipeline behaviors and cross-cutting concerns support. This library provides a clean architecture for processing requests through a pipeline of behaviors with built-in support for logging, validation, error handling, caching, auditing, and metrics.

## Features

- **Handler Pattern**: Mediator/CQRS-style request/response handling
- **Pipeline Behaviors**: Cross-cutting concerns via composable pipeline
- **Built-in Behaviors**: Validation, logging, error handling, caching, auditing, metrics
- **Dependency Injection**: Full Microsoft.Extensions.DependencyInjection integration
- **Performance Optimized**: ValueTask-based, pre-compiled pipelines
- **.NET 9.0**: Built on the latest .NET platform
- **Clean Architecture**: Clear separation of abstractions and implementations

## Installation

Install the package via [NuGet](https://www.nuget.org/packages/Nautilus.HandlerPattern/):

```bash
dotnet add package Nautilus.HandlerPattern
```

Or via the Package Manager Console:

```powershell
Install-Package Nautilus.HandlerPattern
```

## Quick Start

### Basic Setup

```csharp
using Nautilus.HandlerPattern;

// In Program.cs or Startup.cs
builder.Services.AddHandlerPattern();

// Register handlers
builder.Services.AddHandler<CreateUserRequest, UserResponse, CreateUserHandler>();
```

### Define a Handler

```csharp
// Request
public record CreateUserRequest(string Name, string Email);

// Response
public record UserResponse(int Id, string Name, string Email);

// Handler
public class CreateUserHandler : IHandler<CreateUserRequest, UserResponse>
{
    private readonly IUserRepository _repository;

    public CreateUserHandler(IUserRepository repository)
    {
        _repository = repository;
    }

    public async ValueTask<UserResponse> HandleAsync(
        CreateUserRequest request,
        CancellationToken cancellationToken = default)
    {
        User user = await _repository.CreateAsync(request.Name, request.Email);
        return new UserResponse(user.Id, user.Name, user.Email);
    }
}
```

### Send Requests

```csharp
public class UserService
{
    private readonly IHandlerDispatcher _dispatcher;

    public UserService(IHandlerDispatcher dispatcher)
    {
        _dispatcher = dispatcher;
    }

    public async Task<UserResponse> CreateUserAsync(string name, string email)
    {
        var request = new CreateUserRequest(name, email);
        UserResponse response = await _dispatcher.SendAsync<UserResponse>(request);
        return response;
    }
}
```

### Void Handlers (No Response)

```csharp
// Request
public record DeleteUserRequest(int UserId);

// Handler
public class DeleteUserHandler : IHandler<DeleteUserRequest>
{
    private readonly IUserRepository _repository;

    public DeleteUserHandler(IUserRepository repository)
    {
        _repository = repository;
    }

    public async ValueTask HandleAsync(
        DeleteUserRequest request,
        CancellationToken cancellationToken = default)
    {
        await _repository.DeleteAsync(request.UserId);
    }
}

// Send void request
await _dispatcher.SendAsync(new DeleteUserRequest(userId));
```

## Pipeline Behaviors

### Built-in Behaviors

The library includes these pre-built behaviors:

| Behavior | Purpose |
|----------|---------|
| `ValidationBehavior` | Validates requests using FluentValidation |
| `LoggingBehavior` | Logs request/response details |
| `ErrorHandlingBehavior` | Handles exceptions and converts to errors |
| `CachingBehavior` | Caches responses based on request |
| `AuditBehavior` | Logs audit trail for operations |
| `MetricsBehavior` | Collects performance metrics |

### Custom Behaviors

```csharp
public class RetryBehavior<TRequest, TResponse> : IHandlerBehavior<TRequest, TResponse>
    where TRequest : class
    where TResponse : class
{
    private readonly ILogger<RetryBehavior<TRequest, TResponse>> _logger;

    public RetryBehavior(ILogger<RetryBehavior<TRequest, TResponse>> logger)
    {
        _logger = logger;
    }

    public async ValueTask<TResponse> HandleAsync(
        TRequest request,
        Func<TRequest, ValueTask<TResponse>> next,
        IHandlerContext context,
        CancellationToken cancellationToken = default)
    {
        int retryCount = 0;
        const int maxRetries = 3;

        while (true)
        {
            try
            {
                return await next(request);
            }
            catch (Exception ex) when (retryCount < maxRetries)
            {
                retryCount++;
                _logger.LogWarning(ex, "Retry {RetryCount}/{MaxRetries}", retryCount, maxRetries);
                await Task.Delay(1000 * retryCount, cancellationToken);
            }
        }
    }
}

// Register custom behavior
services.AddBehavior<CreateUserRequest, UserResponse, RetryBehavior<CreateUserRequest, UserResponse>>();
```

## Context Pattern

Access shared metadata between behaviors:

```csharp
public class CustomBehavior : IHandlerBehavior<TRequest, TResponse>
{
    public async ValueTask<TResponse> HandleAsync(
        TRequest request,
        Func<TRequest, ValueTask<TResponse>> next,
        IHandlerContext context,
        CancellationToken cancellationToken = default)
    {
        // Set metadata
        context.SetMetadata("StartTime", DateTime.UtcNow);

        // Execute next
        var response = await next(request);

        // Get metadata set by previous behaviors
        var userId = context.GetMetadata<string>("UserId");

        return response;
    }
}
```

## API Reference

### Core Interfaces

#### IHandler\<TRequest, TResponse\>

Defines a handler that processes a request and returns a response.

```csharp
public interface IHandler<in TRequest, TResponse>
    where TRequest : class
    where TResponse : class
{
    ValueTask<TResponse> HandleAsync(TRequest request, CancellationToken cancellationToken = default);
}
```

#### IHandler\<TRequest\>

Defines a handler that processes a request without returning a response.

```csharp
public interface IHandler<in TRequest>
    where TRequest : class
{
    ValueTask HandleAsync(TRequest request, CancellationToken cancellationToken = default);
}
```

#### IHandlerBehavior\<TRequest, TResponse\>

Defines a pipeline behavior that can process requests before/after the handler.

```csharp
public interface IHandlerBehavior<in TRequest, TResponse>
    where TRequest : class
    where TResponse : class
{
    ValueTask<TResponse> HandleAsync(
        TRequest request,
        Func<TRequest, ValueTask<TResponse>> next,
        IHandlerContext context,
        CancellationToken cancellationToken = default);
}
```

#### IHandlerDispatcher

Dispatches requests to the appropriate handler.

```csharp
public interface IHandlerDispatcher
{
    ValueTask<TResponse> SendAsync<TResponse>(IRequest<TResponse> request, CancellationToken cancellationToken = default);
    ValueTask SendAsync(IRequest request, CancellationToken cancellationToken = default);
}
```

### IHandlerContext

Provides context and metadata for request processing.

| Property | Type | Description |
|----------|------|-------------|
| `ExecutionId` | `string` | Unique identifier for the request |
| `Items` | `IDictionary<string, object>` | Custom metadata storage |
| `ExecutedBehaviors` | `IList<string>` | List of executed behavior names |

| Method | Description |
|--------|-------------|
| `GetMetadata<T>(string key)` | Get typed metadata |
| `SetMetadata(string key, object value)` | Set metadata |

## Usage Patterns

### Validation with FluentValidation

```csharp
// Define validator
public class CreateUserRequestValidator : AbstractValidator<CreateUserRequest>
{
    public CreateUserRequestValidator()
    {
        RuleFor(x => x.Name).NotEmpty().MaximumLength(100);
        RuleFor(x => x.Email).NotEmpty().EmailAddress();
    }
}

// Register validator
services.AddValidatorsFromAssemblyContaining<CreateUserRequestValidator>();

// Validation happens automatically in the pipeline
```

### Response Caching

```csharp
// Enable caching for a handler
public class GetUserHandler : IHandler<GetUserRequest, UserResponse>
{
    [Cache(Duration = 300)] // Cache for 5 minutes
    public async ValueTask<UserResponse> HandleAsync(GetUserRequest request, CancellationToken cancellationToken = default)
    {
        return await _repository.FindAsync(request.UserId);
    }
}
```

### Error Handling

```csharp
public class SafeUserService
{
    public async Task<Result<UserResponse>> CreateUserSafe(string name, string email)
    {
        try
        {
            var response = await _dispatcher.SendAsync<UserResponse>(
                new CreateUserRequest(name, email)
            );
            return Result.Success(response);
        }
        catch (ValidationException ex)
        {
            return Result.Failure<UserResponse>(
                new Error("VALIDATION_ERROR", ex.Message)
            );
        }
    }
}
```

### Pipeline Builder

```csharp
// Build custom pipeline
var pipeline = PipelineBuilder.Create<CreateUserRequest, UserResponse>()
    .AddBehavior<ValidationBehavior<CreateUserRequest, UserResponse>>()
    .AddBehavior<LoggingBehavior<CreateUserRequest, UserResponse>>()
    .AddBehavior<CachingBehavior<CreateUserRequest, UserResponse>>()
    .Build();

// Register custom pipeline
services.AddSingleton(pipeline);
```

## Architecture

The library follows Clean Architecture principles:

```
Abstractions/          # Core interfaces (no dependencies)
├── IHandler<,>
├── IHandlerBehavior<,>
├── IHandlerContext
└── IHandlerDispatcher

Behaviors/            # Cross-cutting concerns (depend on Abstractions)
├── ValidationBehavior
├── LoggingBehavior
├── ErrorHandlingBehavior
└── ...

Implementation/       # Core implementations (depend on Abstractions)
├── HandlerDispatcher
├── HandlerContext
└── HandlerBase

Pipeline/             # Pipeline execution (depend on Abstractions + Implementation)
├── HandlerPipeline
├── PipelineBuilder
└── BehaviorBuilder

Extensions/           # DI integration
└── ServiceCollectionExtensions
```

## Performance Considerations

- **ValueTask**: All handlers return `ValueTask` to reduce allocations
- **Pre-compiled Pipeline**: Pipeline is compiled once at construction (O(1) execution)
- **ConfigureAwait(false)**: All async calls avoid capturing synchronization context
- **Transient Services**: Handlers and behaviors are transient by default (stateless)

## License

This project is licensed under the [Business Source License 1.1](LICENSE).

The license allows free commercial use while protecting the source code. The code will automatically convert to the MIT License on **December 31, 2028**, 3 years after the first publication.

## Contributing

Contributions are welcome! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## Support

- **Issues**: [GitHub Issues](https://github.com/synthphonic/nautilus-handlerpattern/issues)
- **Discussions**: [GitHub Discussions](https://github.com/synthphonic/nautilus-handlerpattern/discussions)

## Acknowledgments

Inspired by the [Mediator Pattern](https://refactoring.guru/design-patterns/mediator) and popularized by libraries like [MediatR](https://github.com/jbogard/MediatR).

---

**Note**: This library is in active development. API changes may occur before version 1.0.0.
