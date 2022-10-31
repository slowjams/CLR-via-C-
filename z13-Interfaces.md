Chapter 13-Interfaces
==============================

## Defining an Interface

An interface is a named set of method signatures. Note that interfaces can also define events, parameterless properties, and parameterful properties (indexers in C#) because all of these are just syntax shorthands that map to methods anyway, as shown in previous chapters. However, an interface cannot define any constructor methods. In addition, an interface is not allowed to define any instance fields. C# prevents an interface from defining any of these static members.

An interface definition can "inherit" other interfaces. However, I use the word inherit here rather loosely because interface inheritance doesn't work exactly like class inheritance. I prefer to think of interface inheritance as including the contract of other interfaces. For example, the `ICollection<T>` interface definition includes the contracts of the `IEnumerable<T>` and `IEnumerable` interfaces. This means that any class that inherits the `ICollection<T>` interface must implement all of the methods defined by the `ICollection<T>`, `IEnumerable<T>`, and `IEnumerable` interfaces.

```C#
public interface IEnumerable<out T> : IEnumerable {
   new IEnumerator<T> GetEnumerator();
}

public interface IEnumerable {
   IEnumerator GetEnumerator();
}

public interface IEnumerator<out T> : IDisposable, IEnumerator {
   new T Current { get; }
}

public interface IEnumerator {
   object Current { get; }
   bool MoveNext();
   void Reset();
}

public interface ICollection<T> : IEnumerable<T> {
   int Count { get; }
   bool IsReadOnly { get; }
   void Add(T item);
   void Clear();
   bool Contains(T item);
   void CopyTo(T[] array, int arrayIndex);
   bool Remove(T item);
}

public interface IList<T> : ICollection<T>    // related to index
   T this[int index] { get; set; }
   int IndexOf(T item);
   void Insert(int index, T item);
   void RemoveAt(int index);
}
```

## Implement IEnumerable and IEnumerator on your own collection
The code below uses EIMI, see the next section for details and come back to review the code

```C#
static void Main(string[] args) {
    var list = new MyList() { "Hello", "World" };

    var enumerator = list.GetEnumerator();
    while (enumerator.MoveNext()) {
        Console.WriteLine(enumerator.Current);
    }

    enumerator.Reset();             //reset the index so that the list can be iterated again

    while (enumerator.MoveNext()) {
        Console.WriteLine(enumerator.Current);
    }

    foreach (var item in list) {    // foreach block is simply a syntax sugar that uses above GetEnumerator, MoveNext pattern
        Console.WriteLine(item);
    }

    //use generic list
    var list = new MyList<string>() { "Hello", "World" };
    var enumerator = list.GetEnumerator();
    while (enumerator.MoveNext()) {
        Console.WriteLine(enumerator.Current);
    }
}

public class MyList : IEnumerable {
    private object[] _objects;
    private int index;

    public MyList() {
        _objects = new object[6];
    }

    public void Add(object obj) {
        _objects[index++] = obj;
    }

    public IEnumerator GetEnumerator() {
        return new ListEnumerator(this);

        //return _objects.GetEnumerator();  can use array's own enumerator but want to implement the details by yourself
    }

    IEnumerator IEnumerable.GetEnumerator() {
        return new ListEnumerator(this);
    }

    private class ListEnumerator : IEnumerator {
        private int _currentIndex = -1;
        private MyList list;

        public ListEnumerator(MyList list) {
            this.list = list;
        }

        public bool MoveNext() {
            _currentIndex++;
            return (_currentIndex < list._objects.Length);
        }

        object IEnumerator.Current {
            get
            {
                try {
                    return list._objects[_currentIndex];
                }
                catch (IndexOutOfRangeException) {
                    throw new InvalidOperationException();
                }
            }
        }

        public void Reset() {
            _currentIndex = -1;
        }
    }
}

//Generic Version
public class MyList<T> : IEnumerable<T> {
    private T[] _objects;
    private int index;

    public MyList() {
        _objects = new T[3];
    }

    public void Add(T obj) {
        _objects[index++] = obj;
    }

    public IEnumerator<T> GetEnumerator() {
        return new ListEnumerator<T>(this);
    }

    IEnumerator IEnumerable.GetEnumerator() {
        return GetEnumerator();
    }

    private class ListEnumerator<T> : IEnumerator<T> {
        private int _currentIndex = -1;
        private MyList<T> list;

        public ListEnumerator(MyList<T> list) {
            this.list = list;
        }

        public bool MoveNext() {
            _currentIndex++;
            return (_currentIndex < list._objects.Length);
        }

        object IEnumerator.Current {
            get
            {
                return Current;
            }
        }

        public T Current {
            get
            {
                try {
                    return list._objects[_currentIndex];
                }
                catch (IndexOutOfRangeException) {
                    throw new InvalidOperationException();
                }
            }
        }

        public void Reset() {
            _currentIndex = -1;
        }

        public void Dispose() {
            // implement after learning Chapter 21
        }
    }
}
```

## Inheriting an Interface

**The C# compiler requires that a method that implements an interface be marked as public**. The CLR requires that method that implements an interface be marked as virtual. If you do not explicitly mark the method as virtual in your source code, the compiler marks the method as virtual and sealed; this prevents a derived class from overriding the interface method. If you explicitly mark the method as virtual, the compiler marks the method as virtual (and leaves it unsealed); this allows a derived class to override the interface method.

```C#
interface ITest {
    void Test();      .method public hidebysig newslot abstract virtual instance void  Test() cil managed
}

class Base : ITest {
   // This method is implicitly sealed and cannot be overridden
   // the section on the right shows IL code
   // In IL, the final(sealed) attribute does not conflict with virtual

    public void Test() { }   .method public hidebysig newslot virtual final instance void Test()
}

class WrongDerived : Base {
    public void Test() { }   // this code doesn't compile since Base.Test is sealed, 
                             // actually it does compile but VS shows it needs to be fixed by adding new keyword
}

class CorrectDerived : Base {
   new public void Test() { }
}

class A : ITest {
    public virtual void Test() { }
}

class B : A {
    public override void Test() { }     .method public hidebysig virtual instance void Test()
                                        // override means virtual too, so override = override + virtual
}

class C : B {
    public sealed override void Test() { }
}

class D : C {
    public override void Test() { }   // compile error since C.Test is sealed
}
```

Below is a concrete example:

```C#
static void Main(string[] args) {
    /************************* First Example *************************/
    Base b = new Base();

    // Calls Dispose by using b's type: "This is Base's Dispose"
    b.Dispose();

    // Calls Dispose by using b's object's type: "This is Base's Dispose"
    ((IDisposable)b).Dispose();

    /************************* Second Example ************************/
    Derived d = new Derived();

    // Calls Dispose by using d's type: "This is Derived's Dispose"
    d.Dispose();

    // Calls Dispose by using d's object's type: "This is Derived's Dispose"
    ((IDisposable)d).Dispose();

    /************** Third Example (The most important ones)************/
    b = new Derived();

    // Calls Dispose by using b's type: "This is Base's Dispose"
    // Be careful the output is NOT "This is Derived's Dispose" (even though Derived does inherit Base)        
    // because Base.Dispose method is sealed and Derived.Dispose doesn't implement Base's counterpart, what it implments is IDisposable's counterpart
    b.Dispose();

    // Calls Dispose by casting to IDisposable first: "This is Derived's Dispose"
    ((IDisposable)b).Dispose();
}

internal class Base : IDisposable {
    // This method is implicitly sealed and cannot be overriden
    public void Dispose() {
        Console.WriteLine("This is Base's Dispose");
    }
}

internal class Derived : Base, IDisposable {
    // This method cannot override Base's Dispose. 'New' is used to indicate that 
    // this method re-implements IDisposable's Dispose Method
    new public void Dispose() {
        Console.WriteLine("This is Derived's Dispose");
    }
}
```

<div class="alert alert-info p-1" role="alert">
    Like a reference type, a value type can implement zero or more interfaces. However, when you cast an instance of a value type to an interface type, the value type instance must be boxed. This is because an interface variable is a reference that must point to an object on the heap so that the CLR can examine the object's type object pointer to determine the exact type of the object. Then, when calling an interface method with a boxed value type, the CLR will follow the object’s type object pointer to find the type object’s method table in order to call the proper method.
</div>

## Explicit Interface Method Implementations (EIMI)

When a type is loaded into the CLRR, a method table is created and initialized for the type (as discussed in Chapter 1, "The CLR's Execution Model"). This method table contains one entry for every new method introduced by the type as well as entries for any virtual methods inherited by the type. Inherited virtual methods include methods defined by the base types in the inheritance hierarchy as well as any methods defined by the interface types. So if you have a simple type defined like this:
```C#
internal sealed class SimpleType : IDisposable {
   public void Dispose() { Console.WriteLine("Dispose"); }
}
```

the type's method table contains entries for the following:

<ul>
  <li>All the virtual instance methods defined by Object, the implicitly inherited base class.</li>
  <li>All the interface methods defined by IDisposable, the inherited interface. In this example, there is only one method, Dispose</li>
  <li>The new method, Dispose, introduced by SimpleType</li>
</ul> 

To make things simple for the programmer, the C# compiler assumes that the `Dispose` method introduced by SimpleType is the implementation for `IDisposable`'s `Dispose` method. The C# compiler makes this assumption because the method is public, and the signatures of the interface method and the newly introduced method are identical. That is, the methods have the same parameter and return types. By the way, if the new Dispose method were marked as virtual, the C#
compiler would still consider this method to be a match for the interface method.

When the C# compiler matches a new method to an interface method, it emits metadata indicating that both entries in SimpleType's method table should refer to the same implementation. To help make this clearer, here is some code that demonstrates how to call the class’s public Dispose method as well as how to call the class’s implementation of IDisposable’s Dispose method:

```C#
public static void Main() {
    SimpleType st = new SimpleType();

    // This calls the public Dispose method implementation
    st.Dispose();
    
    // This calls IDisposable's Dispose method implementation
    IDisposable d = st; 
    d.Dispose();   
}
```

C# requires the public Dispose method to also be the implementation for IDisposable's Dispose method, the same code will execute, and, in this example, you can't see any observable difference. The output is as follows "Dispose /n Dispose"

Now, let me rewrite the preceding SimpleType so that you can see an observable difference
```C#
internal sealed class SimpleType : IDisposable {
   public void Dispose() { Console.WriteLine("public Dispose"); }
   void IDisposable.Dispose() { Console.WriteLine("IDisposable Dispose"); }
}
```

Now the output will be the following: "public Dispose /n IDisposable Dispose".

In C#, when you prefix the name of a method with the name of the interface that defines the method (IDisposable.Dispose as in this example), you are creating an **explicit interface method implementation (EIMI)**. Note that when you define an explicit interface method in C#, you are not allowed to specify any accessibility (such as public or private). However, when the compiler generates the metadata for the method, **its accessibility is set to private**, preventing any code using an instance of the class from simply calling the interface method. The only way to call the interface method is through a variable of the interface's type.

Also note that an EIMI method cannot be marked as virtual and therefore cannot be overridden. This is because the EIMI method is not really part of the type's object model; it's a way of attaching an interface (set of behaviors or methods) onto a type without making the behaviors/methods obvious. 

Below is one more example that uses EIMI:
```C#
public interface IWindow {
   Object GetMenu();
}

public interface IRestaurant {
   Object GetMenu();
}

public sealed class MarioPizzeria : IWindow, IRestaurant {
   // This is the implementation for IWindow's GetMenu method.
   Object IWindow.GetMenu() { ... } 

   // This is the implementation for IRestaurant's GetMenu method.
   Object IRestaurant.GetMenu() { ... }

   // This (optional method) is a GetMenu method that has nothing
   // to do with an interface.
   public Object GetMenu() { ... }
}
```

## Generic Interfaces

**First, generic interfaces offer great compile-time type safety.** The following code demonstrates:
```C#
private void SomeMethod1() {
   Int32 x = 1, y = 2;
   IComparable c = x; 

   // CompareTo expects an Object; passing y (an Int32) is OK 
   c.CompareTo(y); // y is boxed here

   // CompareTo expects an Object; passing "2" (a String) compiles
   // but an ArgumentException is thrown at runtime
   c.CompareTo("2"); 
}
```

Obviously, it is preferable to have the interface method strongly typed, and this is why the FCL includes a generic `IComparable<in T>` interface. Below is the new version of the code revised by using the generic interface:
```C#
private void SomeMethod1() {
   Int32 x = 1, y = 2;
   IComparablee<Int32> c = x; 

   // CompareTo expects an Object; passing y (an Int32) is OK 
   c.CompareTo(y); // y is not boxed here

   // CompareTo expects an Object; passing "2" (a String) compiles
   // but an ArgumentException is thrown at runtime
   c.CompareTo("2");      // compile Error

}
```

**The second benefit of generic interfaces is that much less boxing will occur when working with value types.** Notice in SomeMethod1 that the non-generic `IComparable` interface’s CompareTo method expects an Object; passing y (an Int32 value type) causes the value in y to be boxed. However, in SomeMethod2, the generic `IComparable<in T>` interface’s CompareTo method expects an Int32; passing y causes it to be passed by value, and no boxing is necessary.

<div class="alert alert-info p-1" role="alert">
    The FCL defines non-generic and generic versions of the IComparable, ICollection, IList, and IDictionary interfaces, as well as some others. If you are defining a type, and you want to implement any of these interfaces, you should typically implement the generic versions of these interfaces. The non-generic versions are in the FCL for backward compatibility to work with code written before the .NET Framework supported generics. The nongeneric versions also provide users a way of manipulating the data in a more general, less type-safe fashion.
    <br><br>
    Some of the generic interfaces inherit the non-generic versions, so your class will have to implement both the generic and non-generic versions of the interfaces. For example, the generic <code>IEnumerable&lt;out T&gt;</code> interface inherits the non-generic <code>IEnumerable</code> interface. So if your class implements <code>IEnumerable&lt;out T&gt;</code>, your class must also implement <code>IEnumerable</code>.
    <br><br>
    Sometimes when integrating with other code, you may have to implement a non-generic interface because a generic version of the interface simply doesn’t exist. In this case, if any of the interface’s methods take or return Object, you will lose compile-time type safety, and you will get boxing with value types. You can alleviate this situation to some extent by using a technique I describe in the “Improving Compile-Time Type Safety with Explicit Interface Method Implementations” section near the end of this chapter.
</div>

**The third benefit of generic interfaces is that a class can implement the same interface multiple times as long as different type parameters are used.**

The following code shows an example of how useful this could be:

```C#
// This class implements the generic IComparable<T> interface twice
public sealed class Number: IComparable<Int32>, IComparable<String> {
   private Int32 m_val = 5; 

   // This method implements IComparable<Int32>'s CompareTo
   public Int32 CompareTo(Int32 n) {
       return m_val.CompareTo(n);
   }

   // This method implements IComparable<String>'s CompareTo
   public Int32 CompareTo(String s) {
       return m_val.CompareTo(Int32.Parse(s));
   }
}

public static void Main() {
   Number n = new Number();

   // Here, I compare the value in n with an Int32 (5)
   IComparable<Int32> cInt32 = n;
   Int32 result = cInt32.CompareTo(5);

   // Here, I compare the value in n with a String ("5")
   IComparable<String> cString = n;
   result = cString.CompareTo("5");
}
```

An interface’s generic type parameters can also be marked as contra-variant and covariant, which allows even more flexibility for using generic interfaces.

## Generics and Interface Constraints

**The first benefit is that you can constrain a single generic type parameter to multiple interfaces.** When you do this, the type of parameter you are passing in must implement all of the interface constraints. Here is an example:
```C#
public static class SomeType {
   private static void Test() {
      Int32 x = 5;
      Guid g = new Guid();

      // This call to M compiles fine because
      // Int32 implements IComparable AND IConvertible
      M(x);

      // This call to M causes a compiler error because
      // Guid implements IComparable but it does not implement IConvertible
      M(g);
   }
   // M's type parameter, T, is constrained to work only with types that
   // implement both the IComparable AND IConvertible interfaces 
   private static Int32 M<T>(T t) where T: IComparable, IConvertible { ... }
}
```

**The second benefit of interface constraints is reduced boxing when passing instances of value types.**  In the above code fragment, the M method was passed x (an instance of an Int32, which is a value type). No boxing will occur when x is passed to M.  If code inside M does call t.CompareTo(...), still no boxing occurs to make the call (boxing may still happen for arguments passed to CompareTo).

For interface constraints, the C# compiler emits certain Intermediate Language (IL) instructions that result in calling the interface method on the value type directly without boxing it. Aside from using interface constraints, there is no other way to get the C# compiler to emit these IL instructions, and therefore, calling an interface method on a value type always causes boxing.

On the other hand, if M had been declared like this:
```C#
private static Int32 M(IComparable t) {
   ...
}
```

then in order to pass x to M, x would have to be boxed.

## Improving Compile-Time Type Safety with Explicit Interface Method Implementations

Unfortunately, there may be times when you need to implement a non-generic interface because a generic version doesn’t exist. If any of the interface’s method(s) accept parameters of type System.Object or return a value whose type is System.Object, you will lose compile-time type safety, and you will get boxing. In this section, I’ll show you how you can use EIMI to improve this situation somewhat.

Look at the very common IComparable interface:
```C#
public interface IComparable {
   Int32 CompareTo(Object other);
}
```

If I define my own type that implements this interface, the type definition might look like the following:
```C#
internal struct SomeValueType : IComparable {
   private Int32 m_x;
   public SomeValueType(Int32 x) { m_x = x; }
   public Int32 CompareTo(Object other) {
      return(m_x - ((SomeValueType) other).m_x);
   }
}

public static void Main() {
   SomeValueType v = new SomeValueType(0); 
   Object o = new Object();
   Int32 n = v.CompareTo(v);  // Undesired boxing
   n = v.CompareTo(o);        // InvalidCastException
}
```

There are two characteristics of this code that are not ideal:

<ul>
  <li><b>Undesired boxing</b> When v is passed as an argument to the CompareTo method, it must be boxed because CompareTo expects an Object.</li>
  <li><b>The lack of type safety</b> This code compiles, but an InvalidCastException is thrown inside the CompareTo method when it attempts to cast o to SomeValueType.</li>
</ul> 

Both of these issues can be fixed by using EIMIs. below is a modified version of SomeValueType that has an EIMI added to it:
```C#
internal struct SomeValueType : IComparable {
   private Int32 m_x;
   public SomeValueType(Int32 x) { m_x = x};
   public Int32 CompareTo(SomeValueType other) {
      return m_x - other.m_x;
   }

   Int32 IComparable.CompareTo(Object other) {
      return CompareTo((SomeValueType) other);
   }
}
```

Having made these two changes means that we now get compile-time type safety and no boxing:
```C#
public static void Main() {
   SomeValueType v = new SomeValueType(0);
   Object o = new Object();
   Int32 n = v.CompareTo(v);  // No boxing
   n = v.CompareTo(o);        // compile-time error 
}
```

If, however, we define a variable of the interface type, we will lose compile-time type safety and experience undesired boxing again:
```C#
public static void Main() {
   SomeValueType v = new SomeValueType(0);
   IComparable c = v;

   Object o = new Object();
   Int32 n = c.CompareTo(v);  //Undesired boxing
   n = c.CompareTo(o);        // InvalidCastException
}
```

In fact, as mentioned earlier in this chapter, when casting a value type instance to an interface type, the CLR must box the value type instance. Because of this fact, two boxings will occur in the previous Main method.

<div class="alert alert-info p-1" role="alert">
    EIMI used in this scenerio just creates an illusion to developers that this type implements a generic interface so that they can use the generic interface method, but as you can see, when you use the interface variable, you cannot using genetic interface method anymore
</div>

EIMIs are frequently used when implementing interfaces such as IConvertible, ICollection, IList, and IDictionary. They let you create type-safe versions of these interfaces' methods, and they enable you to reduce boxing operations for value types.

## Be Careful with Explicit Interface Method Implementations

It is critically important for you to understand some ramifications that exist when using EIMIs. And because of these ramifications, you should try to avoid EIMIs as much as possible. Fortunately, generic interfaces help you avoid EIMIs quite a bit. But there may still be times when you will need to use them (such as implementing two interface methods with the same name and signature). Here are the big problems with EIMIs:

<ul>
  <li>There is no documentation explaining how a type specifically implements an EIMI method, and there is no Microsoft Visual Studio IntelliSense support.</li>
  <li>Value type instances are boxed when cast to an interface.</li>
  <li>An EIMI cannot be called by a derived type.</li>
</ul> 

Let’s take a closer look at these problems.

When examining the methods for a type in the .NET Framework reference documentation, explicit interface method implementations are listed, but no type-specific help exists; you can just read the general help about the interface methods. For example, the documentation for the Int32 type shows that it implements all of IConvertible interface's methods. This is good because developers know that these methods exist; however, this has been very confusing to developers because you can't call an IConvertible method on an Int32 directly. For example, the following method won't compile.
```C#
public static void Main() {
   Int32 x = 5;
   Singe s = x.ToSingle(null);  // Trying to call an IConvertible method
}
```

When compiling this method, the C# compiler produces the following message: messagepil17: 'int' does not contain a definition for 'ToSingle'. This error message confuses the developer because it's clearly stating that the Int32 type doesn’t define a ToSingle method when, in fact, it does.

To call ToSingle on an Int32, you must first cast the Int32 to an IConvertible, as shown in the following method:
```C#
public static void Main() {
   Int32 x = 5;
   Singe s = ((IConvertible) x).ToSingle(null);  // Trying to call an IConvertible method
}
```

Requiring this cast isn't obvious at all, and many developers won't figure this out on their own. But an even more troublesome problem exists: casting the Int32 value type to an IConvertible also boxes the value type, wasting memory and hurting performance. This is the second of the big problems I mentioned at the beginning of this section.

The third and perhaps the biggest problem with EIMIs is that they cannot be called by a derived class. Here is an example:
```C#
internal class Base : IComparable {
   Int32 IComparable.CompareTo(Object o) {
      Console.WriteLine("Base's CompareTo"); 
      return 0;     
   }  
}

internal sealed class Derived : Base, IComparable {  
   // A public method that is also the interface implementation
   public Int32 CompareTo(Object o) {
      Console.WriteLine("Derived's CompareTo");

      // This attempt to call the base class's EIMI causes a compiler error:
      // error CS0117: 'Base' does not contain a definition for 'CompareTo' 
      base.CompareTo(o); 
      return 0;
   }

}
```

The problem is that the Base class doesn't offer a public or protected CompareTo method that can be called; it offers a CompareTo method that can be called only by using a variable that is of the IComparable type. I could modify Derived’s CompareTo method so that it looks like the following:
```C#
internal sealed class Derived : Base, IComparable {  
   public Int32 CompareTo(Object o) {
      Console.WriteLine("Derived's CompareTo");
      
      // This attempt to call the base class's EIMI causes infinite recursion
      IComparable c = this;
      c.CompareTo(o);   // stackoverflow
      return 0;
   }

}
```

This could be fixed by declaring the Derived class without the IComparable interface but sometimes you cannot simply remove the interface from the type because you want the derived type to implement an interface method. The best way to fix this is for the base class to provide a virtual method in addition to the interface method that it has chosen to implement explicitly. Then the Derived class can override the virtual method. Here is the correct way to define the Base and Derived classes:

```C#
internal class Base : IComparable {

   // Explicit Interface Method Implementation
   Int32 IComparable.CompareTo(Object o) {
      Console.WriteLine("Base's IComparable CompareTo");
     return CompareTo(o); // This now calls the virtual method
   }
   
   // Virtual method for derived classes (this method could have any name)
   public virtual Int32 CompareTo(Object o) {
      Console.WriteLine("Base's virtual CompareTo");
      return 0;
   }
}

internal sealed class Derived : Base, IComparable {
   
   // A public method that is also the interface implementation
   public override Int32 CompareTo(Object o) {
      Console.WriteLine("Derived's CompareTo");
      // Now, we can call Base's virtual method
      return base.CompareTo(o);
   }
}
```

This discussion clearly shows you that EIMIs should be used with great care. When many developers first learn about EIMIs, they think that they're cool and they start using them whenever possible. Don’t do this! EIMIs are useful in some circumstances, but you should avoid them whenever possible because they make using a type much more difficult.

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