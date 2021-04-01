---
layout: post
title:  "Immutable commands and queries"
date:   2021-04-21 18:33:21 +0200
categories: code
---
Recently for the first time I've used MediatR library in .NET Core. I quickly felt in love with it. In this post I want to share with you useful pattern I came up with working with MediatR.

When I started using MediatR I made all my commands and queries simple objects like this:

{% highlight csharp %}
public class CreateProductCommand : IRequest<Guid>
{
    public string Title { get; set; } = default;

    public string Description { get; set; } = default;

    public IEnumerable<string> Tags { get; set; } = new List<string>();
}
{% endhighlight %}

That's perfectly fine. But I think it's much better to implement them as immutable objects, because after command or query is created, it shouldn't be changed by subsequent code. If object is immutable it's guaranteed. It also helps to cleanup the code a little bit. Constructor is required, so default assignments can be removed. This approach also allows most commands and queries to be created in single line, without need for mapping libraries.

{% highlight csharp %}
[ImmutableObject(true)]
public class CreateProductCommand : IRequest<Guid>
{
    public string Title { get; }

    public string Description { get; }

    public IImmutableList<string> Tags { get; set; }

    public CreateProductCommand(string title, string description, IEnumerable<string> tags)
    {
        Title = title;
        Description = description;
        Tags = tags.ToImmutableList();
    }
}
{% endhighlight %}

I like this pattern so much, that I've started using it for objects that are produced by handlers. So data transport object like this:

{% highlight csharp %}
public class GetProductDetailsDto
{
    public string Title { get; set; } = default;

    public string Description { get; set; } = default;
}
{% endhighlight %}

In my code it is implemented as immutable object, like this:

{% highlight csharp %}
[ImmutableObject(true)]
public class GetProductDetailsDto
{
    public string Title { get; set; }

    public string Description { get; set; }

    public IImmutableList<string> Tags { get; set; }

    public GetProductDetailsDto(string title, string description, IEnumerable<string> tags)
    {
        Title = title;
        Description = description;
        Tags = tags.ToImmutableList();
    }
}
{% endhighlight %}

Reasons are the same as for making commands and queries immutable. I think it would be really bad design if objects returned by handler, were then updated by some other handler or method.

What about huge commands? Sometimes they could have more than twenty properties. It would be really messy if we would try to construct such objects in single line.

In those instances I implement mapping method. Let's say data for command comes in following class.

{% highlight csharp %}
public class CreateProductRequest
{
    public string Title { get; set; } = default;

    public string Description { get; set; } = default;

    public IEnumerable<string> Tags { get; set; } = new List<string>();
}
{% endhighlight %}

I could change command code to include mapping method:

{% highlight csharp %}
[ImmutableObject(true)]
public class CreateProductCommand : IRequest<Guid>
{
    public string Title { get; }

    public string Description { get; }

    public IImmutableList<string> Tags { get; set; }

    public CreateProductCommand(string title, string description, IEnumerable<string> tags)
    {
        Title = title;
        Description = description;
        Tags = tags.ToImmutableList();
    }

    public CreateProductCommand From(CreateProductRequest request)
    {
        var result = new CreateProductCommand(request.Title, requst.Description, request.Tags);
        return result;
    }
}
{% endhighlight %}

Problem with that solution is that it creates unnecessary coupling. Command shouldn't depend on other classes. I think much better idea is to make `From` an extension method and keep it in separate class, like this:

{% highlight csharp %}
public static class CreateProductCommandExtensions
{
    public static CreateProductCommand MapToCommand(this CreateProductRequest request)
    {
        var result = new CreateProductCommand(request.Title, requst.Description, request.Tags);
        return result;
    }
}
{% endhighlight %}

This solution is much better. Command class should only be concerned with holding data. With this extension method we can implement creating CreateProductCommand in single line:

{% highlight csharp %}
[HttpPost]
public async Task<IActionResult> Create([FromBody] CreateProductRequest request, CancellationToken cancellationToken)
{
    var command = request.MapToCommand();
    var result = await _mediator.Send(command, cancellationToken);
    return (result) ? Ok() : BadRequest();
}
{% endhighlight %}