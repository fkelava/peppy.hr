---
layout: post
title:  "C#: Extension methods over inline arrays"
date:   2025-09-07 20:02:47 +0200
---
## Where we came from: Fixed-size buffers
In .NET native interop cases, one often has a need to model
fixed-size arrays. Those familiar with the subject matter will know
of the [fixed size buffers](https://github.com/dotnet/csharpstandard/blob/draft-v8/standard/unsafe-code.md#238-fixed-size-buffers) 
feature introduced to deal with this exact class of problem.

One can declare a struct as follows and get expected behavior.
{% highlight csharp %}
{% raw %}
[StructLayout(LayoutKind.Explicit, Pack = 4, Size = 0x2150)]
public unsafe struct Btl {
    [FieldOffset(0x198A)] public fixed byte field_name[8];
    [FieldOffset(0x1FC5)] public fixed byte frontline[3];
    [FieldOffset(0x1FD3)] public fixed byte backline[4];
}
{% endraw %}
{% endhighlight %}

However, fixed size buffers have a few unfortunate downsides. One
is that they require an `unsafe` context. Another is that they sometimes
require the use of the `fixed` statement to pin the array while
indexing it, to prevent the garbage collector from moving it.

Consider the following sample from the .NET examples:
{% highlight csharp %}
{% raw %}
internal unsafe struct Buffer
{
    public fixed char fixedBuffer[128];
}

internal unsafe class Example
{
    public Buffer buffer = default;
}

private static void AccessEmbeddedArray()
{
    var example = new Example();

    unsafe
    {
        // Pin the buffer to a fixed location in memory.
        fixed (char* charPtr = example.buffer.fixedBuffer)
        {
            *charPtr = 'A';
        }
        // Access safely through the index:
        char c = example.buffer.fixedBuffer[0];
        Console.WriteLine(c);

        // Modify through the index:
        example.buffer.fixedBuffer[0] = 'B';
        Console.WriteLine(example.buffer.fixedBuffer[0]);
    }
}
{% endraw %}
{% endhighlight %}

This adds up to a fairly verbose way to model and consume such structures.

## Where we're headed; inline arrays
As an improvement to this, .NET 8 and C# 12 introduced the
[`[InlineArray]` attribute](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-12.0/inline-arrays),
which offers a number of compelling benefits:
- Inline arrays are not restricted to certain primitives.
- Inline arrays of `T` have implicit convertibility to `Span<T>`.
- Inline arrays do not require an unsafe context or a `fixed` accessor.
- Inline arrays have compile-time bounds checking.

One can model the following:
{% highlight csharp %}
{% raw %}
[InlineArray(20)]
public struct ByteArray20 {
    private byte _b;
}

[InlineArray(40)]
public struct ByteArray40 {
    private byte _b;
}

public struct CHRDATA {
    public ByteArray20 chr_name;
    public ByteArray20 chr_name_extended;
    
    public void TestMethod() {
        foreach (byte b in chr_name) 
        // ...
    }
}
{% endraw %}
{% endhighlight %}

and trivially `foreach` over the contents of these arrays. Success!

But we're not out of the woods quite yet.

## Operating over inline arrays generically
The size of an inline array is a compile-time constant. For each
type T and size S, one needs to define an `[InlineArray(S)]` of `T`.

However, you may very well wish to operate over any inline array of `T`
generically. How are we to do that? `[InlineArray]` is a special
attribute, not an interface. We cannot use that as a generic constraint
nor can we define an extension method over it.

A few paragraphs above, we mentioned that inline arrays of `T` are 
implicitly convertible to `Span<T>`. And it is legal to define
extension methods over `Span<T>` or a concrete specialization like `Span<byte>`.

So why not take advantage of that? Let's try the following.
{% highlight csharp %}
{% raw %}
[InlineArray(40)]
public struct ByteArray40 {
    private byte _b;
}
[InlineArray(20)]
public struct ByteArray20 {
    private byte _b;
}

public static class Extensions {
    public static string decode_name(this Span<byte> name) {
        // ...
    }
}

public struct CHRDATA {
    public ByteArray20 chr_name;
    public ByteArray40 chr_name_extended;

    public void test() {
        string str_chr_name          = chr_name.decode_name();
        string str_chr_name_extended = chr_name_extended.decode_name(); 
    }
}
{% endraw %}
{% endhighlight %}

It won't do. This extension method will not appear over our inline arrays.
While they are _implicitly convertible_ to `Span<byte>`, they _aren't_ `Span<byte>`.

What now? Can we explicitly express or induce that conversion? As it were, yes.

## Ref-safety for dummies
If we were in a method, we could obtain the `Span<T>` as follows:
{% highlight csharp %}
{% raw %}
public struct CHRDATA {
    public ByteArray20 chr_name;
    public ByteArray40 chr_name_extended;

    public void test() {
        Span<byte> chr_name_span = chr_name;
    }
}
{% endraw %}
{% endhighlight %}

We have to trigger this conversion _before_ a call to our extension method, however.
Taking the obvious route, we augment the inline array definition with the following member:
{% highlight csharp %}
{% raw %}
[InlineArray(40)]
public struct ByteArray40 {
    private byte _b;
    public Span<byte> as_span() => this;
}
{% endraw %}
{% endhighlight %}

The compiler does not permit this. In C#, `this` for `struct` instance methods 
is ["implicitly scoped"](https://learn.microsoft.com/en-us/dotnet/api/system.diagnostics.codeanalysis.unscopedrefattribute?view=net-9.0#remarks) -
that is, it has a lifetime or 'scope' of `as_span()`, and cannot escape it.

Analogously, in `test()` above the lifetime or 'scope' of `chr_name_span`
is the method itself. It cannot escape `test()`, but it can be used as a method local.

What we need is a way to pinky-promise to the compiler that we will do just that;
only use the obtained `Span<byte>` as a method local in `test()`, which is safe.

This is done using the [`\[UnscopedRef\]` attribute](https://learn.microsoft.com/en-us/dotnet/api/system.diagnostics.codeanalysis.unscopedrefattribute?view=net-9.0).

We arrive at the final syntax:
{% highlight csharp %}
{% raw %}
[InlineArray(40)]
public struct ByteArray40 {
    private byte _b;
    [UnscopedRef] public Span<byte> as_span() => this;
}
{% endraw %}
{% endhighlight %}

and can thus invoke:
{% highlight csharp %}
{% raw %}
public static class Extensions {
    public static string decode_name(this Span<byte> name) {
        // ...
    }
}

public struct CHRDATA {
    public ByteArray20 chr_name;
    public ByteArray40 chr_name_extended;

    public void test() {
        string chr_name_extended_str = chr_name_extended.as_span().decode_name(); 
    }
}
{% endraw %}
{% endhighlight %}

Which brings a day of debugging to a happy ending.
