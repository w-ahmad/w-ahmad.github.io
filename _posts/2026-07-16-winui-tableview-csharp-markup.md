---
layout: single
classes: wide
title: "WinUI.TableView with C# Markup"
date: 2026-07-16
redirect_from:
  - /winui/tableview/winui-tableview-csharp-markup/
tags:
  - WinUI
  - TableView
  - Uno Platform
  - C# Markup
---

You're building a nice [Uno Platform](https://platform.uno/) app with [C# Markup](https://platform.uno/docs/articles/external/uno.extensions/doc/Learn/Markup/Overview.html), you drop in [WinUI.TableView](https://w-ahmad.dev/WinUI.TableView), wire up `ItemsSource` the usual way — and the app crashes.

```
InvalidOperationException: Setting this property directly is not allowed. Use TableView.ItemsSource instead.
```

Confusing, right? Your code looks correct. The binding syntax is fine. So what's going on?

---

## The Problem

C# Markup gives you a clean fluent API:

```csharp
new TableView()
    .ItemsSource(() => ViewModel.People);
```

Under the hood though, C# Markup resolves `ItemsSource` through `ItemsControl` — because that's where the extension methods are defined. So it ends up setting:

```csharp
ItemsControl.ItemsSourceProperty
```

That's normally fine for standard controls. But **WinUI.TableView** hides the base property:

```csharp
public new object? ItemsSource { get; set; }
```

We do this to support features like sorting and filtering, which require intercepting every assignment. Along with that, we register our own dependency property:

```csharp
public new static DependencyProperty ItemsSourceProperty
```

So you end up with two completely separate DPs living side by side:

* `ItemsControl.ItemsSourceProperty`
* `TableView.ItemsSourceProperty`

C# Markup targets the first one. `TableView` guards it and throws. That's your crash.

---

## Why This Happens

Neither C# Markup nor TableView is doing something wrong here.

This is just **property hiding in .NET** — when you declare `public new`, you're not overriding anything. You're creating a brand new member that has no relationship to the base class version. C# Markup has no way of knowing that `TableView` wants a different DP, so it falls back to the one it knows about.

---

## The Fix

Because `TableView.ItemsSource` is of type `object`, the built-in C# Markup extensions don't apply. We need custom ones that target `TableView.ItemsSourceProperty` directly, covering all the common usage patterns:

```csharp
public static class TableViewMarkupExtensions
{
    // 1️⃣ Direct assignment
    public static T ItemsSource<T>(this T element, object itemsSource) where T : TableView
    {
        element.ItemsSource = itemsSource;
        return element;
    }

    // 2️⃣ Binding via DependencyPropertyBuilder
    public static T ItemsSource<T>(this T element, Action<IDependencyPropertyBuilder<object>> configureProperty) where T : TableView
    {
        var instance = DependencyPropertyBuilder<object>.Instance;
        configureProperty(instance);
        instance.SetBinding(element, ItemsControl.ItemsSourceProperty, nameof(TableView.ItemsSource));
        return element;
    }

    // 3️⃣ Strongly-typed binding
    public static T ItemsSource<T, TSource>(this T element, Func<TSource> propertyBinding, [CallerArgumentExpression(nameof(propertyBinding))] string? propertyBindingExpression = null) where T : TableView
    {
        return ItemsSource(element, delegate (IDependencyPropertyBuilder<object> _)
        {
            _.Binding(propertyBinding, propertyBindingExpression);
        });
    }

    // 4️⃣ Strongly-typed binding with conversion
    public static T ItemsSource<T, TSource>(this T element, Func<TSource> propertyBinding, Func<TSource, object> convertDelegate, [CallerArgumentExpression("propertyBinding")] string? propertyBindingExpression = null) where T : TableView
    {
        return ItemsSource(element, delegate (IDependencyPropertyBuilder<object> _)
        {
            _.Binding(propertyBinding, propertyBindingExpression).Convert(convertDelegate);
        });
    }
}
```

---

## Now It Works

With these extensions in place, both styles work exactly as you'd expect:

```csharp
new TableView()
    .ItemsSource(People);
```

```csharp
new TableView()
    .ItemsSource(() => vm.People);
```

The binding now correctly targets `TableView.ItemsSourceProperty` and the crash is gone.

---
