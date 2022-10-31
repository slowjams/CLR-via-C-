Chapter 6-Type and Member Basics
==============================

## The Different Kinds of Type Members

A type can define zero or more of the following kinds of members:
<ul>
  <li><b>Constants</b> A constant is a symbol that identifies a never-changing data value. These symbols are typically used to make code more readable and maintainable. Constants are always associated with a type, not an instance of a type. Logically, constants are always static members.</li>
  <li><b>Fields</b> A field represents a read-only or read/write data value. A field can be static, in which case the field is considered part of the type’s state. A field can also be instance (nonstatic), in which case it's considered part of an object’s state. I strongly encourage you to make fields private so that the state of the type or object can't be corrupted by code outside of the defining type.</li>
  <li><b>Instance constructors</b> An instance constructor is a special method used to initialize a new object’s instance fields to a good initial state.</li>
  <li><b>Type constructors</b> A type constructor is a special method used to initialize a type's static fields to a good initial state.</li>
  <li><b>Methods</b> A method is a function that performs operations that change or query the state of a type (static method) or an object (instance method). Methods typically read and write to the fields of the type or object.</li>
  <li><b>Operator overloads</b> An operator overload is a method that defines how an object should be manipulated when certain operators are applied to the object.</li>
  <li><b>Conversion operators</b> A conversion operator is a method that defines how to implicitly or explicitly cast or convert an object from one type to another type. As with operator overload methods</li>
  <li><b>Properties</b> A property is a mechanism that allows a simple, field-like syntax for setting or querying part of the logical state of a type (static property) or object (instance property) while ensuring that the state doesn't become corrupt. Properties can be parameterless (very common) or parameterful (fairly uncommon but used frequently with collection classes).</li>
  <li><b>Events</b> To work on</li>
  <li><b>Types</b> A type can define other types nested within it. This approach is typically used to break a large, complex type down into smaller building blocks to simplify the implementation.</li>
</ul> 

## How the CLR Calls (using call and callvirt IL ) Virtual Methods 

The Employee class shown below defines three different kinds of methods:
```C#
internal class Employee {
   // A nonvirtual instance method
   public Int32 GetYearsEmployed() { ... }
   
   // A virtual method
   public virtual String GetProgressReport() { ... }
   
   // A static method
   public static Employee Lookup(String name) { ... }
}
```
When the compiler compiles this code, the compiler emits three entries in the resulting assembly's method definition table. Each entry has flags set indicating if the method is instance, virtual, or static.

When code is written to call any of these methods, the compiler emitting the calling code examines the method definition’s flags to determine how to emit the proper IL code so that the call is made correctly. The CLR offers two IL instructions for calling a method:
<ul>
  <li>The <code>call</code> IL instruction can be used to call static, instance, and virtual methods. When the call instruction is used to call a static method, you must specify the type that defineds the method that the CLR should call. When the call instruction is used to call an instance or virtual method, you must specify a variable that refers to an object. <b>The call instruction assumes that this variable is not null</b>. In other words, the type of the variable itself indicates which type defines the method that the CLR should call. If the variable's type doesn’t define the method, base types are checked for a matching method. The call instruction is frequently used to call a virtual method nonvirtually (for compiler optimization when a method is marked virutal but can be called nonvirtually).</li>
  <li>The <code>callvirt</code> IL instruction can be used to call instance and virtual methods, not static methods. When the callvirt instruction is used to call an instance or virtual method, you must specify a variable that refers to an object (in contrast, call IL instruction only requries to specify the type that defines the method). When the callvirt IL instruction is used to call a virtual instance method, the CLR discovers the actual type of the object being used to make the call and then calls the method polymorphically. In order to determine the type, the variable being used to make the call must not be null.  In other words, when compiling this call, the <b>JIT compiler generates code that verifies that the variable's value is not null</b>. If it is null, the callvirt instruction causes the CLR to throw a NullReferenceException. This additional check means that the callvirt IL instruction executes slightly more slowly than the call instruction. Note that this null check is performed even when the callvirt instruction is used to call a nonvirtual instance method. (see <code>o.GetType();</code> examle below)</li>
</ul> 

So now, let’s put this together to see how C# uses these different IL instructions:
```C#
public static void Main() {
   Console.WriteLine();     // Call a static method
   
   Object o = new Object();
   o.GetHashCode();         // Call a virtual instance method
   o.GetType();             // Call a nonvirtual instance method
}
```
If you were to compile the code above and look at the resulting IL, you’d see the following:
```
...
IL_0000: call void System.Console::WriteLine() 
...
IL_000c: callvirt instance int32 System.Object::GetHashCode()
...
IL_0013: callvirt instance class System.Type System.Object::GetType()
...
```

Notice that the C# compiler uses the <code>call</code> IL instruction to call Console's WriteLine method. This is expected because WriteLine is a static method. Next, notice that the <code>callvirt</code> IL instruction is used to call GetHashCode. This is also expected, because GetHashCode is a virtual method. Finally, notice that the C# compiler also uses the <code>callvirt</code> IL instruction to call the GetType method. This is surprising because GetType is not a virtual method. However, this works because while JIT-compiling this code, the CLR will know that GetType is not a virtual method, and so the JIT-compiled code will simply call GetType nonvirtually.

Of course, the question is, why didn’t the C# compiler simply emit the call instruction instead? The answer is because the C# team decided that <b>the JIT compiler should generate code to verify that the object being used to make the call is not null</b>. This means that calls to nonvirtual instance methods are a little slower than they should be. It also means that the following C# code will cause a NullReferenceException to be thrown. In some other programming languages, the intention of the following code would run just fine:
```C#
public sealed class Program {
   public Int32 GetFive() { return 5; }
   public static void Main() {
      Program p = null;
      Int32 x = p.GetFive(); // code compile, but NullReferenceException is thrown at run-time
   }
}
```
Theoretically, the preceding code is fine. Sure, the variable `p` is null, but when calling a nonvirtual method (GetFive), the CLR needs to know just the data type of p, which is Program. If `GetFive` did get called, the value of the `this` argument would be null, but because `this` is not used inside the `GetFive` method, no NullReferenceException would be thrown. However, because the C# compiler emits a callvirt instruction instead of a call instruction, the preceding code will end up throwing the NullReferenceException.

Sometimes, the compiler will use a <code>call</code> instruction to call a virtual method instead of using a <code>callvirt</code> instruction. At first, this may seem surprising, but the code below demonstrates why it is sometimes required:
```C#
internal class SomeClass {
   public override String ToString() {
       // Compiler uses the 'call' IL instruction to call
       // Object’s ToString method nonvirtually.

       // If the compiler were to use 'callvirt' instead of 'call', this
       // method would call itself recursively until the stack overflowed
       return base.ToString();
   }
}
```

Compilers tend to use the call instruction when calling methods defined by a value type because value types are sealed, and in addition, the nature of a value type instance guarantees it can never be null, <code>call</code> instruction can be safely used, which causes the performance of the call to be faster compared to reference types. 

Finally, if you were to call a value type's virtual method virtually(the methods defined by `ValueType` abstract class), the CLR would need to have a reference to the value type's type object in order to refer to the method table within it. This requires boxing the value type. Boxing puts more pressure on the heap, forcing more frequent garbage collections and hurting performance.

<div class="alert alert-info p-1" role="alert">
    Regardless of whether <code>call</code> or <code>callvirt</code> is used to call an instance or virtual method, these methods always receive a hidden <code>this</code> argument as the method's first prarmeter. The <code>this</code> argument refers to the object being operated on.
</div>

When designing a type, you should try to minimize the number of virtual methods you define. First, calling a virtual method is slower than calling a nonvirtual method. Second, virtual mehtod cannot be inlilned by the JIT compiler, which further hurts performance. Third, virtual methods make
versioning of components more brittle, as described in the next section. Fourth, when defining a base type, it is common to offer a set of convenience overloaded methods. If you want these methods to be polymorphic, the best thing to do is to make the most complex method virtual and leave all of the convenience overloaded methods nonvirtual. By the way, following this guideline will also improve the ability to version a component without adversely affecting the derived types. Here is an example:
```C#
public class Set {
   private Int32 m_length = 0;
   // This convenience overload is not virtual
   public Int32 Find(Object value) {
      return Find(value, 0, m_length);
   }
   // This convenience overload is not virtual
   public Int32 Find(Object value, Int32 startIndex) {
      return Find(value, startIndex, m_length - startIndex);
   }
   // The most feature-rich method is virtual and can be overridden
   public virtual Int32 Find(Object value, Int32 startIndex, Int32 endIndex) {
      // Actual implementation that can be overridden goes here...
   }
   // Other methods go here
}
```
Instead of 
```C#
public class Set {
   private Int32 m_length = 0;
   // This convenience overload is not virtual
   public virtual Int32 Find(Object value) {
      // Actual implementation
   }
   // This convenience overload is not virtual
   public virtual Int32 Find(Object value, Int32 startIndex) {
     // Actual implementation
   }
}
```

Below is an interesting tip to make compiler uses <code>call</code> instead of <code>callvirt</code> for optimization:
```C#
public class Car { 
   public void Run() {}
}

static void Main(string[] args) {
    Car c = new Car();
    c.Run();            //  callvirt IL is used by csc.exe compiler since JIT compiler needs to make sure the variable is not null
    
    new Car().Run;      //  call IL is used by csc.exe compiler, since it won't be a null variable in this scenario
}
```

## Using Type Visibility and Member Accessibility Intelligently

With the .NET Framework, applications are composed of types defined in multiple assemblies produced by various companies. This means that the developer has little control over the components he or she is using and the types defined within those components. The developer typically doesn't have access to the source code (and probably doesn’t even know what programming language was used to create the component), and components tend to version with different schedules. Furthermore, due to polymorphism and protected members, a base class developer must trust the code written by the derived class developer. And, of course, the developer of a derived class must trust the code that he is inheriting from a base class. These are just some of the issues that you need to really think about
when designing components and types.

In this section, I'd like to say just a few words about how to design a type with these issues in mind. Specifically, I'm going to focus on the proper way to set type visibility and member accessibility so that you'll be most successful.

First, when defining a new type, compilers should make the class sealed by default so that the class cannot be used as a base class. Instead, many compilers, including C#, default to unsealed classes and allow the programmer to explicitly mark a class as sealed by using the sealed keyword. Obviously, it
is too late now, but I think that today’s compilers have chosen the wrong default and it would be nice if this could change with future compilers. There are three reasons why a sealed class is better than an unsealed class:

<ul>
  <li><b>Versioning</b> When a class is originally sealed, it can change to unsealed in the future without breaking compatibility. However, after a class is unsealed, you can never change it to sealed in the future as this would break all derived classes.</li>
  <li><b>Performance</b> As discussed in the previous section, calling a virtual method doesn't perform as well as calling a nonvirtual method because the CLR must look up the type of the object at run time in order to determine which type defines the method to call. However, if the JIT compiler sees a call to a virtual method using a sealed type, the JIT compiler can produce more efficient code by calling the method nonvirtually. It can do this because it knows there can't possibly be a derived class if the class is sealed. For example, in the code below, the JIT compiler can call the virtual ToString method nonvirtually</li>
</ul> 

```C#
public sealed class Point {
   private Int32 m_x, m_y;

   public Point(Int32 x, Int32 y) { m_x = x; m_y = y; }

   public override String ToString() {
      return String.Format("({0}, {1})", m_x, m_y);
   } 

   public static void Main() {
      Point p = new Point(3, 4);
      // The C# compiler emits the callvirt instruction here but the
      // JIT compiler will optimize this call and produce code that
      // calls ToString nonvirtually because p's type is Point,
      // which is a sealed class
      Console.WriteLine(p.ToString());
   }
}
```
<ul>
  <li><b>Security and predictability</b> A class must protect its own state and not allow itself to ever become corrupted. When a class is unsealed, a derived class can access and manipulate the base class's state if any data fields or methods that internally manipulate fields are accessible and not private. In addition, a virtual method can be overridden by a derived class, and the derived class can decide whether to call the base class’s implementation. By making a method, property, or event virtual, the base class is giving up some control over its behavior and its state. Unless carefully thought out, this can cause the object to behave unpredictably, and it opens up potential security holes</li>
</ul> 

Here are the guidelines I follow when I define my own classes:

<ul>
  <li>When defining a class, I always explicitly make it sealed unless I truly intend for the class to be a base class that allows specialization by derived classes. I also default to making the class internal unless I want the class to be publicly exposed outside of my assembly. Fortunately, if you do not explicitly indicate a type's visibility, the C# compiler defaults to internal. If I really feel that it is important to define a class that others can derive but I do not want to allow specialization, I will simulate creating a closed class by using the above technique of sealing the virtual methods that my class inherits.</li>
  <li>Inside the class, I always define my data fields as private and I never waver on this. Fortunately, C# does default to making fields private. I'd actually prefer it if C# mandated that all fields be private and that you could not make fields protected, internal, public, and so on. </li>
  <li>Inside the class, I always define my methods, properties, and events as private and nonvirtual. Fortunately, C# defaults to this as well. I try to avoid making any of these members protected or internal, because this would be exposing my type to some potential vulnerability. Certainly, I'll make a method, property, or event public to expose some functionality from the type. However, I would sooner make a member protected or internal than I would make a member virtual because a virtual member gives up a lot of control and really relies on the proper behavior of the derived class.</li>
</ul> 

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