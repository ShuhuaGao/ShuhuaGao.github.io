---
layout: post
title: "P/Invoke in C# for struct/structure"
comments: true
description: "How to marshal a structure in C# "
keywords: "P/Invoke, C#, Dll, marshal, struct/structure"
---

For some legacy or third-party C/C++ functions, it is the common case that we only have the dynamic library (.dll on Windows) and have no access to the source codes. In this case, if we want to use these functions in our .net projects using C#/VB.net, the platform invoke (P/Invoke) will be the only choice. For primitve types like `int` and `double`, it is quite easy: just declare the same types in the managed code as a prototype. However, it is sometimes awkward to define a proper prototypes for more complicated types, especially `struct` in C/C++. About marshaling structures and classes in mangaged code, many detailes can be found in the official document [_Marshaling Classes, Structures, and Unions_](https://docs.microsoft.com/en-us/dotnet/framework/interop/marshaling-classes-structures-and-unions). Here, I mainly want to emphasize one special case and demonstrate what we should do if there are **embedded arrays** in a `struct`.
