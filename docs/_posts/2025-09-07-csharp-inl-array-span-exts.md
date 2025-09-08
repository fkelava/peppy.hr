---
layout: post
title:  "C#: Extension methods over inline arrays"
date:   2025-09-07 20:02:47 +0200
---
## The humble array
Some exposition is in order.

In C (and many, _many_ other languages) structures
can freely carry arrays of a given size.

The following declaration:

{% highlight c %}
{% raw %}
typedef struct CHRDATA {
    byte name[20];
    byte name_ext[40];
};
{% endraw %}
{% endhighlight %}

results, somewhat intuitively, in 60
`byte`s laid out sequentially.

In .NET, however, arrays are [reference types](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/reference-types).
Their length is not fixed at declaration time, and variables
of type `byte[]` store only a pointer to the data.

If a structure is defined as follows:

{% highlight csharp %}
{% raw %}
public struct CHRDATA {
    byte[] name;
    byte[] name_ext;
}
{% endraw %}
{% endhighlight %}

the actual layout of the struct is two pointers,
not two streams of bytes laid out sequentially in memory.

For native interop cases, it is essential that we can 
model the latter case. How can we do so? 

## Where we came from: Fixed-size buffers
Those familiar with the subject matter may think
of the [fixed size buffers](https://github.com/dotnet/csharpstandard/blob/draft-v8/standard/unsafe-code.md#238-fixed-size-buffers) 
feature.

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

## Where we're headed: inline arrays
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
    public ByteArray20 name;
    public ByteArray40 name_ext;
    
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
The size of an inline array is a compile-time constant. One needs to define 
a unique struct that is `[InlineArray(S)]` of `T` for each unique size `S` and type `T`.

However, one may wish to operate over any inline array of `T`
generically. How are we to do that? `[InlineArray]` is a special
attribute, not an interface. We cannot use that as a [generic constraint](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/generics/constraints-on-type-parameters)
nor can we define an [extension method](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/extension-methods) over it.

A few paragraphs above, we mentioned that inline arrays of `T` are 
implicitly convertible to `Span<T>`. And it is legal to define
extension methods over `Span<T>` or a concrete specialization like `Span<byte>`.

So why not take advantage of that? Let's try the following.
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

public static class Extensions {
    public static string decode_name(this Span<byte> name) {
        // ...
    }
}

public struct CHRDATA {
    public ByteArray20 name;
    public ByteArray40 name_ext;

    public void test() {
        // CS1929: best extension method overload 'decode_name' 
        // requires receiver of type Span<byte>
        string str_name     = name.decode_name();
        string str_name_ext = name_ext.decode_name();
    }
}
{% endraw %}
{% endhighlight %}

It won't do. This extension method will not appear over our inline arrays.
While they are _implicitly convertible_ to `Span<byte>`, they _aren't_ `Span<byte>`.

What now? Can we explicitly express or induce that conversion? As it were, yes.

## Ref-safety for dummies
Within `decode_name()`, we could obtain the `Span<T>` as follows:
{% highlight csharp %}
{% raw %}
public static class Extensions {
    public static string decode_name(this CHRDATA data) {
        Span<byte> name_span = name; // Valid assignment.
        // ...
    }
}

public struct CHRDATA {
    public ByteArray20 name;
    public ByteArray40 name_ext;
}
{% endraw %}
{% endhighlight %}

But to enable using `decode_name()` over any inline array of `byte`,
we have to trigger this conversion _before_ the call.
Taking the obvious route, we augment the inline array definition as follows:
{% highlight csharp %}
{% raw %}
[InlineArray(40)]
public struct ByteArray40 {
    private byte _b;
    // CS8170: Struct members cannot return 'this' 
    // or other instance members by reference.
    public Span<byte> as_span() => this;
}
{% endraw %}
{% endhighlight %}

The compiler does not permit this. In C#, `this` for `struct` instance methods 
is ["implicitly scoped"](https://learn.microsoft.com/en-us/dotnet/api/system.diagnostics.codeanalysis.unscopedrefattribute?view=net-9.0#remarks) -
that is, it has a lifetime or 'scope' of `as_span()`, and cannot escape it.

Analogously, in `decode_name()` above the lifetime or 'scope' of `name_span`
is the method itself. It cannot escape `decode_name()`, but it can be used as a method local.

What we need is a way to pinky-promise to the compiler that we will do just that;
only use the obtained `Span<byte>` as a method local in `decode_name()`, which is safe.

This is done using the [`[UnscopedRef]` attribute](https://learn.microsoft.com/en-us/dotnet/api/system.diagnostics.codeanalysis.unscopedrefattribute?view=net-9.0).

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
    public ByteArray20 name;
    public ByteArray40 name_ext;

    public void test() {
        string str_name_ext = name_ext.as_span().decode_name(); 
    }
}
{% endraw %}
{% endhighlight %}

which, in the end, allows us to take the benefits
of inline array types while still allowing us to express
generic transformations on them regardless of size.

## Finishing notes
Another benefit of inline arrays that I did not yet
address is that they can be generic.

It is valid to declare:
{% highlight csharp %}
{% raw %}
[InlineArray(S)]
public struct TArrayS<T> {
    private T _b;
    [UnscopedRef] public Span<T> as_span() => this;
}
{% endraw %}
{% endhighlight %}

and an extension `method<T>(this Span<T> span)`. This
permits generalization over both `T` and `S`.
