# Creating Advanced Blazor Components

## One-way binding vs Two-way binding

Two-way binding binds values in both directions, and our Counter component will be able to notify our parent component of any changes

```cs
[Parameter]
public EventCallback<int> CurrentCountChanged { get; set; }
private async Task IncrementCount()
{
    CurrentCount += IncrementAmount;
    await CurrentCountChanged.InvokeAsync(CurrentCount);
}
```

```html
<CounterWithParameterAndEvent IncrementAmount="@incrementamount" @bind-CurrentCount="currentcount" @bind-CurrentCount:event="CurrentCountChanged"/>
```
By using :event, we can tell Blazor exactly what event we want to use; in this case, the CurrentCountChanged event

## Actions and EventCallback

To communicate changes, we can use EventCallback, as shown in the Two-way binding section.
EventCallback<T> differs a bit from what we might be used to in .NET. EventCallback<T> is a class that is specially made for Blazor to be able to have the event callback exposed as a parameter for the component

In .NET, in general, you can add multiple listeners to an event (multi-cast), but with EventCallback<T>, you will only be able to add one listener (single-cast).

.NET events use classes, and EventCallback uses structs. This means that in Blazor, we don’t have to perform a null check before calling EventCallback because a struct cannot be null

EventCallback is asynchronous and can be awaited. When EventCallback has been called, Blazor will automatically execute StateHasChanged on the consuming component to ensure the component updates (if it needs to be updated)

## Using RenderFragment
To make our components even more reusable, we can supply them with a piece of Razor syntax. In Blazor, you can specify RenderFragment, which is a fragment of Razor syntax that you can execute and show
## Component virtualization

```html
<ul>
<Virtualize ItemsProvider="LoadPosts" Context="p">
<li><a href="/Post/@p.Id">@p.Title</a></li>
</Virtualize>
</ul>
```
use the Virtualize component and add a render fragment that shows how each item should be rendered. The Virtualize component uses RenderFragment<T>,
which, by default, will send in an item of type T to the render fragment. In the case of the Virtualize component, the object will be one blog post (since items are List<T> of blog posts). 

We access each post with the variable named context. However, we can use the Context property on the Virtualize component to specify another name, so instead of context, we are now using p.
```cs
public int totalBlogposts { get; set; }
private async ValueTask<ItemsProviderResult<BlogPost>>

LoadPosts(ItemsProviderRequest request)
{
    if (totalBlogposts == 0)
    {
    totalBlogposts = await _api.GetBlogPostCountAsync();
    }
    var numblogposts = Math.Min(request.Count, totalBlogposts - request.StartIndex);
    var blogposts= await _api.
    GetBlogPostsAsync(numblogposts,request.StartIndex);
    return new ItemsProviderResult<BlogPost>(blogposts, totalBlogposts);
}
```

## Error boundaries

```html
<ErrorBoundary>
    <ComponentWithError />
</ErrorBoundary>
```
```html
<ErrorBoundary Context=ex>
    <ChildContent>
        <ComponentWithError />
    </ChildContent>
    <ErrorContent>
        <h1 style="color: red;">Oops... something broke @ex.Message</h1>
    </ErrorContent>
</ErrorBoundary>
```