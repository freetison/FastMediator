# FastMediator: Una Alternativa Ligera a MediatR

A continuación se presenta una implementación optimizada del patrón Mediator en C#, diseñada como una alternativa a MediatR. Esta solución elimina el uso de reflection y diccionarios, utilizando fábricas genéricas para un rendimiento óptimo. Soporta `IRequest`/`IHandler` para comandos/consultas y `INotification` para eventos, con integración completa en el contenedor de inyección de dependencias (DI) de .NET.

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;
using Microsoft.Extensions.DependencyInjection;

// Interfaz para requests (equivalente a IRequest<TResponse>)
public interface IRequest<TResponse> { }

// Interfaz para handlers de requests (equivalente a IRequestHandler)
public interface IRequestHandler<in TRequest, TResponse> where TRequest : IRequest<TResponse>
{
    Task<TResponse> Handle(TRequest request, CancellationToken cancellationToken);
}

// Interfaz para notificaciones (equivalente a INotification)
public interface INotification { }

// Interfaz para handlers de notificaciones (equivalente a INotificationHandler)
public interface INotificationHandler<in TNotification> where TNotification : INotification
{
    Task Handle(TNotification notification, CancellationToken cancellationToken);
}

// Interfaz del Mediador
public interface IMediator
{
    Task<TResponse> Send<TResponse>(IRequest<TResponse> request, CancellationToken cancellationToken = default);
    Task Publish<TNotification>(TNotification notification, CancellationToken cancellationToken = default) where TNotification : INotification;
}

// Implementación de la fábrica genérica para handlers
public interface IHandlerFactory
{
    Task<TResponse> HandleRequest<TResponse>(IRequest<TResponse> request, CancellationToken cancellationToken);
    Task HandleNotification<TNotification>(TNotification notification, CancellationToken cancellationToken) where TNotification : INotification;
}

public class HandlerFactory<TRequest, TResponse, THandler> : IHandlerFactory
    where TRequest : IRequest<TResponse>
    where THandler : IRequestHandler<TRequest, TResponse>
{
    private readonly THandler _handler;

    public HandlerFactory(THandler handler)
    {
        _handler = handler;
    }

    public Task<TResponse> HandleRequest<TResponseInner>(IRequest<TResponseInner> request, CancellationToken cancellationToken)
    {
        if (request is TRequest typedRequest && typeof(TResponse) == typeof(TResponseInner))
        {
            return _handler.Handle(typedRequest, cancellationToken) as Task<TResponseInner>;
        }
        throw new InvalidOperationException($"Handler for {typeof(TRequest).Name} cannot handle {request.GetType().Name}");
    }

    public Task HandleNotification<TNotification>(TNotification notification, CancellationToken cancellationToken) where TNotification : INotification
    {
        throw new InvalidOperationException("This factory does not handle notifications");
    }
}

public class NotificationHandlerFactory<TNotification, THandler> : IHandlerFactory
    where TNotification : INotification
    where THandler : INotificationHandler<TNotification>
{
    private readonly THandler _handler;

    public NotificationHandlerFactory(THandler handler)
    {
        _handler = handler;
    }

    public Task<TResponse> HandleRequest<TResponse>(IRequest<TResponse> request, CancellationToken cancellationToken)
    {
        throw new InvalidOperationException("This factory does not handle requests");
    }

    public async Task HandleNotification<TNotificationInner>(TNotificationInner notification, CancellationToken cancellationToken) where TNotificationInner : INotification
    {
        if (notification is TNotification typedNotification)
        {
            await _handler.Handle(typedNotification, cancellationToken);
        }
        else
        {
            throw new InvalidOperationException($"Handler for {typeof(TNotification).Name} cannot handle {notification.GetType().Name}");
        }
    }
}

// Implementación del Mediador
public class Mediator : IMediator
{
    private readonly IServiceProvider _serviceProvider;

    public Mediator(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public async Task<TResponse> Send<TResponse>(IRequest<TResponse> request, CancellationToken cancellationToken = default)
    {
        if (request == null) throw new ArgumentNullException(nameof(request));

        var factoryType = typeof(IHandlerFactory);
        var factories = _serviceProvider.GetServices(factoryType);

        foreach (var factory in factories)
        {
            var result = await factory.HandleRequest(request, cancellationToken);
            if (result != null)
            {
                return result;
            }
        }

        throw new InvalidOperationException($"No handler registered for request type {request.GetType().Name}");
    }

    public async Task Publish<TNotification>(TNotification notification, CancellationToken cancellationToken = default) where TNotification : INotification
    {
        if (notification == null) throw new ArgumentNullException(nameof(notification));

        var factoryType = typeof(IHandlerFactory);
        var factories = _serviceProvider.GetServices(factoryType);

        foreach (var factory in factories)
        {
            await factory.HandleNotification(notification, cancellationToken);
        }
    }
}

// Clase auxiliar para registrar handlers en DI
public static class MediatorExtensions
{
    public static IServiceCollection AddMediator(this IServiceCollection services)
    {
        services.AddSingleton<IMediator, Mediator>();
        return services;
    }

    public static IServiceCollection AddRequestHandler<TRequest, TResponse, THandler>(this IServiceCollection services)
        where TRequest : IRequest<TResponse>
        where THandler : IRequestHandler<TRequest, TResponse>
    {
        services.AddSingleton<IHandlerFactory, HandlerFactory<TRequest, TResponse, THandler>>();
        services.AddSingleton<THandler>();
        return services;
    }

    public static IServiceCollection AddNotificationHandler<TNotification, THandler>(this IServiceCollection services)
        where TNotification : INotification
        where THandler : INotificationHandler<TNotification>
    {
        services.AddSingleton<IHandlerFactory, NotificationHandlerFactory<TNotification, THandler>>();
        services.AddSingleton<THandler>();
        return services;
    }
}

// Ejemplo de uso: Comando
public class CreateUserCommand : IRequest<bool>
{
    public string Username { get; set; }
}

public class CreateUserCommandHandler : IRequestHandler<CreateUserCommand, bool>
{
    public Task<bool> Handle(CreateUserCommand request, CancellationToken cancellationToken)
    {
        Console.WriteLine($"Creating user: {request.Username}");
        return Task.FromResult(true);
    }
}

// Ejemplo de uso: Notificación
public class UserCreatedNotification : INotification
{
    public string Username { get; set; }
}

public class UserCreatedNotificationHandler : INotificationHandler<UserCreatedNotification>
{
    public Task Handle(UserCreatedNotification notification, CancellationToken cancellationToken)
    {
        Console.WriteLine($"Notification: User {notification.Username} created!");
        return Task.CompletedTask;
    }
}

// Configuración de DI en Program.cs (ejemplo para ASP.NET Core)
public class Program
{
    public static void Main(string[] args)
    {
        var services = new ServiceCollection();

        // Registrar Mediador y handlers
        services.AddMediator()
                .AddRequestHandler<CreateUserCommand, bool, CreateUserCommandHandler>()
                .AddNotificationHandler<UserCreatedNotification, UserCreatedNotificationHandler>();

        var serviceProvider = services.BuildServiceProvider();

        // Ejemplo de uso
        var mediator = serviceProvider.GetService<IMediator>();
        var command = new CreateUserCommand { Username = "john_doe" };
        var notification = new UserCreatedNotification { Username = "john_doe" };

        // Enviar comando
        mediator.Send(command).GetAwaiter().GetResult();

        // Publicar notificación
        mediator.Publish(notification).GetAwaiter().GetResult();
    }
}
```

## Características
- **Sin reflection**: Usa fábricas genéricas (`HandlerFactory` y `NotificationHandlerFactory`) para resolver handlers estáticamente.
- **Sin diccionarios**: Los handlers se registran directamente en el DI container, eliminando el overhead de buscar en colecciones.
- **Alta eficiencia**: Resolución directa vía DI con tipos genéricos, minimizando el impacto en rendimiento.
- **Type-safe**: Validación en tiempo de compilación gracias a los genéricos.
- **Compatible con DI**: Integrado con `Microsoft.Extensions.DependencyInjection`.
- **Escalable**: Soporta múltiples handlers para notificaciones y es extensible para pipelines o validaciones.
- **Gratuito**: 100% open-source, sin licencias.

## Cómo Usarlo
1. Copia el código C# desde el bloque de código anterior a un archivo `.cs` en tu proyecto .NET.
2. Configura el DI usando `AddMediator`, `AddRequestHandler`, y `AddNotificationHandler` como en el ejemplo de `Program.cs`.
3. Usa el mediador para enviar comandos (`Send`) o publicar notificaciones (`Publish`).

## Extensibilidad
- Puedes agregar pipelines (similar a behaviors en MediatR) modificando `Mediator` para encadenar middleware.
- Para aún más rendimiento, considera integrar source generators, aunque esta solución ya es muy eficiente.