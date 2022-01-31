Chapter 7-Constants and Fields
==============================

## Constants

A *constant* is a symbol that has a never-change value. When defining a constant symbol, its value must be determinable at compile time. The compiler then saves the constant's value in the assembly's metadata. This means that you can define a constant only for types that your compiler considers primitive types. In C#, the following types are primitives and can be used to define constants: Boolean, Char, Byte, Int32...Decimal and String. However, C# also allows you to define a constant variable of a non-primitive type if you set the value to null:
```C#
public sealed class SomeType {
   public const SomeType Empty = null;
}
```
Constants are always considered to be static members, not instance members, that's why you can access a constant as accessing a static field like:
```C#
public class Car {
   public const Int32 MaxSpeed = 100;
}

static void Main(string[] args) {
   Console.WriteLine(Car.MaxSpeed);
}
```
Defining a constant causes the creation of metadata.

When code refers to a constant symbol, compilers look up the symbol in the metadata of the assembly that defines the constant, extract the constant's value, and embed the value in the emitted Intermediate Language (IL) code. Because a constant's value is embedded directly in code, constants don't reuqire any memory to be allocated for them at rum time. In addition, you can't get the address of a constant and you can't pass a constant by reference. These constraints also mean that constants don't have a good cross-assembly versioning story, so you should use them only when you know that the value of a symbol will never change. (Defining MaxInt16 as 32767 is a good example.) Let medemonstrate exactly what I mean. First, take the following code and compile it into a DLL assembly:
```C#
public sealed class SomeLibraryType {
   public const Int32 MaxEntriesInList = 50;  // you cannot add static modifier as constants are always implicitly static. 
}
```
Then use the following code to build an application assembly:
```C#
public static void Main() {
   Console.WriteLine("Max entries supported in list: " + SomeLibraryType.MaxEntriesInList);
}
```
When the compiler builds the application code, it sees that MaxEntriesInList is a constant literal with a value of 50 and embeds the Int32 value of 50 right inside the application's IL code, as you can see in the following IL code. In fact, after building the application assembly, the DLL assembly isn't even loaded at run time and can be deleted from the disk **because the compiler doesn't even add a reference to the DLl assembly in the application's metadata.**
```
.method public hidebysig static void Main() cil managed {
   ...
   IL_0000: nop
   IL_0001: ldstr "Max entries supported in list: "
   IL_0006: ldc.i4.s 50
   IL_0008: box [mscorlib]System.Int32
   ...
}
```
This example should make the versioning problem obvious to you. If the developer changes the MaxEntriesInList constant to 1000 and only rebuilds the DLL assembly, the application assembly is not affected. For the application to pick up the new value, it will have to be recompiled as well. You can't use constants if you need to have a value in one assembly picked up by another assembly at run time (instead of compile time). Instead, you can use readonly fields, which I’ll discuss next.

<div class="alert alert-info p-1" role="alert">
    Note that if the dll assembly define a string literal, when you use ILDasm you will also see the string seems to be embedded in the calling assembly's IL code but it is just for readibility feature added by the IL viewer, unless the string is defined as a constant, it won't be embedded in the calling assembly's IL code.
</div>

## Fields

A *field* is a data member that holds an instance of a value type of a reference to a reference type.

Yhe common language run time (CLR) supports both type (static) and instance (nonstatic) fields. For type fields, the dynamic memory required to hold the field's data is allocated inside the type object, which is created when the type is loaded into an AppDomain (see Chapter 22, "CLR Hosting and AppDomains"), which typically happens the first time any method that references the type is just-in-time (JIT)–compiled. For instance fields, the dynamic memory to hold the field is allocated when an instance of the type is constructed.

Because fields are stored in dynamic memory, their value can be obtained at run time only. Fields also solve the versioning problem that exists with constants. In addition, a field can be of any data type, so you don’t have to restrict yourself to your compiler’s built-in primitive types (as you do for
constants).

The CLR supports readonly fields and read/write fields. Most fields are read/write fields, meaning the field's value might change multiple times as the code executes. However, readonly fields can be written to only within a constructor method (which is called only once, when an object is first created). Compilers and verification ensure that readonly fields are not written to by any method other than a constructor. Note that reflection can be used to modify a readonly field.

Let's take the example from the "Constants" section and fix the versioning problem by using a static readonly field. Here's the new version of the DLL assembly's code:
```C#
public sealed class SomeLibraryType {
   public static readonly Int32 MaxEntriesInList = 50;  // The static is required to associate the field with the type. 
}
```
Let's say that the developer of the DLL assembly changes the 50 to 1000 and rebuilds the assembly. When the application code is re-executed, it will automatically pick up the new value: 1000. In this case, the application code doesn't have to be rebuilt—it just works.

The following example shows how to define a readonly static field that is associated with the type itself, as well as read/write static fields and readonly and read/write instance fields, as shown here:
```C#
public sealed class SomeType {
   // This is a static read-only field; its value is calculated and
   // stored in memory when this class is initialized at run time. 
   public static readonly Random s_random = new Random();

   // This is an instance read-only field.
   public readonly String Pathname = "Untitled";  // Note that C# treats initializing a field inline as shorthand syntax for initializing the field
                                                  //in a constructor, check Chapter 8, "Methods" for details

   public SomeType(String pathname) {
      // This line changes a read-only field.
      // This is OK because the code is in a constructor.
      this.Pathname = pathname;
   }
}
```
<div class="alert alert-info p-1" role="alert">
    One biggest difference between constants and readonly fields is, constants are static, and can only be primitive type (most commonly, Int32, String), while readonly is instance member and can be any type.
</div>

<style type="text/css">
.markdown-body {
  max-width: 1800px;
  margin-left: auto;
  margin-right: auto;
}
</style>

<link rel="stylesheet" href="./zCSS/bootstrap.min.css">
<script src="./zCSS/jquery-3.3.1.slim.min.js"></script>
<script src="./zCSS/popper.min.js"></script>
<script src="./zCSS/bootstrap.min.js"></script>