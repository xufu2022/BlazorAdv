# JavaScript Interop

an instance of IJsRuntime has to be injected into the Razor component

```csharp
@inject IJSRuntime jsRuntime
await jsRuntime.InvokeVoidAsync("myscope.methodName");
```

There are two different methods we can use to call JavaScript:
-   • InvokeVoidAsync, which calls JavaScript, but doesn’t expect a return value
-   • InvokeAsync<T>, which calls JavaScript and expects a return value of type T

We can also send in parameters to our JavaScript method if we want. We also need to refer to JavaScript, and JavaScript must be stored in the wwwroot folder.

## JavaScript Isolation

Isolated JavaScript can be stored in the wwwroot folder, but since an update in .NET 6, we can add them in the same way we add isolated CSS. Add them to your component’s folder and name it js at the end (mycomponent.razor.js)

```js
export function showConfirm(message) {
    return confirm(message);
}
```

```csharp
IJSObjectReference jsmodule;
private async Task<bool> ShouldDelete()
{
    jsmodule = await jsRuntime.InvokeAsync<IJSObjectReference>("import", "/_content/Components/RazorComponents/ItemList.razor.js");
    return await jsmodule.InvokeAsync<bool> ("showConfirm", "Are you sure?");
}
```

IJSObjectReference is a reference to the specific script that we will import further down.
It has access to the exported methods in our JavaScript, and nothing else.

```html
<td><button class="btn btn-danger" @onclick="@(async ()=>{ if (await ShouldDelete()) { await DeleteEvent.InvokeAsync(item); }})">Delete</button></td>
```

## JavaScript to .NET

There are three ways of doing a callback from JavaScript to .NET code:
- • A static .NET method call
- • An instance method call
- • A component instance method call

### Static .NET method cal

```cs
[JSInvokable]
public static Task<int[]> ReturnArrayAsync()
{
    return Task.FromResult(new int[] { 1, 2, 3 });
}
```

In the JavaScript file, we can call that function using the following code:

```js
// The DotNet object comes from the Blazor.js or blazor.server.js file.
// BlazorWebAssemblySample is the name of the assembly, and ReturnArrayAsync is the name of the static .NET function.
DotNet.invokeMethodAsync('BlazorWebAssemblySample', 'ReturnArrayAsync')
.then(data => {
        data.push(4);
        console.log(data);
        }
    );
```

It is also possible to specify the name of the function in the JSInvokeable attribute if we don’t want it to be the same as the method name like this:
```cs
[JSInvokable("DifferentMethodName")]
...
```

### Instance method call

need to pass an instance of the .NET object to call it (this is the method that Blazm.Bluetooth is using)

```cs
// need a class that will handle the method call:
using Microsoft.JSInterop;
public class HelloHelper
{
    public HelloHelper(string name){
        Name = name;
    }

    public string Name { get; set; }

    [JSInvokable]
    public string SayHello() => $"Hello, {Name}!";
}
```

```js
// need JavaScript that can call the .NET function:
// we are using the export syntax, and we export a function called sayHello, which takes an instance of DotNetObjectReference called dotnetHelper.
export function sayHello (hellohelperref) {
    return hellohelperref.invokeMethodAsync('SayHello').then(r => console.log(r));
}
```

```cs
@page "/interop"
@inject IJSRuntime jsRuntime
@implements IDisposable

<button type="button" class="btn btn-primary" @onclick="async ()=> {await TriggerNetInstanceMethod(); }"> Trigger .NET instance method HelloHelper.SayHello </button>

@code {
        private DotNetObjectReference<HelloHelper> objRef;
        IJSObjectReference jsmodule;
    
        public async ValueTask<string> TriggerNetInstanceMethod() 
        {
            objRef = DotNetObjectReference.Create(new HelloHelper("BruceWayne"));
            jsmodule = await jsRuntime.InvokeAsync<IJSObjectReference>("import", "/_content/MyBlog.Shared/Interop.razor.js");
            return await jsmodule.InvokeAsync<string>("sayHello", objRef);
        }

// To avoid any memory leaks, we also have to make sure to implement IDiposable, and toward the bottom of the file, we make sure to dispose of the DotNetObjectReference instance
        public void Dispose()
        {
            objRef?.Dispose();
        }
    }
```

There are two more ways that we can use to call .NET functions from JavaScript. We can call a method on a component instance where we can trigger an action, and it is not a recommended approach for Blazor Server. We can also call a method on a component instance by using a helper class

## Implementing an existing JavaScript library

Blazor is pretty smart when syncing the DOM and render tree, but try to avoid manipulating the DOM. If we need to use JavaScript for something, make sure to put a tag outside the manipulation area, and Blazor will then keep track of that tag and not think about what is inside the tag

```html
<div data-js-zone>
    <h1>Hello, world!</h1>
    <p>Some content here.</p>
</div>
```
```js
const jsZone = document.querySelector('[data-js-zone]');
jsZone.classList.add('active');
```

By following this approach, you can use JavaScript in your Blazor applications without risking conflicts or breaking Blazor's automatic synchronization of the DOM and the render tree.

### JavaScript interop in WebAssembly

We have had direct access to the JSRuntime using the IJSInProcessRuntime and IJSUnmarshalledRuntime. But with .NET 7, both are now obsolete, and we have gotten a nicer syntax

To be able to use these features, we need to enable them in the project file by enabling AllowUnsafeBlocks.

```xml
<PropertyGroup>
    <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
</PropertyGroup>
```
## .NET to JavaScript

```cs
@using System.Runtime.InteropServices.JavaScript

<button @onclick="ShowAlert">Show Alert</button>

@code {
        protected async void ShowAlert()
        {
            ShowAlert("Hello from .NET");
        }
        protected override async Task OnInitializedAsync()
        {
            //using JSHost to import the JavaScript and give it the name "nettojs"
            await JSHost.ImportAsync("nettojs", "../JSInteropSamples/NetToJS.razor.js");
        }
    }
```

code-behind
```cs
using System.Runtime.InteropServices.JavaScript;
namespace BlazorWebAssembly.Client.JSInteropSamples;
public partial class NetToJS
{
    // add a JSImport attribute to a method, which will automatically be mapped to the JavaScript call.
    [JSImport("showAlert", "nettojs")]
    internal static partial string ShowAlert(string message);
}
```

## JavaScript to .NET

When calling a .NET method from JavaScript, a new attribute makes that possible called JSExport.


```html
@page "/jstostaticnetwasm"
@using System.Runtime.InteropServices.JavaScript
<h3>This is a demo how to call .NET from JavaScript</h3>
<button @onclick="ShowMessage">Show alert with message</button>
@code {
    protected override async Task OnInitializedAsync()
    {
        await JSHost.ImportAsync("jstonet", "../JSInteropSamples/JSToStaticNET.razor.js");
    }
}
```

```cs
// code-behind
namespace BlazorWebAssembly.Client.JSInteropSamples;
[SupportedOSPlatform("browser")]
public partial class JSToStaticNET
{
    [JSExport]
    internal static string GetAMessageFromNET()
    {
        return "This is a message from .NET";
    }

    [JSImport("showMessage", "jstonet")]
        internal static partial void ShowMessage();
    }
```

Here we are using the SupportedOSPlatform attribute to ensure that this code can only run on a browser.

```js
export async function setMessage() {
    const { getAssemblyExports } = await globalThis.getDotnetRuntime(0);
    var exports = await getAssemblyExports("BlazorWebAssembly.Client.dll");
    alert(exports.BlazorWebAssembly.Client.JSInteropSamples.JSToStaticNET.GetAMessageFromNET());
}
export async function showMessage() {
    await setMessage();
}
```