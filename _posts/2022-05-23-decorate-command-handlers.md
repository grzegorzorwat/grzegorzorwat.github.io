---
layout: post
title:  "Decorating Command Handlers"
subtitle: "Using MediatR and ASP.NET Core DI"
date: 2020-05-22 12:00:00 -0200
background: '/img/posts/2022-05-23-decorate-command-handlers.jpg'
---

When implementing CQRS with the Command-Handler pattern using the MediatR library, we're using the `IRequest` interface for both commands and queries, but we often want different handling of those request types.

Following the example of [Kamil Grzybek's Modular Monolith with DDD](https://github.com/kgrzybek/modular-monolith-with-ddd) to have the ability to differentiate commands from queries, I created marker interfaces `ICommand` with `ICommandHandler` and `IQuery` with `IQueryHandler`:
```c#
public interface ICommand : IRequest { }
public interface ICommandHandler<in TCommand>
    : IRequestHandler<TCommand> where TCommand : ICommand { }

public interface IQuery<out TResult> : IRequest<TResult> { }
public interface IQueryHandler<in TQuery, TResult>
    : IRequestHandler<TQuery, TResult> where TQuery : IQuery<TResult> { }
```
And now it is possible to create decorators only for commands or queries handlers. The only difference is that in his application he's using the Autofac IoC container, which is more mature and feature rich than the built-in ASP.NET Core IoC container (also called ASP.NET Core DI), which I'm using in my project. Autofac has a concept of decorators and is able to return a decorated `ICommandHandler` when MediatR requests an `IRequestHandler`.

To achieve the same result with the ASP.NET Core DI, we have to write some custom code.

As an example of a decorator that I would like to use only on one type of operation is `UnitOfWorkCommandHandlerDecorator` with which I would like to decorate all my `ICommandHandlers`. This decorator commits `UnitOfWork` at the end of the `Handle` method. 

```c#
internal class UnitOfWorkCommandHandlerDecorator<TCommand>
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

As ASP.NET Core DI doesn't have a concept of decorators, we can use a library called [Scrutor](https://github.com/khellang/Scrutor) which extends the possibilities of registrations in the ASP.NET Core container, including registering decorators in ASP.NET. To register a decorator, we can use the `Decorate` extension method, which underneath replaces a registered implementation with a decorated one:
```c#
services.Decorate(typeof(ICommandHandler<>),
    typeof(UnitOfWorkCommandHandlerDecorator<>));
```
It also supports open generic registration, so in this case, it decorates every registered `ICommandHandler` with `UnitOfWorkCommandHandlerDecorator`.

Unfortunately, it isn't enough, because MediatR still requires and uses `IRequestHandlers` instead of `ICommandHandlers`. To get instances of handlers, it uses its `ServiceFactory` delegate, which in the default ASP.NET Core DI MediatR registration is `p => p.GetRequiredService`.

So, in order for MediatR to use decorated `ICommandHandlers`, we have to make ASP.NET Core DI return them whenever MediatR requests `IRequestHandlers`.

To accomplish this, we can define an extension method that makes use of Scrutor's `Scan` method to get all non-generic (closed) implementations of a given type and register them:
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

And then invoke it to register all closed implementations (so decorators are filtered out) of `ICommandHandler` with transient scope:
```c#
services.AddClosedGenericTypes(typeof(ICommandHandler<>).Assembly,
    typeof(ICommandHandler<>), ServiceLifetime.Transient);
```

Now we can decorate all registered `ICommandHandlers`:
```c#
services.Decorate(typeof(ICommandHandler<>),
    typeof(UnitOfWorkCommandHandlerDecorator<>));
```

The last step is to make the ASP.NET Core DI return registered decorated `ICommandHandlers` when MediatR requests an `IRequestHandler`.

To do so, we can define an extension method that gets all `ServiceDescriptors` of a given registered generic type, and then for each of these, it gets its expected type and register it with a factory that returns the registered type.
```c#
public static IServiceCollection RegisterGenericAs(
    this IServiceCollection services,
    Type registerType,
    Type asType)
{
    var registeredServiceDescriptors = services
        .Where(x => x.ServiceType.IsGenericType
            && x.ServiceType.GetGenericTypeDefinition() == registerType)
        .ToArray();

    foreach (var serviceDescriptor in registeredServiceDescriptors)
    {
        var requestHandlerType = serviceDescriptor.ServiceType
            .GetInterfaces()
            .First(x => x.IsGenericType
                && x.GetGenericTypeDefinition() == asType);
        services.Add(new ServiceDescriptor(requestHandlerType,
            sp => sp.GetRequiredService(serviceDescriptor.ServiceType),
                serviceDescriptor.Lifetime));
    }

    return services;
}
```

So in our case, when we invoke it, it gets all `ICommandHandlers'` `ServiceDescriptors` and then registers their `IRequestHandler` type with the factory, which returns a decorated `ICommandHandler`:
```c#
services.RegisterGenericAs(typeof(ICommandHandler<>),
    typeof(IRequestHandler<>));
```

And at the end, we can use the `AddMediatR` extension method to register `IRequestHandlers` that are not `ICommandHandlers` and the other of the dependencies required by MediatR:
```c#
services.AddMediatR(typeof(ICommand));
```

Now whenever MediatR requests `IRequestHandler` ASP.NET Core DI will return a decorated `ICommandHandler`.

## Alternative approach
Another way to get MediatR to use decorated `ICommandHandler` would be to make it request `ICommandHandlers` from the container instead of `IRequestHandlers`.

In order to do this, we have to register `ICommandHandlers` and decorate them like in the first approach:
```c#
services.AddClosedGenericTypes(typeof(ICommand).Assembly,
    typeof(ICommandHandler<>), ServiceLifetime.Transient);
services.Decorate(typeof(ICommandHandler<>),
    typeof(UnitOfWorkCommandHandlerDecorator<>));
```

And then register custom MediatR's `ServiceFactory` delegate, which would check if MediatR requests `IRequestHandler` and if there is `ICommandHandler` for it:
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

But I don't like this approach because it is harder to understand, and it has more overhead on runtime.

## Conclusion
As you can see, it is possible to create marker interfaces for commands and queries and use them with MediatR and ASP.NET Core DI. As an alternative, I'd consider using more advanced DI Container like Autofac if you have a need for features which ASP.NET Core DI doesn't provide. Another approach I can recommend is using the Command-Handler pattern without MediatR like in [Oskar Dudycz's How to register all CQRS handlers by convention](https://event-driven.io/en/how_to_register_all_mediatr_handlers_by_convention/).