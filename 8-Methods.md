Chapter 8-Methods
==============================

## Instance Constructors and Classes (Reference Types)
Constructors are special **methods** that allow an instance of a type to be initialized to a good state. Constructor methods are always called .ctor (for constructor) in a method definition metadata table. When creating an instance of a reference type, memory is allocated for the instance’s data fields, the
object’s overhead fields (type object pointer and sync block index) are initialized, and then the type’s instance constructor is called to set the initial state of the object.

When constructing a reference type object, the memory allocated for the object is always zeroed out before the type’s instance constructor is called. Any fields that the constructor doesn’t explicitly overwrite are guaranteed to have a value of 0 or null.

<div class="alert alert-info p-1" role="alert">
    CLR always zero out the new pages. Compared to C, we have <code>malloc()</code> and <code>calloc()</code>, the former won't zero out pages requested from OS while the latter will. But OS sometimes still zeroed out pages even we use <code>malloc()</code> because OS might free memory that was previously used by a different process, in this case (normally we ask for a very large chunka of space) OS has to zero out pages for security reasons. If we just ask for small memory allocation, OS just free existing memory used by current process, so in this case, pages won't be zeroed out
</div>

Reference: https://stackoverflow.com/questions/8029584/why-does-malloc-initialize-the-values-to-0-in-gcc

If you define a class that does not explicitly define any constructors, the C# compiler defines a default (parameterless) constructor for you whose implementation simply calls the base class’s parameterless constructor.

```C#
public class SomeType { }
```
it is as though you wrote the code as follows:
```C#
public class SomeType {
   public SomeType() : base() { }
}
```
If the class is abstract, the compiler-produced default constructor has **protected** accessibility; otherwise, the constructor is given **public** accessibility.

If the base class doesn't offer a parameterless constructor (when you define a parameterized constructor, the default constructor goes away), the derived class must explicitly call a base class constructor or the compiler will issue an error

If the class is static (sealed and abstract), the compiler will not emit a default constructor at all into the class definition.

The C# compiler will generate a call to the default base class’s constructor automatically if the derived class’s constructor does not explicitly invoke one of the base class’s constructors.

```C#
public class Derived : SomeType
{
    public Derived(int num) /*: base() */{ }   //every constructor users define call base's constructor by default implicitly
}
```

Ultimately, `System.Object`'s public, parameterless constructor gets called. This constructor does nothing—it simply returns. This is because `System.Object` defines no instance data fields, and therefore its constructor has nothing to do

C# offers a simple syntax that allows the initialization of fields defined within a reference type when an instance of the type is constructed:
```C#
internal sealed class SomeType {
   private Int32 m_x = 5;
}
```
Intermediate Language (IL) for SomeType’s constructor method shown as:
```
.method public hidebysig specialname rtspecialname
        instance void .ctor() cil managed
{
   // Code size 14 (0xe)
   .maxstack 8
   IL_0000: ldarg.0
   IL_0001: ldc.i4.5
   IL_0002: stfld int32 SomeType::m_x
   IL_0007: ldarg.0
   IL_0008: call instance void [mscorlib]System.Object::.ctor()
   IL_000d: ret
}
```
SomeType's constructor contains code to store a 5 into m_x and then calls the base class's constructor. In other words, the C# compiler allows the convenient syntax that lets you initialize the instance fields inline and **translates this to code in the constructor method** to perform the initialization. This means that you should be aware of code explosion, as illustrated by the following class definition:

```C#
internal sealed class SomeType : OtherType
{
    private Int32 m_x = 5;
    private String m_s = "Hi there";
    private Double m_d = 3.14159;
    private Byte m_b;

    // Here are some constructors.
    public SomeType() { ... }
    public SomeType(Int32 x) { ... }
    public SomeType(String s) {  
        // ...  execute all the field initializations
        // ...  call base constructor
        m_d = 10;                  // executed after base construct is called
        Console.WriteLine(m_d);    // executed after base construct is called
    }
}
```

`m_d` will eventually be 10 as the generated IL code sequence is like:
<ol>
  <li><b>Step 1-initialization statements executes</b></li>
  <li><b>Step 2-base constructor called</b></li>
  <li><b>Step 3-any code statment defined inside the constructor execute</b></li>
</ol> 

<div class="alert alert-info p-1" role="alert">
    The compiler initializes any fields by using the convenient syntax before calling a base class's constructor to maintain the impression that these fields always have a value as the source code appearance dictates. The potential problem occurs if you define a base class'ss constructor invokes a virtual method (might change fields) that calls back into a method defined by the derived class because the fields initialized by using the convenient syntax have been initialized before the virtual method is called (probably OK) while the fields initialized inside the constructor will not been fully initialized, which result in unpredictable behavior. Anyway, you should not call any virtual methods within a constructor that can affect the object being constructed. 
</div>


If you define multiple constructors as above, all initialization IL code will be duplicated in every constructor. You should avoid it by creating a single constructor that performs the common initialization, which reduces generated IL code as:

```C#
internal sealed class SomeType
{
    // Do not explicitly initialize the fields here.
    private Int32 m_x;
    private String m_s;
    private Double m_d;
    private Byte m_b;

    // This constructor sets all fields to their default. All of the other constructors explicitly invoke this constructor.
    public SomeType() {
        m_x = 5;
        m_s = "Hi there";
        m_d = 3.14159;
        m_b = 0xff;
    }

    public SomeType(Int32 x) : this() { m_x = x; }

    public SomeType(String s) : this() { m_s = s }
    ...
}
```

## Instance Constructors and Classes (Value Types)

Like a class, you can create an instance of a struct using the new keyword. You can either invoke the default parameterless constructor, which initializes all fields in the struct to their default values, or you can invoke a custom constructor.

```C#
struct MyStruct {
   int num;
   public MyStruct(int i) { num = i };
}

// Method 1 - Parameterless constructor, data in struct initialized to default values
MyStruct m1 = new MyStruct();
 
// Method 2 - Call custom constructor
MyStruct m2 = new MyStruct(1);
```

You can also create an instance of a struct by just declaring it, without using the new operator.  The object is created, although you can't use it until you’ve explicitly initialized all of its data members.

```C#
// Method 3 - Just declare it
MyStruct m3;
 
// Visual Studio compile error - Can't use m3 yet (use of unassigned field)
Console.WriteLine(m3.num);
```
We need to first initialize all data members:
```C#
MyStruct m3;
m3.num = 10;
Console.WriteLine(m3.num);
```

<div class="alert alert-info p-1" role="alert">
     CLR initialises the stack area to zero. Note that .Net 5 introduces a way to turn this off. </br>
     At runtime <code>MyStruct m</code>; and <code>MyStruct m = new MyStruct();</code> will both lead to m being zeroed-out. The difference is that C# enforces an additional compile-time requirement
</div>

C# purposely disallows value types from defining parameterless constructors to remove any confusion a developer might have about when that constructor gets called. If the constructor can't be defined, the compiler can never generate code to call it automatically. For example, below code doesn't compile:

```C#
internal struct Point {
   public Int32 m_x, m_y;
   public Point() {        // compile error!  users cannot define parameterless constructors
      m_x = m_y = 5;
   }
}
```
Therefore you cannot use the convenient field initialization syntax neither as:

```C#
internal struct SomeValType {
   private Int32 m_x = 5;   //compiler error, cannot do inline instance field initialization in a value type.
}
```
<div class="alert alert-info p-1" role="alert">
    C# doesn't allow users to define default constructors, probably because if we use an array of struct objects such as <code>MyStruct[100] m</code>, CLR needs to quickly create the zeroed out memory pages, otherwise calling constructors for 100 times is inefficient. For reference type, it is OK since reference type can have default null value.
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