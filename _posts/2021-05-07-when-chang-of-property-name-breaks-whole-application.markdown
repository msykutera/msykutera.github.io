---
layout: post
title:  "When change of property name breaks whole application"
date:   2021-05-07 18:33:21 +0200
categories: code
---
Recently I saw .NET Core API method implemented like this:

{% highlight csharp %}
[HttpPost]
public async Task<ActionResult<int>> Create(CreateTodoItemCommand command)
{
    return await Mediator.Send(command);
}
{% endhighlight %}

Using `CreateTodoItemCommand` as input parameter type is convenient. You don't have to create another class, say `CreateTodoItemRequest`, and map properties between the two. But I think it's convenient only if nothing changes. For example, if you change one of property name in `CreateTodoItemCommand` you can cause yourself big problem, if you are not cautious.

Imagine that API and Web UI projects are developed separately. Developer refactored his code and in the process he changed name of one of properties. Let's say he changed `Done` property to `IsDone`. Because this class is used as input parameter type in API method, it changes API design. It is small and unimportant change from the point of view of the API code. But it's important change from Web UI project standpoint. But developer hasn't noticed that. He ran all tests in the solution - all of them passed, because he hasn't introduced any bug in the API code. Situation like this could even break whole application, if it would impact critical endpoint.

I don't feel comfortable if smallest change of property name can cause big problem. It is risky and installs atmosphere of fear. If minor change of property name can have such a big, negactive impact then it's best to keep everything as it is, even if it's badly designed or implemented. I find that strong incentives to maintain status quo are one of the biggest enemies of code quality.

Other source of potential bugs from changing property names are often mapping libraries. Let's say that `CreatePortfolioPositionCommand` has same properties as `PortfolioPosition`. So you can just implement mapping of those two class like this in Automapper:

{% highlight csharp %}
CreateMap<CreatePositionCommand, PortfolioPosition>();
{% endhighlight %}

... and map them in single line, like this:

{% highlight csharp %}
var portfolioPosition = _mapper.Map<PortfolioPosition>(command);
{% endhighlight %}

Convenient isn't it? But only if property names won't change. If they do and developer will forget to update other class, he introduces a bug. You can fix this issue by implementing explicit mapping in Automapper, but it defeats whole idea of using mapping library in this case.

{% highlight csharp %}
CreateMap<CreatePositionCommand, Position>()
    .ForMember(x => x.Name, x => x.MapFrom(y => y.Name))
    .ForMember(x => x.Description, x => x.MapFrom(y => y.Description));
{% endhighlight %}

It's much better to implement constructor in `PortfolioPosition` class. Then you can create new object in single line without exposing yourself to introducing bug by changing property name.

{% highlight csharp %}
var portfolioPosition = new PortfolioPosition(command.Name, command.Request);
{% endhighlight %}

Alright, but what if you have to create class and fill all of it's 50 properties? Using constructor in that case would be messy. In those special situations you can always implement your own extension method that will do the mapping. For case described above code would look like this:

{% highlight csharp %}
public static class PortfolioPositionMappingExtensions
{
    public static PortfolioPosition From(this CreatePositionCommand command)
    {
        var portfolioPosition = new PortfolioPosition
        {
            Name = command.Name,
            Description = command.Description
        };
        return portfolioPosition;
    }
}
{% endhighlight %}

This approach will also guard you from introducing bug by changing property names. I use techniques described above in all my projects. That way I can easily change names my classes without fear of introducing bugs and wasting time on fixing broken test methods. I think that is very important if one wants to maintain high quality of his code.