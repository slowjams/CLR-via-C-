Chapter 12-Generics
==============================

*Generics* is another mechanism offered by the common language runtime (CLR) and programming languages that provides one more form of code reuse: algorithm reuse. Here is what this class definition looks like 

```C#
public class List<T> : IList<T>, ICollection<T>, IEnumerable<T>, IList, ICollection, IEnumerable {
    public List();
    public void Add(T item);
    public Int32 BinarySearch(T item);
    ...
    public T this[Int32 index] { get; set; }
}
```
When defining a generic type or method, any variables it specifies for types (such as T) are called **type parameters**

Now that the generic `List<T>` type has been defined, other developers can use this generic algorithm by specifying the exact data type they would like the algorithm to operate on. When using a generic type or method, the specified data types are referred to as *type arguments*, Here is some code that shows this:
```C#
private static void SomeMethod() {
   List<DateTime> dtList = new List<DateTime>();
   // Add a DateTime object to the list
   dtList.Add(DateTime.Now);    // No boxing
   
   // Attempt to add a String object to the list
   dtList.Add("1/1/2004");      // Compile-time error
  
   // Extract a DateTime object out of the list
   DateTime dt = dtList[0];     // No cast required
}
```

Generics provide the following big benefits to developers as exhibited by the code just shown:
<ul>
  <li><b>Source code protection</b> The developer using a generic algorithm doesn’t need to have access to the algorithm’s source code.</li>
  <li><b>Type safety</b>When a generic algorithm is used with a specific type, the compiler and the CLR understand this and ensure that only objects compatible with the specified data type are used with the algorithm. Attempting to use an object of an incompatible type will result in either a compiler error or a run-time exception being thrown.</li>
  <li><b>Cleaner code</b>fewer casts are required in your source code, meaning that your code is easier to write and maintain.</li>
  <li><b>Better performance</b> Before generics, the way to define a generalized algorithm was to define all of its members to work with the Object data type. If you wanted to use the algorithm with value type instances, the CLR had to box the value type instance prior to calling the members of the algorithm. Boxing causes memory allocations on the managed heap, which causes more frequent garbage collections, which, in turn, hurt an application’s performance. Because a generic algorithm can now be created to work with a specific value type, <b>the instances of the value type can be passed by value</b>, and the CLR no longer has to do any boxing. In addition, because casts are not necessary, the CLR doesn’t have to check the type safety of the attempted cast, and this results in faster code too.</li>
</ul> 

Let's compare the memory images between storing value types with non-generic collections and generic collections to show why generic collections have better performance on value types:
```C#
//use non-generic collections, poor performance
ArrayList al = new ArrayList();
al.Add(1);
al.Add(2);

   Heap                    other sections of heap (might not be continuous)
-------------            ----------------
|   ptr1    |----------->|      1       | 
------------      ^       ----------------
|   ptr2    |-----|----->|      2       |
-------------     ^      ----------------
                  |____ CLR will box value type instances first i.e allocating memory from heap, 
                        copied value types' fields from stack to heap, hurting performance

//use generic collections to improve performance
List<int> lt = new List<int>();
lt.Add(1);
lt.Add(2);

   Heap                   
-----------            
|    1    |
-----------            
|    2    |
-----------                 
```

## Generics Infrastructure

Generics were added to version 2.0 of the CLR, and it was a major task that required many people working for quite some time. Specifically, to make generics work, Microsoft had to do the following:
<ul>
  <li>Create new Intermediate Language (IL) instructions that are aware of type arguments</li>
  <li>Modify the format of existing metadata tables so that type names and methods with generic parameters could be expressed.</li>
  <li>Modify the various programming languages(C#, Microsoft Visual Basic .NET, etc.) to support the new syntax, allowing developers to define and reference generic types and methods</li>
  <li>Modify the compilers to emit the new IL instructions and the modified metadata format</li>
  <li>Modify the just-in-time (JIT) compiler to process the new type-argument–aware IL instructions that produce the correct native code.</li>
  <li>Create new reflection members so that developers can query types and members to determine if they have generic parameters. Also, new reflection emit members had to be defined so that developers could create generic type and method definitions at run time.</li>
  <li>Modify the debugger to show and manipulate generic types, members, fields, and local variables.</li>
  <li>Modify the Microsoft Visual Studio IntelliSense feature to show specific member prototypes when using a generic type or a method with a specific data type.
</li>
</ul> 

## Open and Closed Types
A type with generic type parameters is called an *open type*, and the CLR does not allow any instance of an open type to be constructed. If actual data
types are passed in for all of the type arguments, the type is called a *closed type* and the CLR does allow instances of a closed type to be constructed. It should also point out that the CLR allocates a type's static fields inside the type object (as discussed in Chapter 4, "Type Fundamentals"). So each closed type has its own static fields. In other words, if `List<T>` defined any static fields, these fields are not shared between a `List<DateTime>`
and a `List<String>`; each closed type object has its own static fields. Also, if a generic type defines a static constructor (discussed in Chapter 8, "Methods"), this constructor will execute once per closed type.

## Generic Types and Inheritance
If a linked-list node class is defined like this:
```C#
internal sealed class Node<T>
{
    public T m_data;
    public Node<T> m_next;

    public Node(T data) { }

    public Node(T data, Node<T> next) {
        m_data = data; m_next = next;
    }

    public override string ToString() {
        return m_data.ToString() + ((m_next != null) ? m_next.ToString() : String.Empty);
    }
}
```
then I can write some code to build up a linked list that would look something like the following:
```C#
private static void SameDataLinkedList() {
    Node<Char> head = new Node<Char>('C');
    head = new Node<char>('B', head);
    head = new Node<Char>('A', head);
    Console.WriteLine(head.ToString()); // Displays "ABC"
}
```
In the Node class just shown, the m_next field must refer to another node that has the same kind of data type in its m_data field. This means that the linked list must contain nodes in which all data items are of the same type (or derived type). For example, I can't use the Node class to create a linked list in which one element contains a Char, another element contains a DateTime and another element contains a String. Well, I could if I use Node<Object> everywhere, but then I would lose compile-time type safety, and value types would get boxed


So a better way to go would be to define a non-generic Node base class and then define a generic TypedNode class. Now, I can have a linked list in which each
node can be of a specific data type (not Object), get compile-time type safety, and avoid the boxing of value types. Here are the new class definitions:
```C#
internal class Node
{
    protected Node m_next;

    public Node(Node next) {
        m_next = next;
    }
}

internal sealed class TypedNode<T> : Node
{
    public T m_data;

    public TypedNode(T data) : this(data, null) { }

    public TypedNode(T data, Node next) : base(next) {
        m_data = data;
    }

    public override String ToString() {
        return m_data.ToString() + ((m_next != null) ? m_next.ToString() : String.Empty);
    }
}
```
I can now write code to create a linked list in which each node is a different data type. The code could look something like the following:
```C#
private static void DifferentDataLinkedList() {
    Node head = new TypedNode<Char>('.');
    head = new TypedNode<DateTime>(DateTime.Now, head);
    head = new TypedNode<String>("Today is ", head);
    Console.WriteLine(head.ToString());
}
```

## Code Explosion
When a method that uses generic type parameters is JIT-compiled, the CLR takes the method's IL, substitutes the specified type arguments, and then creates native code that is specific to that method operating on the specified data types. This is exactly what you want and is one of the main features of generics. However, there is a downside to this: **the CLR keeps generating native code for every method/type combination.** This is referred to as code explosion. This can end up increasing the application's working set substantially, thereby hurting performance.

Fortunately, the CLR has some optimizations built into it to reduce code explosion. First, if a method is called for a particular type argument, and later, the method is **called again using the same type argument, the CLR will compile the code for this method/type combination just once.** So if one assembly uses `List<DateTime>`, and a completely different assembly (loaded in the same AppDomain) also uses `List<DateTime>`, the CLR will compile the methods for `List<DateTime>` just once. This reduces code explosion substantially. 

The CLR has another optimization: **the CLR considers all reference type arguments to be identical, and so again, the code can be shared.** For example, the code compiled by the CLR for `List<String>`s methods can be used for `List<Stream>`'s methods, because String and Stream are both reference types. In fact, for any reference type, the same code will be used. The CLR can perform this optimization because **all reference type arguments or variables are really just pointers** to objects on the heap, and object pointers are all manipulated in the same way. 

But if any type argument is a value type, the CLR must produce native code specifically for that value type.

## Generic Interfaces
Obviously, the ability to define generic reference and value types was the main feature of generics. However, it was critical for the CLR to also allow generic interfaces. Without generic interfaces, any time you tried to manipulate a value type by using a non-generic interface (such as IComparable), boxing and a loss of compile-time type safety would happen again. This would severely limit the usefulness of generic types. And so the CLR does support generic interfaces. A reference or value type can implement a generic interface by specifying type arguments, or a type can implement a generic interface by leaving the type arguments unspecified. Let’s look at some examples:
```C#
public interface IEnumerator<T> : IDisposable, IEnumerator { T Current { get; } }
```
Here is an example of a type that implements this generic interface and that specifies type arguments:
```C#
internal sealed class Triangle : IEnumerator<Point> {
   private Point[] m_vertices;
 
   // IEnumerator<Point>'s Current property is of type Point
   public Point Current { get { ... } }
   ...
}
```
Now let's look at an example of a type that implements the same generic interface but with the type arguments **left unspecified**:
```C#
internal sealed class ArrayEnumerator<T> : IEnumerator<T> {
   private T[] m_array; 
   public T Current { get { ... } }
}
```


## Interface and Delegate Contra-variant and Covariant Generic Type Arguments

Each of an interface/delegate's generic type parameters can be marked as *covariant* or *contra-variant*. This feature allows you to cast a variable of a generic interface/delegate type to the same interface/delegate type where the generic parameter types differ. Covariance and contravariance are collectively referred to as *variance*. A generic type parameter can be any one of the following:
<ul>
  <li><b>Invariant</b> Meaning that the generic type parameter cannot be changed. (This is the default when you are not using in/out keyword)</li>
  <li><b>Covariant</b> In C#, you indicate covariant generic type parameters with the <code>out</code> keyword, meaning that the generic type argument can change from a class to one of its base classes. </li>
  <li><b>Contra-variant</b> In C#, you indicate contra-variant generic type parameters with the <code>in</code> keyword, meaning that the generic type parameter can change from a class to a class derived from it (contra- means "against; opposite", becuase it is like casting from base type to derived type, which is against the normal flow "derived to base")</li>
</ul> 

<div class="alert alert-info p-1" role="alert">
    Variance applies only to reference types, if you specify a value type for a variant type parameter, that type parameter is invariant for the resulting constructed type.
</div>

For example, let's say that the following delegate type definition exists (which, by the way, it does):
```C#
public delegate TResult Func<in T, out TResult>(T arg);
```
Here, the generic type parameter T is marked with the in keyword, making it contra-variant; and the generic type parameter TResult is marked with the out keyword, making it covariant. So now, if I have a variable declared as follows:
```C#
Func<Object, ArgumentException> fn1 = null;
```
I can cast it to another Func type, where the generic type parameters are different:
```C#
Func<String, Exception> fn2 = fn1; // No explicit cast is required here
Exception e = fn2("");
```

Note that `in` and `out`keyword in two contexts:
<ul>
  <li>As a parameter modifier, which lets you pass an argument to a method by reference rather than by value.</li>
  <li>In generic type parameter declarations for interfaces and delegates, which specifies that a type parameter is covariant or contravariant</li>
</ul> 

Since they are different things in different context, therefore they cannot be combined and used together, which means variance is not allowed on a generic type parameter if an argument of that type is passed to a method by using the `out` or `ref` keyword. For example, the line of code below causes the compiler to generate the following error message: 

```C#
// doesn't compile 
//Error message: Invalid variance: The type parameter 'T' must be invariantly valid on 'SomeDelegate<T>.
delegate void SomeDelegate<in T>(ref T t);
```

Variance applies only if the compiler can verify that a reference conversion exists between types. In other words, variance is not possible for value types because boxing would be required. In my opinion, this restriction is what makes these variance features not that useful. For example, if I have the following method:
```C#
void ProcessCollection(IEnumerable<Object> collection) { ... }
```
I can't call it passing in a reference to a `List<DateTime>` object(`DateTime` is struct) because a reference conversion doesn’t exist between the `DateTime` value type and Object even though DateTime is derived from Object. You can solve this problem by declaring ProcessCollection as follows:
```C#
void ProcessCollection<T>(IEnumerable<T> collection) { ... }
```
But there is a big benefit of using `ProcessCollection(IEnumerable<Object> collection)` because there is only one version of the JITted code. With `ProcessCollection<T>­ (IEnumerable<T>collection)`, there is also only one version of the JITted code shared by all Ts that are reference types, but you do get other versions of JITted code for Ts that are value types, but now you can at least call the method passing it a collection of value types.

When using delegates that take generic arguments and return types, it is recommended to always specify the in and out keywords for contra-variance and covariance whenever possible, because doing this has no ill effects and enables your delegate to be used in more scenarios.

Like delegates, an interface with generic type parameters can have its type parameters be contravariant or covariant. Here is an example of an interface with a covariant generic type parameter:
```C#
public interface IEnumerator<out T> : IEnumerator {
   Boolean MoveNext();
   T Current { get; }
}
```
Because T is covariant, it is possible to have the following code compile and run successfully:
```C#
// This method accepts an IEnumerable of any reference type
Int32 Count(IEnumerable<Object> collection) { ... }
...
// The call below passes an IEnumerable<String> to Count
Int32 c = Count(new[] { "Grant" });
```

Below is a general code example why contra-variance is useful:
```C#
class AnimalWalker<in T> where T: Animal { 
   public void walk(T animal) { 
      animal.walk(); 
   } 
}

AnimalWalker<Animal> animalWalker = new AnimalWalker<Animal>();
AnimalWalker<Dog> dogWalker = animalWalker; 
dogWalker.walk(new Dog()); 
```
You might think contra-variance means casting a base type to derived type, which doesn't make sense in the beginning, but when you look at the code, you will realize that contra-variance really invert/reverse the casting flow from "base type to derived type" back to "derived type to base type" in the interface's method, that's why `in` keyword (input as context meaning) is choose to represent contra-variance (type parameter used as an input of the interface's method, and that's why contra-variant generic type parameters can appear only in input positions such as a method's argument as mentioned before).

Below is a concrete code example to demostrate the use of contra-variance:
```C#
public interface IComparer<in T> { int Compare(T x, T y); }

public class SortedSet<T> : ... {
   ...
   public SortedSet(IComparer<T> comparer);
   ...
}

abstract class Shape {
    public virtual double Area { get { return 0; } }
}

class Circle : Shape {
    private double r;
    public Circle(double radius) { r = radius; }
    public double Radius { get { return r; } }
    public override double Area { get { return Math.PI * r * r; } }
}

class ShapeAreaComparer : IComparer<Shape> {
    int IComparer<Shape>.Compare(Shape a, Shape b) {
        if (a == null)
            return b == null ? 0 : -1;
        return b == null ? 1 : a.Area.CompareTo(b.Area);
    }
}

class Program
{
    static void Main(string[] args) {
        // Even though the constructor for SortedSet<Circle> expects IComparer<Circle>, 
        // You can pass ShapeAreaComparer, which implements IComparer<Shape>,
        // because type parameter T of IComparer<in T> is contravariant.
        SortedSet<Circle> circlesByArea = new SortedSet<Circle>(new ShapeAreaComparer())
               { new Circle(7.2), new Circle(100), null, new Circle(.01) };
    }
}
```


## Generic Methods
When you define a generic class, struct, or interface, any methods defined in these types can refer to a type parameter specified by the type. A type parameter can be used as a method's parameter, a method's return type, or as a local variable defined inside the method. However, the CLR also supports the ability for a method to specify its very own type parameters. And these type parameters can also be used for parameters, return types, or local variables. 
Here is a somewhat contrived example of a type that defines a type parameter and a method that has its very own type parameter:
```C#
internal sealed class GenericType<T> {
    private T m_value;
    public GenericType(T value) { m_value = value; }
    public TOutput Converter<TOutput>() {
       TOutput result = (TOutput) Convert.ChangeType(m_value, typeof(TOutput));
       return result;
    }
}
```
A reasonably good example of a generic method is the Swap method:
```C#
private static void Swap<T>(ref T o1, ref T o2) {
   T temp = o1;
   o1 = o2;
   o2 = temp;
}

private static void CallingSwap() {
   Int32 n1 = 1, n2 = 2;
   Swap<Int32>(ref n1, ref n2);

   String s1 = "Aidan", s2 = "Grant";
   Swap<String>(ref s1, ref s2);
}
```
Using generic types with methods that take out and ref parameters can be particularly interesting because the variable you pass as an out/ref argument must be the same type as the method's parameter to avoid a potential type safety exploit.

## Generic Methods
C# generic syntax can be confusing with all of its less-than and greaterthan signs. To help improve code creation, readability, and maintainability, the C# compiler offers type inference when calling a generic method. Type inference means that the compiler attempts to determine (or infer) the type to use automatically when calling a generic method. Here is some code that demonstrates type inference:
```C#
private static void CallingSwapUsingInference() {
   Int32 n1 = 1, n2 = 2;
   Swap(ref n1, ref n2);// Calls Swap<Int32> 

   String s1 = "Aidan";
   Object s2 = "Grant";
   Swap(ref s1, ref s2);// Error, type can't be inferred 
}
```
In this code, notice that the calls to Swap do not specify type arguments in less-than/greater-than signs. In the first call to Swap, the C# compiler was able to infer that n1 and n2 are Int32s, and therefore, it should call Swap by using an Int32 type argument. In the second call to Swap, C# sees that s1 is a String and s2 is an Object (even though it happens to refer to a String). Because s1 and s2 are variables of different data types, the compiler can’t accurately infer the type to use for Swap's type argument

A type can define multiple methods with one of its methods taking a specific data type and another taking a generic type parameter, as in the following example:
```C#
private static void Display(String s) {
   Console.WriteLine(s);
}

private static void Display<T>(T o) {
   Display(o.ToString()); // Calls Display(String)
}
```
C# compiler always prefers a more explicit match over a generic match unless you specify a generic type argument such as `Display<String>("Aidan"); `

## Verifiability and Constraints
When compiling generic code, the C# compiler analyzes it and ensures that the code will work for any type that exists today or that may be defined in the future. Let’s look at the following method:
```C#
private static Boolean MethodTakingAnyType<T>(T o) {
   T temp = o;
   Console.WriteLine(o.ToString());
   Boolean b = temp.Equals(o);
   return b; 
}
```
This method works for any type. If T is a reference type, it works. If T is a value or enumeration type, it works. If T is an interface or delegate type,
it works. This method works for all types that exist today or that will be defined tomorrow because every type supports assignment and calls to methods defined by Object (such as `ToString` and `Equals`).

But look at the following method:
```C#
private static T Min<T>(T o1, T o2) {
   if (o1.CompareTo(o2) < 0) return o1;    //compile error CS1061: 'T' does not contain a definition for 'CompareTo' accepting a first argument of type 'T' could be found 
   return o2;
}
```
There are lots of types that do NOT offer a CompareTo method, and therefore, the C# compiler can't compile this code and guarantee that this method would work for all types.

Fortunately, compilers and the CLR support a mechanism called constraints that you can take advantage of to make generics useful again.

A constraint is a way to limit the number of types that can be specified for a generic argument. Limiting the number of types allows you to do more with those types. Here is a new version of the Min method that specifies a constraint:
```C#
public static T Min<T>(T o1, T o2) where T : IComparable<T> {
   if (o1.CompareTo(o2) < 0) return o1;
   return o2;
}
```

## Primary Constraints
A primary constraint can be a reference type that identifies a class that is not sealed. You cannot specify one of the following special reference types: `System.Object`, `System.Array`, `System.Delegate`(it is supported on C# on latest version), `System.MulticastDelegate`, `System.ValueType`, `System.Enum`, or `System.Void`

When specifying a reference type constraint, you are promising the compiler that a specified type argument will either be of the same type or of a type derived from the constraint type. For example, see the following generic class:
```C#
internal sealed class PrimaryConstraintOfStream<T> where T : Stream {
   public void M(T stream) {
      stream.Close();  // OK
   }
}
```

There are two special primary constraints: **class** and **struct**. The *class* constraint promises the compiler that a specified type argument will be a reference type. Any class type, interface type, delegate type, or array type satisfies this constraint. For example, see the following generic class:
```C#
internal sealed class PrimaryConstraintOfClass<T> where T : class {
   public void M() {
      T temp = null;  // Allowed because T must be a reference type
   }
}
```
In this example, setting temp to null is legal because T is known to be a reference type, and all reference type variables can be set to null. If T were unconstrained, the preceding code would not compile because T could be a value type, and value type variables cannot be set to null.

The *struct* constraint promises the compiler that a specified type argument will be a value type. Any value type, including enumerations, satisfies this constraint. However, the compiler and the CLR treat any `System.Nullable<T>` value type as a special type, and nullable types do not satisfy this
constraint. The reason is because the `Nullable<T>` type constrains its type parameter to struct, and the CLR wants to prohibit a recursive type such as `Nullable<Nullable<T>>`. Nullable types are discussed in Chapter 19, "Nullable Value Types."

Here is an example class that constrains its type parameter by using the struct constraint:
```C#
internal sealed class PrimaryConstraintOfStruct<T> where T : struct {
   public static T Factory() {
      // Allowed because all value types implicitly
      // have a public, parameterless constructor
      return new T();
   }
}
```
Note that if T were unconstrained, constrained to a reference type, or constrained to class, the above code would not compile because some reference types do
not have public, parameterless constructors.

## Secondary Constraints
A type parameter can specify zero or more secondary constraints where a secondary constraint represents an interface type. When specifying an interface type constraint, you are promising the compiler that a specified type argument will be a type that implements the interface. And because you can specify multiple interface constraints, the type argument must specify a type that implements all of the interface constraints (and all of the primary constraints too, if specified).

There is another kind of secondary constraint called a type parameter constraint (sometimes referred to as a naked type constraint). This kind of constraint is used much less often than an interface constraint. It allows a generic type or method to indicate that there must be a relationship between specified type arguments. A type parameter can have zero or more type constraints applied to it. Here is a generic method that demonstrates the use of a type parameter constraint:
```C#
private static List<TBase> ConvertIList<T, TBase>(IList<T> list) where T : TBase {
   List<TBase> baseList = new List<TBase>(list.Count);
   for (Int32 index = 0; index < list.Count; index++) {
      baseList.Add(list[index]);
   }
   return baseList; 
}
```

## Constructor Constraints
When specifying a constructor constraint, you are promising the compiler that a specified type argument will be a non-abstract type that implements a public, parameterless constructor. Here is an example class that constrains its type parameter by using the constructor constraint:
```C#
internal sealed class ConstructorConstraint<T> where T : new() {
   public static T Factory() {
      // Allowed because all value types implicitly
      // have a public, parameterless constructor and because
      // the constraint requires that any specified reference
      // type also have a public, parameterless constructor
      return new T();
   }
}
```
In this example, newing up a T is legal because T is known to be a type that has a public, parameterless constructor. This is certainly true of all value types, and the constructor constraint requires that it be true of any reference type specified as a type argument. 

Sometimes developers would like to declare a type parameter by using a constructor constraint whereby the constructor takes various parameters itself. As of now, the CLR (and therefore the C# compiler) supports **only parameterless constructors**. Microsoft feels that this will be good enough for almost all scenarios.

## Casting a Generic Type Variable
Casting a generic type variable to another type is illegal unless you are casting to a type compatible with a constraint:
```C#
private static void CastingAGenericTypeVariable1<T>(T obj) {
   Int32 x = (Int32) obj;   // Error
   String s = (String) obj; // Error
}
```
The compiler issues an error on both lines above because T could be any type, and there is no guarantee that the casts will succeed. You can modify this code to get it to compile by casting to Object first:
```C#
private static void CastingAGenericTypeVariable2<T>(T obj) {
   Int32 x = (Int32) (Object) obj;     // No error
   String s = (String) (Object) obj;   // No error
   String s1 = obj as String;          // No error
}
```
Although this code will now compile, it is still possible for the CLR to throw an InvalidCastException at run time.

## Setting a Generic Type Variable to a Default Value
Setting a generic type variable to null is illegal unless the generic type is constrained to a reference type.
```C#
private static void SettingAGenericTypeVariableToNull<T>() {
   T temp = null; // CS0403 – Cannot convert null to type parameter 'T' because it could
   // be a non-nullable value type. Consider using 'default(T)' instead
}
```
Because T is unconstrained, it could be a value type, and setting a variable of a value type to null is not possible. If T were constrained to a reference type, setting temp to null would compile and run just fine.

Microsoft’s C# team felt that it would be useful to give developers the ability to set a variable to a default value. So the C# compiler allows you to use the default keyword to accomplish this:
```C#
private static void SettingAGenericTypeVariableToDefaultValue<T>() {
   T temp = default(T); // OK
}
```
The use of the default keyword above tells the C# compiler and the CLR’s JIT compiler to produce code to set temp to null if T is a reference type and to set temp to all-bits-zero if T is a value type.

## Comparing a Generic Type Variable with null
Comparing a generic type variable to null by using the == or != operator is legal regardless of whether the generic type is constrained:
```C#
private static void ComparingAGenericTypeVariableWithNull<T>(T obj) {
   if (obj == null) { /* Never executes for a value type */ }   // compile OK
}
```
Because T is unconstrained, it could be a reference type or a value type. If T is a value type, obj can never be null. Normally, you’d expect the C# compiler to issue an error because of this. However, the C# compiler does not issue an error; instead, it compiles the code just fine. When this method is called using a type argument that is a value type, the JIT compiler sees that the if statement can never be true, and the JIT compiler will not emit the native code for the if test or the code in the braces. If I had used the != operator, the JIT compiler would not emit the code for the if test (because it is always true), and it will emit the code inside the if's braces. This is a great feature provided by JIT compiler.

By the way, if T had been constrained to a struct, the C# compiler would issue an error because you shouldn’t be writing code that compares a value type variable with null because the result is always the same.

## Comparing Two Generic Type Variables with Each Other
Comparing two variables of the same generic type is illegal if the generic type parameter is not known to be a reference type:
```C#
private static void ComparingTwoGenericTypeVariables<T>(T o1, T o2) {
   if (o1 == o2) { }    // compile rror
}
```
In this example, T is unconstrained, and whereas it is legal to compare two reference type variables with one another, it is not legal to compare two value type variables with one another unless the value type overloads the == operator. If T were constrained to class, this code would compile.

When you write code to compare the primitive value types—Byte, Int32, Single, Decimal, etc.—the C# compiler knows how to emit the right code. However, for non-primitive value types, the C# compiler doesn’t know how to emit the code to do comparisons. So if ComparingTwoGenericTypeVariables method's T were constrained to struct, the compiler would issue an error. And you're not allowed to constrain a type parameter to a specific value type because it is implicitly sealed, and therefore no types exist that are derived from the value type. Allowing this would make the generic method constrained to a specific type, and the C# compiler doesn't allow this because it is more efficient to just make a non-generic method.

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