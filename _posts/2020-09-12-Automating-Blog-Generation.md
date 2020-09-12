---
layout: post
title: Programmatic blog generation
subtitle: 700 lines of code to produce 20,000 lines of blog
tags: [arm64, intrinsics, automation]
---

## Motivation

Currently, the Microsoft docs for methods in [System.Runtime.Intrinsics.Arm.AdvSimd](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.intrinsics.arm.advsimd?view=net-5.0) and [System.Runtime.Intrinsics.Arm.AdvSimd.Arm64](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.intrinsics.arm.advsimd?view=net-5.0) class doesn’t have much description and just have the ARM instruction name that it uses. A .NET developer who is not familiar with these APIs will first check the ARM instruction for the .NET API and then search for that instruction in the ARM manual to understand what it does. Sometimes, the description in the ARM manual can be better understood with actual input/output examples. Hence the developer might then write sample method using the .NET API giving it various inputs and checking the output it produces. I have been through that process when I [optimized IndexOf() APIs](https://github.com/dotnet/runtime/pull/37624) with ARM64 hardware intrinsics APIs and pretty sure many others will have to go through too.

As a result, I decided to tie the ARM64 intrinsic APIs in .NET with the description of corresponding instructions present in the ARM manual. However, it was not an easy task to do since there are around 360 ARM64 intrinsic APIs that were added. Manually writing about them would have taken me months to do. I wanted to do this smartly without taking much time. So, I decided to programmatically generate as much content as possible with little manual editing.

In this blog, I will describe about the  **~700** lines of C# project, [AdvSimdBlogGen](https://github.com/kunalspathak/AdvSimdBlogGen), that I wrote to generate **24,000** lines of C# code that further generated  **20,300** lines of blog contents. I am planning to publish the generated blog in next few weeks. 

## Task

For each of 360 APIs under `System.Runtime.Intrinsics.Arm.AdvSimd` and `System.Runtime.Intrinsics.Arm.AdvSimd.Arm64`, here is the template that I wanted to come up with:

* Method signature of the API.
* C# sample code that invokes the API.
* Print the parameters passed to the API.
* Print the result returned by the API.
* List all the overloads of the API.
* Reference to Microsoft and ARM docs for the API.
* JIT disassembly generated for the API.

As you can see, it would have been a lot of efforts to hand-write all that information for 360 APIs.

## Steps

Here are the steps I took to automate the blog contents for the hardware intrinsic APIs.

### Roslyn to extract method information

First task was to extract all the method names that need to be documented. I used [Roslyn APIs](https://docs.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/get-started/syntax-analysis#traversing-trees) to parse [AdvSimd.cs](https://github.com/dotnet/runtime/blob/65d294d8b2726a9344b80c395972631d3c564f36/src/libraries/System.Private.CoreLib/src/System/Runtime/Intrinsics/Arm/AdvSimd.cs) file and extract all the methods declared in the file. Since there were lot of overloaded methods for most of the APIs and I wanted to group the overloaded methods in my blog, I grouped the extracted method information by method name. The method information contained the signature of the method, list of parameters that it takes along with contents of summary docs. At the end of this step, I had a map of method name to a list of overloaded methods along with the method information for each overload.

### Generate C# code that will produce blog contents

With the method information I collected for all APIs using Roslyn, it was not sufficient to produce the blog post of API documentation that I wanted. As you can see in [Task section](#task), I also wanted to mention example input and output for each API along with the JIT disassembly produced for them. This can be possible only if I execute the sample code that I have included in my blog for an API. To do this, I decided to programmatically generate C# code which when executed will produce the blog contents with all the information I want to include. Naturally, in order to do that, the generated C# code should also contain the method definitions of the sample code of my blog so that I can execute them to print the parameters, result and disassembly code for each API. For example, for `Abs()` method, I wanted to have a sample code of something like this:

```csharp
private Vector64<ushort> AbsTest(Vector64<short> value)
{
  return AdvSimd.Abs(value);
}
// value = <-11, -12, -13, 14>
// Result = <11, 12, 13, 14>

// <<JIT disassembly code>>
```

To do that, my generated C# code should not only print this method and parameter values, but also execute it to produce the result output and disassembly code. Otherwise, I would have to hand-type these examples, pass them parameters, copy its values along with return result and paste it in my blog. Programmatically doing it was easy. From the method information, it was easy to generate the sample `*Test()` method such that it calls the API that I am describing. Thus, roughly, I ended up generating the following C# code for each API.

```csharp
[MethodImpl(MethodImplOptions.NoInlining)]
private static  Vector64<ushort> AbsTest(Vector64<short> value)
{
  return AdvSimd.Abs(value);
}

// Sample *Test() methods for other APIs

public static void Main(string[] args)
{
    string apiResult;
    try 
    {
        apiResult = AbsTest(Vector64short_0).ToString();
    } catch (Exception) {
        apiResult = "TODO";
    }

    Console.WriteLine(
@"

### 1. Abs

__`Vector64<ushort> Abs(Vector64<short> value)`__

{0}

~~~csharp
private Vector64<ushort> AbsTest(Vector64<short> value)
{
return AdvSimd.Abs(value);
}
// value = {1}
// Result = {2}

~~~
", "Performs 'Abs' operation.", Vector64short_0, apiResult);

// Code to print other APIs information
}
```
<p/>

### Arguments for the sample code

The next challenge was to figure out arguments to be passed to the sample `*Test()` methods. For 360 methods I wanted a way to generate a list of local variables of relevant type such that I can just pass them while invoking the `*Test()` methods. In the process of generating the C# code, I also tracked the unique parameter types that these methods take. Since most of them operate on `Vector64<T>` and `Vector128<T>`, with different types of `T`, there were in all approximately 30 unique types. So, I manually created 30 local variables for those unique types that I passed as an argument to invoke the API. For APIs, that contained multiple parameters of same type, I created different local variable for each of them so I can pass different value to them. For example, `Vector64<float>` is the parameter type taken by `AbsoluteCompareGreaterThan(Vector64<float> left, Vector64<float> right)` and `AbsoluteCompareGreaterThanOrEqual(Vector64<float> left, Vector64<float> right)` and several others. Based on the parameter type, I created 2 local variables (each having different value) for these methods, that I passed later as arguments to them.

```csharp
// ...
private static Vector64<float> Vector64float_0 = Vector64.Create((float)11.5f, 12.5f);
private static Vector64<float> Vector64float_1 = Vector64.Create((float)21.5f, 22.5f);
// ...
// Other local variables

public static void Main(string[] args) 
{
    // Other code...
    AbsoluteCompareGreaterThanTest(Vector64float_0, Vector64float_1);
    // ...
    AbsoluteCompareGreaterThanOrEqual(Vector64float_0, Vector64float_1);
}
```
<p/>

### Generating the JIT disassembly

In the generated C# code, so far, I was able to invoke the sample code, get its result and print that sample code along with the parameter and result values. Next, I wanted to print the disassembly of the sample code. I wanted to print the disassembly **after** I print the sample code so a person not interested in seeing the disassembly can just ignore it. In checked build of RyuJIT, you can print the JIT disassembly of a method by setting environment variable `COMPLUS_JitDisasm=<method_name>`. The JIT code is generated, printed and then the method was executed. So, in my case, the disassembly would get printed before the sample code get printed. If you see the sample code in previous section, `AbsTest()` is called first which means the JIT disassembly of `AbsTest()` would get printed first before I print the portion inside `Console.WriteLine()`. On the contrary, I wanted to invoke `AbsTest()` before `Console.WriteLine()` so I can embed the return result of `AbsTest()`. This became chicken and egg problem. To solve this, I ended up creating duplicate method definitions of all `*Test()` methods named `Dummy_*Test()` methods and sole objective of those methods was to produce result of the API that I can print inside the `Console.WriteLine()` section. Once I do that, I would invoke `*Test()` method to print its disassembly. Here is what my updated generated C# code looked like:

```csharp
[MethodImpl(MethodImplOptions.NoInlining)]
private static  Vector64<ushort> AbsTest(Vector64<short> value)
{
  return AdvSimd.Abs(value);
}

private static  Vector64<ushort> Dummy_AbsTest(Vector64<short> value)
{
  return AdvSimd.Abs(value);
}

// Other *Test() and Dummy_*Test() methods

public static void Main(string[] args)
{
    string apiResult;
    try 
    {
        apiResult = Dummy_AbsTest(Vector64short_0);
    } catch (Exception) {
        apiResult = "TODO";
    }

    Console.WriteLine(
@"

### 1. Abs

__`Vector64<ushort> Abs(Vector64<short> value)`__

{0}

~~~csharp
private Vector64<ushort> AbsTest(Vector64<short> value)
{
return AdvSimd.Abs(value);
}
// value = {1}
// Result = {2}
~~~

Generated disassembly:

~~~asm

", "Performs 'Abs' operation.", Vector64short_0, apiResult);

// To print disassembly of AbsTest()
try {
  AbsTest(Vector64short_0);
} catch {}

Console.WriteLine("~~~");
```

<p/>

### Storing static information in a csv

At this point, I had working C# code, that generated another C# code which when compiled and executed on ARM machine would produce a huge .md file containing all the information I wanted to include for each API. The only task remaining was to add the API description for each API and verifying the parameter values and result explains what the API does. I started hand editing the generated .md file while cross checking the parameter values and result. Sometimes, if the parameters were not appropriate, I changed the values to make the API description more understandable. After going through few APIs, I soon realized that I might have a need in future to update the template of my generated blog content for number of reasons. If I want to add some more information for each API or if I want to try a new version of RyuJIT that produce better JIT disassembly or something else. If I regenerate the blog contents, all my hand edited method description and parameter values will be lost which I cannot afford to do. Hence, I decided to write a `.csv` file containing method description, each parameter value and the final result. To begin with, it contained the templated description of `"Performs 'Abs()' operation."` and the default local variable values that were passed as arguments and the result that I got by executing the generated C# code. Once I had this .csv file, I tweaked the `AdvSimdBlogGen` such that in the generated C# code, it would first check if relevant value (method description, parameter or result value) exist in csv and if yes, use it while printing the sample code usage. If not, use the default value. With that the updated generated C# code included call to `GetFromCsv()` to look up the information in csv before using the default value:

```csharp
// Other Dummy_*Test() and *Test() methods

public static void Main(string[] args)
{
  // Other code related to Abs
    Console.WriteLine(
@"

### 1. Abs

__`Vector64<ushort> Abs(Vector64<short> value)`__

{0}

~~~csharp
private Vector64<ushort> AbsTest(Vector64<short> value)
{
return AdvSimd.Abs(value);
}
// value = {1}
// Result = {2}
~~~

Generated disassembly:

~~~asm

", GetFromCsv("Abs", "Performs 'Abs' operation.", 0),
   GetFromCsv("Abs", Vector64short_0, 1),
   GetFromCsv("Abs", apiResult, 2));
}
```

After this, I started editing the .csv file with method description and/or parameter, result values and the generated C# code would fetch it from .csv to generate .md file. It took 2~3 weeks to populate the method description of all the APIs.

### Miscellaneous

There were other miscellaneous things needed to polish the blog contents like reference to Microsoft and ARM docs, table of contents, deleting unneeded lines from JIT disassembly, etc. I [tweaked RyuJIT](https://github.com/kunalspathak/runtime/commit/8f2b507dd2e7fab0dec052e452a61c3784cea9ca) to not include irrelevant information in JIT disassembly. Since I did not want to publish all the API documentation in a single blog, I added splitter to split the blog contents to include N APIs / blog. This all added a need to include table of contents in each blog listing the APIs covered in it.

## Process

To give a brief overview, here was the entire process:

```cmd
|---------------------------|  1. Generate C# code     |---------------------------|
|   AdvSimdBlogGen.csproj   | --------------------->   |  AdvSimdGenerated.csproj  |
|---------------------------|                          |---------------------------|
               |                                                |
               | 4. Split the blog                              | 2. dotnet build
               V                                                V
|--------------------------|    3. Execute on ARM   |--------------------------|
|        ApiBlog.md        | <--------------------- |   AdvSimdGenerated.dll   |
|--------------------------|                        |--------------------------|
            |
            |
            V
|--------------------------|
|     ApiBlog-Part1.md     |
|     ApiBlog-Part2.md     |
|     ApiBlog-Part3.md     |
|           ...            |
|     ApiBlog-PartN.md     |
|--------------------------|
```

## Conclusion

 There was considerable amount of time I spent in thinking through the entire problem of programmatically producing the C# code and the blog content, but I think it was worth every minute. Without this, not only I would have spent more time handwriting each detail of every API, the details itself might not have been consistent. And of course, it would have been less fun!

All the source code of `AdvSimdBlogGen` project can be found here: [https://github.com/kunalspathak/AdvSimdBlogGen](https://github.com/kunalspathak/AdvSimdBlogGen).

Namaste!