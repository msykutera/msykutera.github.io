---
layout: post
title:  "Should I use data access code inside handlers?"
date:   2021-04-30 18:33:21 +0200
categories: code
---
Recently I was confronted with task of architecting .NET Core API for enterpise application. Rather than reinvent the wheel and build it from scratch, I based it on Clean Architecture Solution Template created by Jason Taylor. One of the most controversial choices he made was to use data access code inside the handlers. Here is example handler from Clean Architecture Solution Template:

{% highlight csharp %}
public class CreateTodoItemCommandHandler : IRequestHandler<CreateTodoItemCommand, int>
{
    private readonly IApplicationDbContext _context;

    public CreateTodoItemCommandHandler(IApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<int> Handle(CreateTodoItemCommand request, CancellationToken cancellationToken)
    {
        var entity = new TodoItem
        {
            ListId = request.ListId,
            Title = request.Title,
            Done = false
        };
        entity.DomainEvents.Add(new TodoItemCreatedEvent(entity));
        _context.TodoItems.Add(entity);
        await _context.SaveChangesAsync(cancellationToken);
        return entity.Id;
    }
}
{% endhighlight %}

I talked to three developers about this and for each of them it was big no no. All of them argued that it's mixing business logic with data access code (?), which they considered poor design. They prefered different flavor of handlers, one implemented with data access fully abstracted away in repositories.

{% highlight csharp %}
public class CreateTodoItemCommandHandler : IRequestHandler<CreateTodoItemCommand, int>
{
    private readonly ITodoItemRepository _repository;

    public CreateTodoItemCommandHandler(ITodoItemRepository repository)
    {
        _repository = repository;
    }

    public async Task<int> Handle(CreateTodoItemCommand request, CancellationToken cancellationToken)
    {
        var entityId = await _repository.Add(request.ListId, request.Title, false);
        return entityId;
    }
}
{% endhighlight %}

I don't agree with developers that reject first approach. I think second approach is much better because it is much simpler and still it provides a lot of flexibility in choice of database. In Microsoft documentation I've found over twenty different providers for Entity Framework Core, including ones for Oracle, Cosmos, MySql, PostgreSQL, Firebird, Teradata and OpenEdge databases. I bet that one can find couple times more providers on the Internet. So this first code snippet is independent of database - you could use same code with probably over 50 different databases. You can also implement one yourself if required. Also EF Core DbContext already implements Unit of Work and Repository patterns, so wrapping it up in Repository or Unit of Work pattern doesn't add any value.

Lastly when used with repostiories, most of request handlers become thin. All they do is to call repository method and return data. In example above handler can be reduced to single line. Conceptually it's not even a handler, because request is really handled in repository. Handler just passes it to repository and returns data it receives. I think it's much better when most of application logic is encapsulated in handler itself, because it's easy to find and change. It also requires much less code.

Obviosuly if you don't use Entity Framework Core but some other data access library, that doesn't support changing data provider, it is best if you abstract it away using Unit of Work or Repository pattern. But if you use EF Core I don't think that is necessary. You can greatly simplify your application using it directly in you handlers and still enjoy flexibility of freely changing data persistence technology.