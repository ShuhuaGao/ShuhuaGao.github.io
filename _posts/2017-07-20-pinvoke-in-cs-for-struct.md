---
layout: post
title: "P/Invoke in C# for struct/structure"
comments: true
description: "How to marshal a structure in C# "
keywords: "P/Invoke, C#, Dll, marshal, struct/structure"
---

For some legacy or third-party C/C++ functions, it is the common case that we only have the dynamic library (.dll on Windows) and have no access to the source codes. In this case, if we want to use these functions in our .net projects using C#/VB.net, the platform invoke (P/Invoke) will be the only choice. For primitve types like `int` and `double`, it is quite easy: just declare the same types in the managed code as a prototype. However, it is sometimes awkward to define a proper prototypes for more complicated types, especially `struct` in C/C++. About marshaling structures and classes in mangaged code, many detailes can be found in the official document [_Marshaling Classes, Structures, and Unions_](https://docs.microsoft.com/en-us/dotnet/framework/interop/marshaling-classes-structures-and-unions). Here, I mainly want to emphasize one special case and demonstrate what we should do if there are **embedded arrays** in a `struct`.
In a project about serial communicaiton with a JCM bill recycler, a dynamic library written in C is provided. However, we want to integrate it into a Windows Form application, which requires the P/Invoke of C interfaces. The key data structure for interaction with the machine is given as below.

## Core data structure

```c
enum BvModels
	{
		BvModelVega,
		BvModelOther
	};


	typedef struct BVUControl
	{		 
		int idebugMode;     // 1 enables the debug output and 0 disables it
							// by default it is 0
		int iSerialPort;    // port number
		int iDenominations; // Bill denominations to take		 
		int iDirection;     // Bill direction to accept		 
		int iBarCodeLen;    // bar code ticket characters length
							// Min is 6 and Max is 18
		int iBillValue;		// bill value
		int iMaxNoResponseInterval; // Maximum time to wait for a
									// response from host after a bill/coupon is received
					// if the DLL doesn't receive from a response , it will return the bill
		char cInfo[100];	// bar code ticket information & version info 
		BYTE failureInfo;	//BV failure code 
		char cUnitType;

		BvModels Model;
		char cCountry[4];	// Country code abbreviation
		int firmwareMajorVersion;
		char cDenomination[DENOM_MAX][16];	// Currency assignment table
							// Example: Escrow 61=7f0A01 -> cDenomination[1] = "100"
		
		//These are related to a bv with a recycler
		BOOL HasRecycler;
		int RecycleBoxCount;
		int RecycledNoteCount[BOX_MAX];			//Stores how many notes are stored in each recycler box			ex {3, 1, 0,  0, ... , 0}
		int RecycleDenom[BOX_MAX];				//Stores which denom is being recycled in each recycler box		ex {1, 5, 0,  0, ... , 0} 
		int RecycleDenomBitPosition[DENOM_MAX];	//Tells which bit corresponds to which denom					ex {1, 0, 5, 10, ... , 0}  //Bit 0 = 1's, Bit 2 = 5's, etc

		//These are related to the escrow unit
		BOOL EscrowUnit;
		int EscrowCapacity;
		int EscrowBillCount;
		int EscrowBills[ESCROW_MAX];

	} BVU_CONTROL, *P_BVU_CONTROL;
```
For simple built-in types, like `int` or `BOOL`, it is easy to define the equivalent type in C#, which is documented at [Marshaling Data with Platform Invoke](https://docs.microsoft.com/en-us/dotnet/framework/interop/marshaling-data-with-platform-invoke). The only difficulty lies in the marshaling of `char` and `int` arrays in `BVU_CONTROL`.  
## Traditional ways
As you may find by Googling *array P/Invoke*, a common practice is like follows:
+ Treat `char[]` (*character array*) in C as `string` in C#.
  For example, the `cInfo` field is then declared in C# to be 
```c#
[MarshalAs(UnmanagedType.ByValTStr, SizeConst = 100)]
public string cInfo;
```
and the two-dimensional array `cDenomination` is transformed as 
```c#
[MarshalAs(UnmanagedType.ByValArray, SizeConst = 16 * ID003BasicDriver.DENOM_MAX)]
public char[] cDenomination;
```
where ```ID003BasicDriver.DENOM_MAX``` is a constant defined as the one in C. Here it should be noted that we can only declared a 1-dimensional array in C# for P/Invoke purpose. This kind of marshaling is intuitive and straightforward, since in C there is no intrisinc string type and we usually use character array to denote a string.
+ Treat array of other types in C, like `int[]`, as a corresponding array type in C# with special marshaling attribute.
  Specifically, in `MarshalAs` attribute, we should specify the length of the array and the `UnmanagedType` enumeration as ```UnmanagedType.ByValArray```. The `int EscrowBills[ESCROW_MAX]` field in the above C struct iis mapped into 
```c#
[MarshalAs(UnmanagedType.ByValArray, SizeConst = 100)]
public int[] EscrowBills;
```
More details about string and array marshaling can be found at [Marshaling strings](https://docs.microsoft.com/en-us/dotnet/framework/interop/marshaling-strings) and [Marshaling Different Types of Arrays](https://docs.microsoft.com/en-us/dotnet/framework/interop/marshaling-different-types-of-arrays).
## Another perspective
Here we must realize that the above core data structure will be transfered forward and backward in order to retrieve information from the machine or send command to it. The corresponding functions using this data structure are declared as 
```c
// pass in a callback
extern "C"  __declspec(dllexport) 
int __stdcall BVUOpen(BVU_CONTROL *pCtl,
void (__stdcall* CallbackFunction) (BVU_CONTROL *pCtl, int iEvent)); 
```
As can be seen, at first a `BVU_CONTROL` is passed into the function and then the underlying library may change it according to the communication status with the machine. We also register a callback to retrieve information, which is also stored in the `BVU_CONTROL` structure. Therefore, along the whole lifetime of the application, there is only one `BVU_CONTROL` residing in memory for data transfer. 
However, when marshaling data, the interop marshaler can copy or pin the data being marshaled, see (Copying and Pinning)[https://docs.microsoft.com/en-us/dotnet/framework/interop/copying-and-pinning]. Only formatted blittable classes (or structures) are pinned during marshaling. 
*Formatted blittable classes have fixed layout (formatted) and common data representation in both managed and unmanaged memory. When these types require marshaling, a pointer to the object in the heap is passed to the callee directly. The callee can change the contents of the memory location being referenced by the pointer.*
Then, which kind of classes or structures are formatted  blittable? We should address two points here.
+formatted
To make a class or structure formatted, i.e., have the same memory layout with C, we just need to specify its struct layout as sequential by `[StructLayout(LayoutKind.Sequential)]`. In fact, this is the default behavior for `struct` in C#, though it is still better to show it explicitly.
+blittable
To identify which types are blittable, please check [Blittable and Non-Blittable Types](https://docs.microsoft.com/en-us/dotnet/framework/interop/blittable-and-non-blittable-types). The question here is whether an array of blittable types like `int` itself is blittable.
Though an array of __blittable types__ of a __fixed length__ in __function parameter passing__ is considered as blittable, an array member in a class/struct definition will make this class/struct __non-blittable__. Therefore, in the above prototype declaration in C#, the `BVUControl` struct is NOT blittable, which means there is always a copy of the `BVUControl` instance from managed memory to unmanaged memory when we pass it as an in argument. After that, the instance after modification in unmanaged memory will be copied back into managed memory again if we want the result using the `[Out]` attribute in marshaling. Besides, a `string` type is also non-blittable, which of course renders the `struct` it lies in to be non-blittable. This is a waste of resources by copying forward and backforward. What's more, it may cause some weired errors. According to the normal semantics, what we want is to share the `BVUControl` instance between managed and unmanaged memory.
Now comes the question: how can we declare a blittable `struct` such that is can be pinned even if there are fields of array type? The answer is:
##Use unsafe code in C Sharp
The programming guide for unsafe codes in C# can be found at [Unsafe Code and Pointers](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/unsafe-code-pointers/) and the `fixed` statement is introduced in [fixed Statement (C# Reference)](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/fixed-statement). 
To map an array field from C into C# in a blittable way, all it needs is to declare the field as a [fixed size buffer](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/unsafe-code-pointers/fixed-size-buffers). Then, the `char cInfo[100]` in C is declared in C# to be
```c#
private fixed byte cInfo[100]
```
since a `char` type in C can be mapped to be a `byte` type in C# if it is used as a *C-style string*. Now the whole structure in a `unsafe` environment is declared to be (`fixed` can only be used in `unsafe` code):
```c#
[StructLayout(LayoutKind.Sequential)]
public unsafe struct BVUControl
{
  public int idebugMode;
  public int iSerialPort;
  public int iDenominations; // Bill denominations to take	
  public int iDirection;     // Bill direction to accept		 
  public int iBarCodeLen;    
  public int iBillValue;      // bill value
  public int iMaxNoResponseInterval; 
  private fixed byte cInfo[100];
  public byte failureInfo;    //BV failure code 
  private byte cUnitType;

  public BvModels Model;
  private fixed byte cCountry[4];
  public int firmwareMajorVersion;

  private fixed byte cDenomination[16 * ID003BasicDriver.DENOM_MAX];

  private int hasRecycler;
  public int RecycleBoxCount;
  private fixed int recycledNoteCount[10];
  private fixed int recycleDenom[10];
  private fixed int recycleDenomBitPosition[16];

  public int EscrowUnit;
  public int EscrowCapacity;
  public int EscrowBillCount;
  public fixed int EscrowBills[100];
}
```
Of coure, to facilitate the client using this structure, we have added some property methods to access the `fixed` buffer and parse it into normal C# types. For example, since `cInfo` is actually meant to represent a string, the corresponding fixed buffer is translated into a string as follows.
```c#
public string CInfo
{
  get
  {
  	fixed (byte* data = cInfo)
  {
  	return new string((sbyte*)data);
  }
  }
}
```
The complete codes in this tutorial can be found in my GitHub: [PInvoke](https://github.com/ShuhuaGao/PInvoke).








