---
title: ".NET MAUI - Common memory leak pitfalls "
date: 2025-02-28T16:30:03+00:00
# weight: 1
# aliases: ["/first"]
tags: [".NET", "MAUI", "Memory Leak", "Memory", "Leak", "Android"]
author: "Lauri Kattelus"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "How to avoid falling into the memory leak pitfalls while working with .NET MAUI"
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "<image path/url>" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
editPost:
    URL: "https://github.com/laukatti/kattelus-net/"
    Text: "Suggest Changes" # edit text
    appendFilePath: false # to append file path to Edit link
---
# .NET MAUI - Common memory leak pitfalls

## Foreword
 I have been working with Xamarin/MAUI for some years now. The premise sounds great, develop native multiplatform applications using .NET Stack.
 In my experience with long running MAUI Android/iOS applications, I have found that memory is surprisingly easily leaked.
 I’ve primarily used `dotnet-dsrouter, dotnet-gcdump`, and `MemoryToolkit.Maui` to profile memory usage.
 - [https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-dsrouter](dotnet-dsrouter)
 - [https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-gcdump](dotnet-gcdump)
 - [https://github.com/AdamEssenmacher/MemoryToolkit.Maui](MemoryToolkit.Maui)

The first two tools, dotnet-dsrouter, dotnet-gcdump, are used to capture memory dumps from Android devices. The last one, MemoryToolkit.Maui simplifies memory management of views. However, MemoryToolkit.Maui may occasionally report non-existent memory leaks if it prematurely assumes a page should have been disposed. Because of this, it’s still crucial to collect real memory dumps using `dotnet-dsrouter` and `dotnet-gcdump` to investigate potential leaks.

Even though MAUI still has some bugs, I enjoy working with it overall.

Here are some easy-to-leak memory scenarios that I’ve learned about while profiling memory leaks on MAUI.


## Dependency injection
When using dependency injection with `Microsoft.Extensions.DependencyInjection`, avoid implementing `IDisposable` on your transient classes. In this DI framework, transient `IDisposable` objects are not automatically disposed, leading to potential memory leaks.
A good approach is simply not to implement `IDisposable` on your transient classes. Instead, provide a `CleanUp()` method that manually unsubscribes from events, stops timers, and clears any references that would otherwise keep the object alive. Since the DI container won’t automatically dispose of transient services, calling `CleanUp()` lets you explicitly manage whatever resources or subscriptions your class holds. This approach effectively handles everything you’d normally put in `Dispose()`, without relying on disposal semantics that just don’t happen for transient objects in `Microsoft.Extensions.DependencyInjection`.

```
public class ViewModelBase{
    public virtual Task OnNavigatedToAsync()
    {
        return Task.CompletedTask;
    }

    public virtual Task OnNavigatedFromAsync()
    {
        return Task.CompletedTask;
    }

    public virtual void CleanUp()
    {
        //Remove subscriptions, stop timers here
    }
}
```
Having a base class like this makes it easy to call `CleanUp()` on a ViewModel when you pop the page from the `NavigationStack`.
## Views/Pages/XAML
I’ve found that setting the `BindingContext` to null when navigating away from a page significantly helps with memory management. It also makes it simpler to clean up page state (animations, timers, subscriptions) right before removal.

By setting the `BindingContext` to null in your navigation logic, you can override `On`BindingContext`Changed()` and stop animations or unsubscribe from events.

e.g having this kind of Navigation logic when popping a page from stack can help you a lot.

```
//Navigation logic handling class

public async Task PopAsync(){
    // get view models and views
    await NavigationStack.PopAsync();
    view.`BindingContext` = null;
    viewModel.CleanUp();
}
```
In the code-behind, you can observe when the page is no longer needed (often after `PopAsync()` or a removal from the `NavigationStack`) to perform cleanup actions like stopping timers.

```
	override On`BindingContext`Changed()
	{
		if (`BindingContext` == null)
		{
			//Stop timers and animations and other clean up here
		}
	}
```
Below are elements I’ve found that can easily leak memory if not handled correctly:
- Animations
    - Be sure to stop animation before removing page from stack.
- Timers
    - Don't use anonymous delegates to handle timer ticks, instead create event handler and unsubscribe from it before removing page from stack.
- CarouselControl handler
    - As the time of writing this blog CarouselControl is not disposed correctly when page is disposed. Developer needs to manually Disconnect view handler from it with `carouselView.Handler.DisconnectHandler()`.
Not handling these situations correctly, may cause the XAML page to stay on memory until application is closed.

## WeakEventManager
When you have a long-lived publisher object and short-lived subscriber objects, consider using a WeakEventManager for events. This pattern adds a bit of overhead but significantly reduces the risk of memory leaks.

## TLDR

- **DI & Transient Classes:** Avoid `IDisposable` on transient services. Use a custom `CleanUp()` method instead.
- **Set `BindingContext` to Null:** When navigating away from a page, set `BindingContext = null` to release references.
- **Stop Animations & Timers:** Always stop any ongoing animations and unsubscribe from timer events before removing pages.
- **Check CarouselControl:** As of now, this control may not dispose properly—disconnect its handler manually.
- **Consider WeakEventManager:** Use weak event patterns for long-lived publishers and short-lived subscribers to avoid leaks.
