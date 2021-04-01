---
layout: post
title:  "Thoughts on Test Driven Development"
date:   2021-04-14 18:33:21 +0200
categories: code
---
Test Driven Development is presented by some as silver bullet solution – process that should be used in every software project, to every line of code that is written. TDD has it's merits, there is no doubt. When used properly it will assure high code coverage, which is very important. But I don't think that TDD works well in all situations.

Let's consider following scenario, where QuickSort algorithm is implemented and encapsulated using Strategy Pattern.

{% highlight csharp %}
public QuickSortService<T> : ISortService<T>
{
    public IEnumerable<T> Sort(IEnumerable<T> values)
    {
        // logic goes here
    }
}
{% endhighlight %}

In this situation Test Driven Development is perfect. You write unit tests first and they provide you quick feedback loop on code you are writing. This will speed up code development – you won't waste your time manually checking if code is working properly. Also design is very simple and it's hihgly unlikely that it will change. Because of that tests won't change during the development process.

But there are other scenarios where Test Driven Development is not helpful at all, because it slows down development without adding value. Let's consider different scneraio.

Recently I wrote repository method that fetched descriptions for given product id. I started by coding interface. It looks reasonably, right?

{% highlight csharp %}
public interface IProductDescriptionsRepository : IRepository<ProductDescription>
{
    IEnumerable<ProductDescriptionDto> GetProductDescriptions(string productId);
}
{% endhighlight %}

But then when I was implementing repository method itself I remembered, that it should be asynchronous. So I've changed interface.

{% highlight csharp %}
public interface IProductDescriptionsRepository : IRepository<ProductDescription>
{
    Task<IEnumerable<ProductDescriptionDto>> GetProductDescriptions(string productId);
}
{% endhighlight %}

I continued development when I remembered another thing, because it's async I should pass Cancellation Token to the method. So I've changed interface for the second time.

{% highlight csharp %}
public interface IProductDescriptionsRepository : IRepository<ProductDescription>
{
    Task<IEnumerable<ProductDescriptionDto>> GetProductDescriptions(string productId, CancellationToken cancellationToken);
}
{% endhighlight %}

Then I noticed that this method should support pagination. So I change the interface once more.

{% highlight csharp %}
public interface IProductDescriptionsRepository : IRepository<ProductDescription>
{
    Task<PaginatedListDto<ProductDescriptionDto>> GetProductDescriptions(string productId, CancellationToken cancellationToken);
}
{% endhighlight %}

When I did that, I had to add additional method input parameter, because descriptions will be displayed on the screen in separate sections. So I changed the interface for last time.

{% highlight csharp %}
public interface IProductDescriptionsRepository : IRepository<ProductDescription>
{
    Task<PaginatedListDto<ProductDescriptionDto>> GetProductDescriptions(string productId, ProductDescriptionType type, CancellationToken cancellationToken);
}
{% endhighlight %}

I didn't use TDD to write that code and it was a blessing, because I didn't have to spent time on updating tests. TDD in that scenario would slow me down. I could focus on implementing the logic, without having to bother about unit tests.

Some of you may argue that if I would think interface desing through before I started coding, then TDD would work well in that scenario. That's true, but I'm not convinced that it is always best approach. Many times when I used TDD, I found myself changing whole design, based on things I learned when implementing the code. Sometimes I forgot some requirements, sometimes I found more efficient or elegant solution, sometimes I discovered some facts about code or database schema that I didn't know about. That's reality of day to day software engineering. Because of that I think that in many scenarios, starting with simple, transient design and letting it evolve is much better approach, than forcing oneself to come up with full and perfect design beforehand.

To sum up, I don't think TDD is silver bullet solution that should be always used. There are scenarios where it works great, but in some of them it clearly isn't the best process to follow. Because of that I don't think software engineers should be forced to use TDD, assuming that they always write tests for code they write.