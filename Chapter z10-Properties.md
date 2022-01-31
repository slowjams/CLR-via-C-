Chapter 10-Properties
==============================

CLR offers two kinds of properties: **parameterless properties**, which are simply called *properties*, and **parameterful properties** which are called *indexers*

## Parameterless Properties

```C#
public sealed class Employee {
    private String m_Name;
    private Int32 m_Age;

    public String Name {
      get { return m_Name; }
      set { m_Name = value; } // The 'value' keyword always identifies the new value.
    } 

    public Int32 Age {
      get { return m_Age; }
      set {
          if (value < 0)
             throw new ArgumentOutOfRangeException("value", value.ToString(), "The value must be greater than or equal to 0");
          m_Age = value;
      }
    }

}
```

When you define a property, depending on its definition, the compiler will emit either two or three of the following items into the resulting managed assembly:

<ul>
  <li><b>A method representing the property’s get accessor method. This is emitted only if you define a get accessor method for the property</b></li>
  <li><b>A method representing the property’s set accessor method. This is emitted only if you define a set accessor method for the property</b></li>
  <li><b>A property definition in the managed assembly’s metadata. This is always emitted.</b></li>
</ul> 

Refer back to the Employee type shown earlier. As the compiler compiles this type, it comes across the Name and Age properties. Because both properties have get and set accessor methods, the compiler emits four method definitions into the Employee type. It's as though the original source were written as follows:

```C#
public sealed class Employee
{
    private String m_Name;
    private Int32 m_Age;
    public String get_Name() {
        return m_Name;
    }
    public void set_Name(String value) {
        m_Name = value; // The argument 'value' always identifies the new value.
    }
    public Int32 get_Age() {
        return m_Age;
    }
    public void set_Age(Int32 value) {
        if (value < 0) // The 'value' always identifies the new value.
            throw new ArgumentOutOfRangeException("value", value.ToString(),
            "The value must be greater than or equal to 0");
        m_Age = value;
    }
}
```

The compiler automatically generates names for these methods by prepending get_ or set_ to the property name specified by the developer

## Automatically Implemented Properties

If you are creating a property to simply encapsulate a backing field(private field), then C# offers a simplified syntax known as automatically implemented properties (AIPs), as shown here for the Name property:

```C#
public sealed class Employee
{
    // This property is an automatically implemented property
    public String Name { get; set; }
    private Int32 m_Age;
    public Int32 Age
    {
        get { return (m_Age); }
        set
        {
            if (value < 0) // The 'value' keyword always identifies the new value.
                throw new ArgumentOutOfRangeException("value", value.ToString(),
                "The value must be greater than or equal to 0");
            m_Age = value;
        }
    }
}
```

When you declare a property and do not provide an implementation for the get/set methods, then the C# compiler will automatically declare for you a private field. In this example, the field will be of type String, the type of the property. And, the compiler will automatically implement the get_Name and set_Name methods for you to return the value in the field and to set the field’s value, respectively.

You might wonder what the value of doing this is, as opposed to just declaring a public String field called Name. Well, there is a big difference. Using the AIP syntax means that you have created a property. Any code that accesses this property is actually calling get and set methods. If you decide later to implement the get and/or set method yourself instead of accepting the compiler's default implementation, then any code that accesses the property will not have to be recompiled. However, if you declared Name as a field and then you later change it to a property, then all code that accessed the field will have to be recompiled so that it now accesses the property methods.

<div class="alert alert-info p-1" role="alert">
   The runtime serialization engines persist the name of the field in a serialized stream. The name of the backing field for an AIP is determined by the compiler, and it could actually change the name of this backing field every time you recompile your code, negating the ability to deserialize instances of any types that contain an AIP. Do not use the AIP feature with any type you intend to serialize or deserialize. 
</div>

## Parameterful Properties

C# supports what I call parameterful properties, whose get accessor methods accept one or more parameters and whose set accessor methods accept two or more parameters. Different programming languages expose parameterful properties in different ways. Also, languages use different terms to refer to parameterful properties: C# calls them indexers and Visual Basic calls them default properties. In this section, I’ll focus on how C# exposes its indexers by using parameterful properties.

In C#, parameterful properties (indexers) are exposed using an array-like syntax. In other words, you can think of an indexer as a way for the C# developer to overload the [] operator:
```C#
public sealed class BitArray
{
    private Byte[] m_byteArray;
    private Int32 m_numBits;

    public BitArray(Int32 numBits) {
        if (numBits <= 0)
            throw new ArgumentOutOfRangeException("numBits must be > 0");
        m_numBits = numBits;

        m_byteArray = new byte[(numBits + 7) / 8];
    }

    public Boolean this[Int32 bitPos]
    {
        get
        {
            return (m_byteArray[bitPos / 8] & (1 << (bitPos % 8))) != 0;
        }

        set
        {
            if ((bitPos < 0) || (bitPos >= m_numBits))
                throw new ArgumentOutOfRangeException("bitPos", bitPos.ToString());
            if (value) {
                // Turn the indexed bit on.
                m_byteArray[bitPos / 8] = (Byte)
                (m_byteArray[bitPos / 8] | (1 << (bitPos % 8)));
            } else {
                // Turn the indexed bit off.
                m_byteArray[bitPos / 8] = (Byte)
                (m_byteArray[bitPos / 8] & ~(1 << (bitPos % 8)));
            }
        }
    }
}
```

The fact that C# requires `this[...]` as the syntax for expressing an indexer was purely a choice made by the C# team. What this choice means is that C# allows indexers to be defined only on instances of objects. C# doesn't offer syntax allowing a developer to define a static indexer property, although the CLR does support static parameterful properties.

For the BitArray class shown earlier, the compiler compiles the indexer as though the original source code were written as follows:
```C#
public sealed class BitArray {
   // This is the indexer's get accessor method.
   public Boolean get_Item(Int32 bitPos) { /* ... */ }

   // This is the indexer's set accessor method.
   public void set_Item(Int32 bitPos, Boolean value) { /* ... */ }
}
```

The compiler automatically generates names for these methods by prepending get_ and set_ to the indexer name. Because the C# syntax for an indexer doesn’t allow the developer to specify an indexer name, the C# compiler team had to choose a default name to use for the accessor methods; they chose Item. Therefore, the method names emitted by the compiler are `get_Item` and `set_Item`.

 C# does not allow define properties that introduce their own generic type parameters. If you want your object to expose some behavior—generic or not—define a method, not a property

## Object and Collection Initializers

It is very common to construct an object and then set some of the object's public properties (or fields). To simplify this common programming pattern, the C# language supports a special object initialization syntax. The following is an example:
```C#
Employee e = new Employee() { Name = "Jeff", Age = 45 };
```

The real benefit of the object initializer syntax is that it allows you to code in an expression context (as opposed to a statement context), permitting composability of functions, which in turn increases code readability. For example, I can now write the following:
```C#
String s = new Employee() { Name = "Jeff", Age = 45 }.ToString().ToUpper();
```
As a small side note, C# also lets you omit the parentheses before the open brace if you want to call a parameterless constructor. The following line produces the same IL as the preceding line:
```
String s = new Employee { Name = “Jeff”, Age = 45 }.ToString().ToUpper();
```

If a property's type implements the `IEnumerable` or `IEnumerable<T>` interface, then the property is considered to be a collection, and initializing a collection is an **additive** operation as opposed to a replacement operation. For example, suppose I have the following class definition:
```C#
public sealed class Classroom
{
    private List<String> m_students = new List<String>();
    public List<String> Students { get { return m_students; } }
    public Classroom() { }
}
```
I can now have code that constructs a Classroom object and initializes the Students collection as follows:
```C#
public static void M() {
    Classroom classroom = new Classroom
    {
        Students = { "Jeff", "Kristin", "Aidan", "Grant" }
    };
}
```
When compiling this code, the compiler sees that the Students property is of type `List<String>` and that this type implements the `IEnumerable<String>` interface. Now, the compiler assumes that the `List<String>` type offers a method called Add (because most collection classes actually offer an Add method that adds items to the collection). The compiler then generates code to call the collection’s Add method. So, the preceding code is converted by the compiler into the following.
```C#
public static void M() {
    Classroom classroom = new Classroom();
    classroom.Students.Add("Jeff");
    classroom.Students.Add("Kristin");
    classroom.Students.Add("Aidan");
    classroom.Students.Add("Grant");   
}
```

If the property’s type implements `IEnumerable` or `IEnumerable<T>` but the type doesn't offer an Add method, then the compiler does not let you use the collection initialize syntax to add items to the collection; instead, the compiler issues something like the following message: error CS0117:
`System.Collections.Generic.IEnumerable<string>` does not contain a definition for 'Add'.

Some collection’s Add methods take multiple arguments—for example, Dictionary's Add method:
```C#
public void Add(TKey key, TValue value);
```
You can pass multiple arguments to an Add method by using nested braces in a collection initializer, as follows:
```C#
var table = new Dictionary<String, Int32> {
   { "Jeffrey", 1 }, { "Kristin", 2 }, { "Aidan", 3 }, { "Grant", 4 }
};
```
The preceding line is identical to the following:
```C#
var table = new Dictionary<String, Int32>();
table.Add("Jeffrey", 1);
table.Add("Kristin", 2);
table.Add("Aidan", 3);
table.Add("Grant", 4);
```

## Anonymous Types
C#’s anonymous type feature allows you to automatically declare an immutable **tuple** type by using a very simple and succinct syntax. *A tuple type is a type that contains a collection of properties that are usually related to each other in some way.*

In the top line of the following code, I am defining a class with two properties (Name of type String, and Year of type Int32), constructing an instance of this type:
```C#
// Define a type, construct an instance of it, & initialize its properties
var o1 = new { Name = "Jeff", Year = 1964 };

// Display the properties on the console:
Console.WriteLine("Name={0}, Year={1}", o1.Name, o1.Year);
```

This top line of code creates an anonymous type because I did not specify a type name after the new keyword, so the compiler will create a type name for me automatically and not tell me what it is (which is why it is called an anonymous type). Now, let's focus on what the compiler is actually doing. When you write a line of code like this:
```C#
var o = new { property1 = expression1, ..., propertyN = expressionN };
```

the compiler infers the type of each expression, creates private fields of these inferred types, creates public **read-only** properties for each of the fields, and creates a constructor that accepts all these expressions. The constructor's code initializes the private read-only fields from the expression results passed in to it. In addition, the compiler overrides Object's Equals, GetHashCode, and ToString methods and generates code inside all these methods. The properties are readonly as opposed to read/write to help prevent the object’s hashcode from changing. Changing the hashcode for an object used as a key in a hashtable can prevent the object from being found.

The compiler supports two additional syntaxes for declaring a property inside an anonymous type where it can infer the property names and types from variables:
```C#
String Name = "Grant";
DateTime dt = DateTime.Now;

// Anonymous type with two properties
// 1. String Name property set to Grant
// 2. Int32 Year property set to the year inside the dt
var o2 = new { Name, dt.Year };
```
In this example, the compiler determines that the first property should be called **Name**. Because Name is the name of a local variable, the compiler sets the type of the property to be the same type as the local variable: String. For the second property, the compiler uses the name of the field/property: **Year**.

The compiler is very intelligent about defining anonymous types. If the compiler sees that you are defining multiple anonymous types in your source code that have the identical structure, the compiler will create just one definition for the anonymous type and create multiple instances of that type. By
"same structure", I mean that the anonymous types have the same type and name for each property and that these properties are specified in the same order.

## The System.Tuple Type

In the System namespace, Microsoft has defined several generic Tuple types (all derived from Object) that differ by arity (the number of generic parameters). Here is what the simplest and most complex ones essentially look like:
```C#
// This is the simplest:
[Serializable]
public class Tuple<T1>
{
    private T1 m_Item1;
    public Tuple(T1 item1) { m_Item1 = item1; }
    public T1 Item1 { get { return m_Item1; } }

    public override bool Equals(object obj);
    public override int GetHashCode();
    public override string ToString();
}


// This is the most complex:
[Serializable]
public class Tuple<T1, T2, T3, T4, T5, T6, T7, TRest>
{
    private T1 m_Item1; private T2 m_Item2; private T3 m_Item3; private T4 m_Item4;
    private T5 m_Item5; private T6 m_Item6; private T7 m_Item7; private TRest m_Rest;
    public Tuple(T1 item1, T2 item2, T3 item3, T4 item4, T5 item5, T6 item6, T7 item7, TRest rest) {
        m_Item1 = item1; m_Item2 = item2; m_Item3 = item3; m_Item4 = item4;
        m_Item5 = item5; m_Item6 = item6; m_Item7 = item7; m_Rest = rest;
    }
    public T1 Item1 { get { return m_Item1; } }
    public T2 Item2 { get { return m_Item2; } }
    public T3 Item3 { get { return m_Item3; } }
    public T4 Item4 { get { return m_Item4; } }
    public T5 Item5 { get { return m_Item5; } }
    public T6 Item6 { get { return m_Item6; } }
    public T7 Item7 { get { return m_Item7; } }
    public TRest Rest { get { return m_Rest; } }

    public override bool Equals(object obj);
    public override int GetHashCode();
    public override string ToString();
}
```

Like anonymous types, after a Tuple is created, it is immutable (all properties are read-only). I don't show it here, but the Tuple classes also offer CompareTo, Equals, GetHashCode, and ToString methods, as well as a Size property. In addition, all the Tuple types implement the IStructuralEquatable, IStructuralComparable, and IComparable interfaces so that you can compare two Tuple objects with each other to see how their fields compare with each other.

Here is an example of a method that uses a Tuple type to return two pieces of information to a caller:
```C#
// Returns minimum in Item1 & maximum in Item2
private static Tuple<Int32, Int32> MinMax(Int32 a, Int32 b) {
    return new Tuple<Int32, Int32>(Math.Min(a, b), Math.Max(a, b));
}

// This shows how to call the method and how to use the returned Tuple
private static void TupleTypes() {
    var minmax = MinMax(6, 2);
    Console.WriteLine("Min={0}, Max={1}", minmax.Item1, minmax.Item2); // Min=2, Max=6
}
```

Of course, it is very important that the producer and consumer of the Tuple have a clear understanding of what is being returned in the Item# properties. With anonymous types, the properties are given actual names based on the source code that defines the anonymous type. With Tuple types, the properties are assigned their Item# names by Microsoft and you cannot change this at all. Unfortunately, these names have no real meaning or significance, so it is up to the producer and consumer to assign meanings to them. This also reduces code readability and maintainability so you should add comments to your code explaining what the producer/consumer understanding is.

Tips: You can use Tuple's Equals method to simplify Equals as: `public override bool Equals(object other) => other is Point p && (p.X, p.Y, p.Z).Equals((this.x, this.y, this.z));`.

The compiler can only infer generic types when calling a generic method, not when you are calling a constructor. For this reason, the System namespace also includes a non-generic, static Tuple class containing a bunch of static Create methods that can infer generic types from arguments. This class acts as a factory for creating Tuple objects, and it exists simply to simplify your code. Here is a rewrite of the MinMax method shown earlier using the static Tuple class:

```C#
public static class Tuple {
    public static Tuple<T1> Create<T1>(T1 item1);
    public static Tuple<T1, T2> Create<T1, T2>(T1 item1, T2 item2);
    ...
    public static Tuple<T1, T2, T3, T4, T5, T6, T7> Create<T1, T2, T3, T4, T5, T6, T7>(T1 item1, T2 item2, T3 item3, T4 item4, T5 item5, T6 item6, T7 item7);
    public static Tuple<T1, T2, T3, T4, T5, T6, T7, Tuple<T8>> Create<T1, T2, T3, T4, T5, T6, T7, T8>(T1 item1, T2 item2, T3 item3, T4 item4, T5 item5, T6 item6, T7 item7, T8 item8);
}

private static Tuple<Int32, Int32> MinMax(Int32 a, Int32 b) {
   return Tuple.Create(Math.Min(a, b), Math.Max(a, b)); // Simpler syntax
}
```

If you want to create a Tuple with more than eight elements in it, then you would pass another Tuple for the Rest parameter as follows:
```C#
var t = Tuple.Create(0, 1, 2, 3, 4, 5, 6, Tuple.Create(7, 8));
Console.WriteLine("{0}, {1}, {2}, {3}, {4}, {5}, {6}, {7}, {8}", 
t.Item1, t.Item2, t.Item3, t.Item4, t.Item5, t.Item6, t.Item7, 
t.Rest.Item1.Item1, t.Rest.Item1.Item2);   // note that the last two are not t.Rest.Item1 or t.Rest.Item2
```

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