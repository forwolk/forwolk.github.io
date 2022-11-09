---
title: "Memory alignment fun"
date: 2018-07-17
---

Let’s start this post with a little quiz. What do you think is the size of this data structure?

```
public struct StructWithIncorrectLayout
{
public short hitpoints;
public int x;
public short attackPower;
public int y;
}
```

If you said 12 bytes, you were wrong. It’s actually 16!

To explain this phenomenon, we will have to introduce two concepts. <b>Memory alignment</b> and <b>padding</b>.

It turns out for the processor to be able to efficiently read memory, memory must be properly aligned.

A processor can usually execute read cycles only on the <a href="https://aticleworld.com/data-alignment-and-structure-padding-bytes/">addresses</a>, divisible by 4 or 8.

For instance, some processors would not even allow you to write a short into an odd address, resulting in an exception. Modern processors (excluding some old ARM processors) while tolerating <b>memory misalignment</b>, will have to issue more read instructions to be able to consume data. The processor will execute 2 read cycles using different memory offsets and then via bitwise operations present you with a final value. To avoid that there are certain requirements to data alignment of different types. For instance, <b>char</b> has to be 1-byte aligned, <b>short</b> 2-byte aligned and <b>int</b> 4-bytes aligned. Let’s bring up the example of what happens when the data is misaligned using the class defined above.

<img class="alignnone wp-image-81" src="https://forwolk.github.io/docs/assets/images/ReadUnaligned.png" alt="" width="691" height="131" sizes="(max-width: 691px) 100vw, 691px"/>

As you can see, to retrieve the value of X field, a processor has to execute two read operations on two different offsets. Because of this, the default strategy of the compilers is to sacrifice memory for the performance. Compilers will insert <b>padding</b> between elements of your struct so that any of its members can be read in a single read cycle. Knowing about this we can illustrate our original example above.

<img class="alignnone wp-image-83" src="https://forwolk.github.io/docs/assets/images/Aligned.png" alt="" width="761" height="116" sizes="(max-width: 761px) 100vw, 761px"/>

As you can see there is now padding inserted by the compiler and even though the structure occupies more space all the memory lookups will be completed in one read cycle.

C# has a way of controlling how data is laid out via <b>StructLayout</b> <a href="https://learn.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.structlayoutattribute?redirectedfrom=MSDN&view=net-7.0">attribute</a>. If the attribute is not explicitly specified, compiler behavior will be similar to <b>LayoutKind.Sequential</b> mode. It means that memory layout will be defined by the order in which fields are declared. Automatic padding will be generated for every field which is smaller than the biggest in a struct. In the case above it means the following:
1. Int is the largest type in our structure
2. It consists of 4 bytes
3. Thus, every short in our struct automatically gets plus 2 bytes.

This is how we get our 16 bytes instead of 12.

We can go down to 12 bytes and still keep the memory aligned. We just have to rewrite our struct the following way:

```
public struct StructWithCorrectLayout
{
public int x;
public int y;
public short hitpoints;
public short attackPower;
}
```

The size of the struct is now 12 bytes, as the <b>shorts</b> are coming after <b>ints</b>, forming an aligned 4 bytes value. That’s a strategy you can opt for if you want to reduce the memory footprint of your structs. Declare the fields in the descending order of their sizes.

Theoretically, this is how <b>LayoutKind.Auto</b> mode should behave but in my case, it was still making the 16 bytes size structure. Please check it out for yourself.

If you don’t care about <b>memory alignment</b> (and you’ve tested that your target ARM processors don’t care about it either) but memory itself is a constraint for you, you can try to setup padding manually, further reducing the memory footprint of your application. By specifying struct attribute this way <b>StructLayout(LayoutKind.Sequential, Pack = 1)</b>, you are saying that there should be no padding between any elements of the struct. If you had set Pack = 0, default padding would be used instead.

<h1>Present situation</h1>

Now that we’ve discussed ways of controlling memory alignment in C#, let’s take a look at the impact of using misaligned memory on the performance. According to this <a href="https://lemire.me/blog/2012/05/31/data-alignment-for-speed-myth-or-reality/">article</a>, memory alignment performance penalty is a thing of the past for x86 processors and can be experienced only in very specific cases. Intel documents describing <b>Nehalem</b> processor architecture (which hit the market in 2008),  state that misaligned memory access was heavily optimized and does not incur the performance penalty it used to.
Our own benchmarks show that this seems to be true. We’ve run tests on MacBook Pro and Samsung Galaxy S6 and saw no significant difference in operations on aligned and misaligned memory. If you are curious, please check the <a href="https://bitbucket.org/dev_blog/paddingbenchmark/src/master/">benchmarks</a> for yourself.

At the same time, there is a case when having misaligned memory results in a very poor performance. That is when you use atomic operations. In C# they are wrapped in an <a href="https://learn.microsoft.com/en-us/dotnet/api/system.threading.interlocked?redirectedfrom=MSDN&view=net-7.0">Interlocked</a> class. Our <a href="https://github.com/catstrike/cs_cmma">benchmarks</a> show that executing <a href="https://learn.microsoft.com/en-us/dotnet/api/system.threading.interlocked.exchange?redirectedfrom=MSDN&view=net-7.0#System_Threading_Interlocked_Exchange_System_Int32__System_Int32_">Interlocked.Exchange</a> on an unaligned memory may be 52 times slower than calling it on an aligned one. This happens if a variable got physically allocated between two cache lines.

Another consequence of using non-aligned memory is that the read / write operations are no longer atomic. This means that if you are reading and writing to the same variable from 2 different threads without synchronization constructs you may experience “torn reads”. This means that when you write 16 to a variable, upon reading from it you may suddenly get 18405345.

In essence:
1. If you care about memory consumption of your structs, make sure their fields are defined in the proper order.
2. If you want to reduce memory consumption even further (potentially introducing issues in your code on certain processors), make your structs ignore memory alignment requirements via special attributes.
3. If you are using atomic operations, always use aligned memory.

Benchmark sources are available <a href="https://bitbucket.org/dev_blog/paddingbenchmark/src/master/">here</a> and <a href="https://github.com/catstrike/cs_cmma">here</a>.

This article was co-authored by Anton Trukhan and Lenar Sharipov.

