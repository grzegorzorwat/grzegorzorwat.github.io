---
layout: post
title:  "Decorating Command Handlers"
subtitle: "Using MediatR and ASP.NET Core DI"
date: 2022-05-28 14:00:00 -0200
background: '/img/posts/2022-05-23-decorate-command-handlers.jpg'
---

When implementing CQRS with the MediatR library, we're using the `IRequest` interface for both commands and queries, but we often want different handling of those request types.

Following the example of [Kamil Grzybek's Modular Monolith with DDD](https://github.com/kgrzybek/modular-monolith-with-ddd), we can make marker interfaces ICommand with ICommandHandler and IQuery with IQueryHandler to distinguish commands from queries:
```c#
public interface ICommand : IRequest { }
public interface ICommandHandler<in TCommand>
    : IRequestHandler<TCommand> where TCommand : ICommand { }

public interface IQuery<out TResult> : IRequest<TResult> { }
public interface IQueryHandler<in TQuery, TResult>
    : IRequestHandler<TQuery, TResult> where TQuery : IQuery<TResult> { }
```

It is now possible to create decorators only for command or query handlers. An example of a decorator that should decorate only `ICommandHandlers` is the `UnitOfWorkCommandHandlerDecorator`, which commits `UnitOfWork` at the end of the `Handle` method.

```c#
internal sealed class UnitOfWorkCommandHandlerDecorator<TCommand>
    : ICommandHandler<TCommand> where TCommand : ICommand
{
    private readonly ICommandHandler<TCommand> _decorated;
    private readonly IUnitOfWork _unitOfWork; 

    public UnitOfWorkCommandHandlerDecorator(
        ICommandHandler<TCommand> decorated,
        IUnitOfWork unitOfWork)
    {
        _decorated = decorated;
        _unitOfWork = unitOfWork;
    }

    public async Task<Unit> Handle(
        TCommand request,
        CancellationToken cancellationToken)
    {
        var res = await _decorated.Handle(request, cancellationToken);
        await _unitOfWork.CommitAsync();
        return res;
    }
}
```

We now need a method to register all `ICommandHandlers` because we can't use MediatR's registration method, which doesn't know about our new interfaces. To do this, we can define an extension method that uses a library called [Scrutor](https://github.com/khellang/Scrutor), which extends the registration capabilities of ASP.NET Core DI. With its `Scan` method, we can find and register all closed implementations of a given generic type (filtering out decorators):

```c#
public static IServiceCollection AddClosedGenericTypes(
    this IServiceCollection services,
    Assembly assembly,
    Type typeToRegister,
    ServiceLifetime serviceLifetime)
{
    services.Scan(x => x.FromAssemblies(assembly)
        .AddClasses(classes => classes.AssignableTo(typeToRegister)
            .Where(t => !t.IsGenericType))
        .AsImplementedInterfaces(t => t.IsGenericType
            && t.GetGenericTypeDefinition() == typeToRegister)
        .WithLifetime(serviceLifetime));
    return services;
}
```

Then invoke it to register all closed implementations of `ICommandHandler` with transient scope:

```c#
services.AddClosedGenericTypes(typeof(ICommandHandler<>).Assembly,
    typeof(ICommandHandler<>), ServiceLifetime.Transient);
```

As ASP.NET Core DI doesn't have a concept of decorators, we can use Scrutor's `Decorate` method, which underneath replaces a registered implementation with a decorated one:

```c#
services.Decorate(typeof(ICommandHandler<>),
    typeof(UnitOfWorkCommandHandlerDecorator<>));
```

It also supports open generic registration, so in this case, it decorates every registered `ICommandHandler` with `UnitOfWorkCommandHandlerDecorator`.

Unfortunately, it isn't enough, because MediatR still requires and uses `IRequestHandlers` rather than `ICommandHandlers`. In order to make it use decorated `ICommandHandlers`, we have to force ASP.NET Core DI to return them whenever MediatR requests `IRequestHandlers`.

To accomplish this, we can define an extension method that gets all `ServiceDescriptors` of a given registered generic type, and then for each of these, it gets its exact parent type and registers it with a factory that returns the registered child type.

```c#
public static IServiceCollection RegisterGenericsAsItsParent(
    this IServiceCollection services,
    Type registeredGenericType,
    Type parentGenericType)
{
    var registeredServiceDescriptors = services
        .Where(x => x.ServiceType.IsGenericType
            && x.ServiceType.GetGenericTypeDefinition()
                == registeredGenericType)
        .ToArray();

    foreach (var serviceDescriptor in registeredServiceDescriptors)
    {
        var parentType = serviceDescriptor.ServiceType
            .GetInterfaces()
            .First(x => x.IsGenericType
                && x.GetGenericTypeDefinition() == parentGenericType);
        services.Add(new ServiceDescriptor(parentType,
            sp => sp.GetRequiredService(serviceDescriptor.ServiceType),
                serviceDescriptor.Lifetime));
    }

    return services;
}
```

When we call it, it retrieves the `ServiceDescriptors` of all `ICommandHandlers` and then registers their parent IRequestHandler types with the factory, which returns a decorated `ICommandHandler` instead:

```c#
services.RegisterGenericsAsItsParent(typeof(ICommandHandler<>),
    typeof(IRequestHandler<,>));
```

We must register it as `IRequestHandler<,>` instead of `IRequestHandler<>` because when MediatR requests `IRequestHandler` with no return type, it actually requests `IRequestHandler<TRequest, Unit>` as `Unit` is its default returning type.

ASP.NET Core DI will now return a decorated `ICommandHandler` whenever MediatR requests an `IRequestHandler`.

Lastly, we can use the `AddMediatR` extension method to register IRequestHandlers that are not `ICommandHandlers` as well as the other dependencies that MediatR needs:

```c#
services.AddMediatR(typeof(ICommand));
```

Another way to make MediatR use decorated `ICommandHandler` is to make it request `ICommandHandlers` from the container rather than `IRequestHandlers`.

In order to do this, we have to register `ICommandHandlers` and decorate them similarly to the first approach:

```c#
services.AddClosedGenericTypes(typeof(ICommand).Assembly,
    typeof(ICommandHandler<>), ServiceLifetime.Transient);
services.Decorate(typeof(ICommandHandler<>),
    typeof(UnitOfWorkCommandHandlerDecorator<>));
```

Then register custom MediatR's `ServiceFactory` delegate, that checks if MediatR requests an `IRequestHandler` and if there is an `ICommandHandler` for it, and if there is, it returns it instead:

```c#
services.AddTransient<ServiceFactory>(sp =>
{
    return t =>
    {
        if (t.IsGenericType
            && t.GetGenericTypeDefinition() == typeof(IRequestHandler<,>))
        {
            var genericArguments = t.GetGenericArguments();
            var commandHandlerType = typeof(ICommandHandler<,>)
                .MakeGenericType(genericArguments);
            return sp.GetService(commandHandlerType)
                ?? sp.GetRequiredService(t);
        }
        return sp.GetRequiredService(t);
    };
});
```

However, I don't like this approach because it is harder to understand and has overhead on runtime.

As you can see, it is possible to create marker interfaces for commands and queries and use them with MediatR and ASP.NET Core DI. As an alternative, as shown in [Modular Monolith with DDD repository](https://github.com/kgrzybek/modular-monolith-with-ddd), I'd consider using a more advanced DI container, such as Autofac, which would make this task much easier. I can also recommend using [How to register all CQRS handlers by convention by Oskar Dudycz](https://event-driven.io/en/how_to_register_all_mediatr_handlers_by_convention/) approach as an alternative to MediatR.