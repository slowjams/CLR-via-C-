Chapter 16-Arrays
==============================

Arrys are mechanisms that allow you to treat several items as a single collection. The Microsoft .NET common language runtime (CLR) supports single-dimensional array, multi-dimensional arrays, and jagged arrays (this is, arrays of arrays). All array types are implicitly derived from the `System.Array` abstract class, which itself is derived from `System.Object`. 
```C#
public abstract class Array : ICloneable, IList, ICollection, IEnumerable, IStructuralComparable, IStructuralEquatable {
   ...
   public static void Copy(Array sourceArray, Array destinationArray, int length);
   public static void Copy(Array sourceArray, int sourceIndex, Array destinationArray, int destinationIndex, int length);
   public static Array CreateInstance(Type elementType, int[] lengths, int[] lowerBounds);

   public static int BinarySearch(Array array, int index, int length, object value, IComparer comparer);
   public static int IndexOf<T>(T[] array, T value, int startIndex, int count);
   public static void Clear(Array array, int index, int length);

   public int Length { get; }
   public int GetLowerBound(int dimension);
   public int GetUpperBound(int dimension);
   public object GetValue(int index);
   public void SetValue(object value, int index);
   ...
   Object IList.this[int index] {
      get { return GetValue(index); }
      set { SetValue(value, index); }
   }

   Boolean IStructuralEquatable.Equals(Object other, IEqualityComparer comparer) {
      if (other == null)
         return false;
      if (Object.ReferenceEquals(this, other)) {
         return true;  
      }

      Array o = other as Array;
      if (o == null || o.Length != this.Length)
         return false;
      
      int i = 0;  
      while (i < o.Length) {
         object left = GetValue(i); 
         object right = o.GetValue(i); 
         if (!comparer.Equals(left, right))
            return false;
         i++;
      }
      return true;
   }

   int IStructuralEquatable.GetHashCode(IEqualityComparer comparer) {
      if (comparer == null)
         throw new ArgumentNullException("comparer");
      ...
      int ret = 0; 
      
      for (int i = (this.Length >= 8 ? this.Length - 8 : 0); i < this.Length; i++) {
         ret = CombineHashCodes(ret, comparer.GetHashCode(GetValue(i)));  
      }
      return ret;
   }
}

public interface IStructuralEquatable {
   bool Equals(object other, IEqualityComparer comparer);
   int GetHashCode(IEqualityComparer comparer);  
}
```
This means that arrays are always reference types that are allocated on the managed heap and that your application's varaiable or field contains a reference to the array and not the elements of the array itself. AThe following code makes this clearer:
```C#
Int32[] myIntegers;           // Declares a reference to an array
myIntegers = new Int32[100];  // Creates an array of 100 Int32s
```

On the first line, myIntegers is a variable that's capable of pointing to a single-dimensional array of Int32s, Initially, myIntegers will be set to null because I haven't allocated an array. the second line of code allocates an array of 100 Int32 values; all of the Int32s are initialized to 0. Because
arrays are reference types, the memory block required to hold the 100 unboxed Int32s is allocated on the managed heap. Actually, in addition to the array's elements, the memory block occupied by an array object also contains a type object pointer, a sync block index, and some additional overhead members as well. The address of this array's memory block is returned and saved in the variable myIntegers.

Common Language Specification (CLS) compliance requires all arrays to be zero-based. CLR does support non-zero–based arrays even though their use is discouraged. For those of you who don't care about a slight performance penalty or cross-language portability, I'll demonstrate how to create and use non-zero–based arrays later in this chapter.

![alt text](./zImages/16-1.png "Title")

Notice in Figure 16-1 that each array has some additional overhead information associated with it. This infomation contains the rank of the array (number of dimensions), the lower bounds for each dimension of the array (almost always 0), and the length of each dimension. The overhead also contains the array's element type. I'll mention the methods that allow you to query this overhead information later in this chapter.

So far, I've shown examples demonstrating how to create single-dimensional arrays. When possible, you should stick with single-dimensional, zero-based arrays, sometimes refered to as ***SZ arrays***, or ***vectors***. Vectors give the best performance because you can use specific IL instructions. However, if you prefer to work with multi-dimensional arrays, you can. Here are some examples of multi-dimensional arrays:
```C#
// Create a two-dimensional array of Doubles.
Double[,] myDoubles = new Double[10,20];

// Create a three-dimensional array of String references. 
String[,,] myStrings = new String[5,3,10];
```

The CLR also supports jagged arrays, which are arrays of arrays. Zero-based, single-dimensional jagged arrays have the same performance as normal vectors, However, accessing the elements of a jagged array means that two or more array access must occur. Here are some examples of how to create an array of polygons with each polygon consisting of an array of Point instances:
```C#
// Create a single-dimensional array of Point arrays.
Point[][] myPolygons = new Point[3][];

// myPolygons[0] refers to an array of 10 Point instances. 
myPolygons[0] = new Point[10];

// myPolygons[1] refers to an array of 20 Point instances.
myPolygons[1] = new Point[20];

// myPolygons[2] refers to an array of 30 Point instances. 
myPolygons[2] = new Point[30];

// Display the Points in the first polygon.
for (Int32 x = 0; x < myPolygons[0].Length; x++) 
   Console.WriteLine(myPolygons[0][x]);
```
Because a jagged array is an array of arrays. Each array is not guaranteed to be of the same size. so you can't do:
```C#
Point[][] myPolygons = new Point[3][10];   //compile error
```

<div class="alert alert-info p-1" role="alert">
    The CLR verifies that an index into an array is valid. In other words, you can't create an array with 100 elements in it (numbered 0 through 99) and then try to access the element at index –5 or 100. Doing so will cause a System.IndexOutOfRangeException to be thrown. Allowing access to memory outside the range of an array would be a breach of type safety and a potential security hole, and the CLR doesn’t allow verifiable code to do this. Usually, the performance degradation associated with index checking is insubstantial because the just-in-time (JIT) compiler normally checks array bounds once before a loop executes instead of at each loop iteration. However, if you’re still concerned about the performance hit of the CLR’s index checks, you can use unsafe code in C# to access the array.
</div>

## Initializing Array Elements

In the previous section, I showed how to create an array object and then I showed how to initialize the elements of the array. C# offers syntax that allows you to do these two operations in one statement. The following shows an example:
```C#
String[] names = new String[] { "Aidan", "Grant" };
```
The comma-separated set of tokens contained within the braces is called an ***array initializer***. Each token can be an arbitrarily complex expression or, in the case of a multi-dimensional array, a nested array initializer. In the preceding exampe, I used just two simple String expressions.

If you are declaring a local variable in a method to refer to the initialized array, then you can use C#'s implicitly typed local variable (var) feature to simplify the code a little:
```C#
var names = new String[] { "Aidan", "Grant" };
```

You can use C#'s implicitly typed array feature to have the compiler infer the type of the array's element. Notice the following line has no type specified between new and []:
```C#
var names = new[] { "Aidan", "Grant", null };
```

In the preceding line, the compiler examines the types of the expressions being used inside the array to initialize the array’s elements, and the compiler chooses the closest base class that all the elements have in common to determine the type of the array. In this example, the compiler sees two Strings and null. Because null is implicitly castable to any reference type (including String), the compiler infers that it should be creating and initializing an array of String references. But if you had this code:
```C#
var names = new[] { "Aidan", "Grant", 123 };  // compile error
```
the compiler would issue the message error CS0826: No best type found for implicitlytyped array. This is because the base type in common between the two Strings and the Int32 is Object, which would mean that the compiler would have to create an array of Object references and then box the 123 and have the last array element refer to a boxed Int32 with a value of 123. The C# compiler team thinks that boxing array elements is too heavy-handed for the compiler to do for
you implicitly, and that is why the compiler issues the error.

As an added syntactical bonus when initializing an array, you can write the following:
```C#
String[] names = { "Aidan", "Grant" };
```
Notice that on the right of the assignment operator ==, only the array initializer expression is given with no new, no type and no []. This syntax is nice, but unfortunately, the C# compiler does not allow you to use implicitly typed local variables with this syntax:
```C#
var names = { "Aidan", "Grant" };  // compile error
```
If you try to compile the preceding line of code, the compiler issues two messages: error CS0820: Cannot initialize an implicitly-typed local variable with an array initializer and error CS0622: Can only use array initializer expressions to assign to array types. Try using a new expression instead. Although the compiler could make this work, the C# team thought that the compiler would be doing too much for you here. It would be inferring the type of the array, new’ing the array, initializing the array, and inferring the type of the local variable, too.

The last thing I'd like to show you is how to use implicitly typed arrays with anonymous types and implicitly typed local variables. Anonymous types and how type identity applies to them are discussed in Chapter 10, “Properties.” Examine the following code:
```C#
// Using C#’s implicitly typed local, implicitly typed array, and anonymous type features:
var kids = new[] {new { Name="Aidan" }, new { Name="Grant" }};

// Sample usage (with another implicitly typed local variable):
foreach (var kid in kids)
   Console.WriteLine(kid.Name);
```

In this example, I am using an array initializer that has two expressions for the array elements. Each expression represents an anonymous type (because no type name is specified after the new operator). Because the two anonymous types have the identical structure (one field called Name of type String), the compiler knows that these two objects are of the exact same type. Now, I use C#’s implicitly typed array feature (no type specified between the new and the []) so that the compiler will infer the type of the array itself, construct this array object, and initialize its references to the two instances of the one anonymous type.1 Finally, a reference to this array object is assigned to the kids local variable, the type of which is inferred by the compiler due to C#'s implicitly typed local variable feature.

## Casting Arrays

For arrays with reference type elements, the CLR allows you to implicitly cast the source array's element type to a target type. For the cast to succeed, both array types must have the same number of dimensions, and an implicit or explicit conversion from the source element type to the target element type must exist. The CLR doesn't allow the casting of arrays with value type elements to any other type. (However, by using the Array.Copy method, you can create a new array and populate its elements in order to obtain the desired effect.) The following code demonstrates how array casting works:
```C#
// Create a two-dimensional FileStream array.
FileStream[,] fs2dim = new FileStream[5, 10];

// Implicit cast to a two-dimensional Object array
Object[,] o2dim = fs2dim; 

// Can't cast from two-dimensional array to one-dimensional array
Stream[] s1dim = (Stream[]) o2dim;   // Compiler error CS0030: Cannot convert type 'object[*,*]' to 'System.IO.Stream[]'

// Explicit cast to two-dimensional Stream array
Stream[,] s2dim = (Stream[,]) o2dim;

// Explicit cast to two-dimensional String array
// Compiles but throws InvalidCastException at runtime
String[,] st2dim = (String[,]) o2dim;  // compiles but throw exception at runtime

// Create a one-dimensional Int32 array (value types).
Int32[] i1dim = new Int32[5]; 

// Can't cast from array of value types to anything else
Object[] o1dim = (Object[]) i1dim;   // Compiler error CS0030: Cannot convert type 'int[]' to 'object[]'

// Create a new array, then use Array.Copy to coerce each element in the
// source array to the desired type in the destination array.
// The following code creates an array of references to boxed Int32s.
Object[] ob1dim = new Object[i1dim.Length]; 
Array.Copy(i1dim, ob1dim, i1dim.Length);
```
The `Array.Copy` method is not just a method that copies elements from one array to another. The Copy method handles overlapping regions of memory correctly, as does C's memmove function. C's memcpy function, on the other hand, doesn't handle overlapping regions correctly. 

**Background Knowledge**: What's memory overlapping?
```
+++++++++++++++++++++++++++++++
| 'a' | 'b' | 'c' | 'd' | 'e' |
+++++++++++++++++++++++++++++++
 0x100 0x101 0x102 0x103 0x104
```
`memcpy` takes 3 arguments, a pointer to the destination block of memory, a pointer to the source block of memory, and the size of bytes to be copied. what if the destination is 0x102, the source is 0x100 and the size is 3, memory overlapping happens here. that is, 0x100 would be copied into 0x102, 0x101 would be copied into 0x103 and 0x102 would be copied into 0x104. 

Notice that we first copied into 0x102 then we copied from 0x102 which means that the value which was originally in 0x102 was lost as we overwrote it with the value we copied into 0x102 before we copy from it. so we would end up with something like:
```
+++++++++++++++++++++++++++++++
| 'a' | 'b' | 'a' | 'b' | 'a' |
+++++++++++++++++++++++++++++++
 0x100 0x101 0x102 0x103 0x104
```
instead of 
```
+++++++++++++++++++++++++++++++
| 'a' | 'b' | 'a' | 'b' | 'c' |
+++++++++++++++++++++++++++++++
 0x100 0x101 0x102 0x103 0x104
```
`memmove` take care of memory overlapping by coping the bytes to be copied into a temporary array then pastes them into the destination block as oppose to a function like `memcpy` which copies directly from the source block to the destination block.

Come back to the Array.Copy topic, The Array.Copy mehtod can also convert each array elements as it is copied of conversion is required. The Copy method is capable of performing the following conbersions:
<ul>
  <li>Boxing value type elements to reference type elements, such as coping an Int32[] to an Object[]</li>
  <li>Unboxing reference type elements to value type elements, such as coping an Object[] to an Int32[]</li>
  <li>Widening CLR primitive value types, such as coping elements from an Int32[] to a Double[]</li>
  <li>Downcasting elements when copying between array types that can't be proven to be compatible based on the array's type, such as when casting from an Object[] to an IFormattable[]. If every object in the Objec[] implements IFormattabke, Copy will succeed, if not, runtime ArrayTypeMismatchException will be thrown</li>
</ul> 

Here's another example showing the usefulness of Copy:
```C#
// Define a value type that implements an interface
internal struct MyValueType : IComparable {
   public Int32 CompareTo(Object obj) {
      ...
   }   
}

public static void Main() {
   // Create an array of 100 value types.
   MyValueType[] src = new MyValueType[100];

   // Create an array of IComparable references
   IComparable[] dest = new IComparable[src.Length]; 

   // Initialize an array of IComparable elements to refer to boxed 
   // versions of elements in the source array.
   Array.Copy(src, dest, src.Length);  
}
```
As you might imagine, the FCL takes advantages of Array's Copy method quite frequently.

In some situations, it is useful to cast an array from one type to another. This kind of functionality is called ***array covariance***. When you take advantage of array covariance, you should be aware of an associated performance penalty. Let's say you have the following code:
```C#
String[] sa = new String[100]; 
Object[] oa = sa;  // oa refers to an array of String elements 
oa[5] = "Jeff";    // Perf hit: CLR checks oa's element type for String; OK
oa[3] = 5;         // Perf hit: CLR checks oa's element type for Int32; throws ArrayTypeMismatchException
```
In the preceding code, the oa variable is typed as an Object[]; however, it really refers to a String[]. The compiler will allow you to write code that attempts to put 5 ito an array element because 5 is an Int32, which is derived from Object. Of course, the CLR must ensure type safety, and when assigning to an array element, the CLR myst  ensure that the assignment is legal. So the CLR must check at run time whether the array can contain Int32 elements. In this case, it doesn't , and the assignment cannot be allowed; The CLR will throw an ArrayTypeMismatchException.

<div class="alert alert-info p-1" role="alert">
    To work in the future : Buffer's BlockCopy method and Array's ConstrainedCopy method.
</div>

## All Arrays Are Implicitly Derived from System.Array

When you declare an array variable like this:
```C#
FileStream[] fsArray;
```
then the CLR automatically creates a FileStream[] type for the AppDomain. This type will be implicitly derived from the System.Array type, and therefore, all of the instance methods and properties defined on the System.Array type will be ingerited by the FileStream[] type, allowing these methods and properties to be called using the fsArray variable. This makes working with arrays extremely convenient because there are many helpful instance methods and properties defined by System.Array such as Clone, CopyTo, GetLength, GetLongLength, GetLowerBound, GetUpperBound, Length, Rank, and others.

The System.Array type also exposes a large number of extremely useful static methods that operate on arrays. These methods all take a reference to an array as a parameter. Some of the useful static methods are AsReadOnly, BinarySearch, Clear, ConstrainedCopy, ConvertAll, Copy, Exists, Find, FindAll, FindIndex, FindLast, FindLastIndex, ForEach, IndexOf, LastIndexOf, Resize, Reverse, Sort, and TrueForAll.

## All Arrays Implicitly Implement IEnumerable, ICollection, and IList

There are many methods that operate on various collection objects because the methods are declared with parameters such as `IEnumerable`, `ICollection`, and `IList`. It is possible to pass arrays to these methods because System.Array also implements these three interfaces. System.Array implements these non-generic interfaces because they treat all elements as System.Object. However, it would be nice to have System.Array implement the generic equivalent of these interfaces, providing better compile-time type safety as well as better performance.

The CLR team didn't want System.Array to implement `IEnumerable<T>`, `ICollection<T>`, and `IList<T>`, though, because of issues related to multi-dimensional arrays and non-zero-based arrays. Defining these interfaces on System.Array would have enabled these interfaces for all array types. Instead, the CLR performs a little trick: when a single-dimensional, zero-lower bound array type is created, the CLR automatically makes the array type implement `IEnumerable<T>`, `ICollection<T>`, and `IList<T>` where T is the array's element type and also implements these three interfaces for all of the array type's base types as long as they are reference types. 
So, for example, if you have the following line of code:
```C#
FileStream[] fsArray;
```
then when the CLR creates the FileStream[] type, it will cause this type to automatically implement the `IEnumerable<FileStream>`, `ICollection<FileStream>`, and `IList<FileStream>`. Furthermore, the FileStream[] type will also implement the interfaces for the base types: `IEnumerable<Stream>`, `IEnumerable<Object>`, `ICollection<Stream>`, `ICollection<Object>`, `IList<Stream>`, and `IList<Object>`. Because all of these interfaces are automatically implemented by the CLR, the fsArray variable could be used wherever any of these interfaces exist. For example, the fsArray variable could be passed to methods that have any of the following prototypes:
```C#
void M1(IList<FileStream> fsList) { … }
void M2(ICollection<Stream> sCollection) { … }
void M3(IEnumerable<Object> oEnumerable) { … }
```
Note that if the array contains value type elements, the array type will not implement the interfaces for the element's base types. For example, if you have the following line of code:
```C#
DateTime[] dtArray; // An array of value types
```
then the DateTime[] type will implement `IEnumerable<DateTime>`, `ICollection<DateTime>`, and `IList<DateTime>` only; it will not implement versions of these interfaces that are generic over System.ValueType or System.Object. This means that the dtArray variable cannot be passed as an argument to the M3 method shown earlier. The reason for this is because arrays of value types are laid out in memory differently than arrays of reference types.

## Passing and Returning Arrays

When passing an array as an argument to a method, you are really passing a reference to that array. Therefore, the called method is able to modify the elements in the array. If you don't want to allow this, you must make a copy of the array and pass the copy into the method. Note that the Array.Copy method performs a shallow copy, and therefore, if the array's elements are reference types, the new array refers to the already existing objects.

Similarly, some methods return a reference to an array. If the method constructs and initializes the array, returning a reference to the array is fine. But if the method wants to return a reference to an internal array maintained by a field, you must decide if you want the method’s caller to have direct
access to this array and its elements. If you do, just return the array’s reference. But most often, you won't want the method’s caller to have such access, so the method should construct a new array and call Array.Copy, returning a reference to the new array. Again, be aware that Array.Copy makes a shallow copy of the original array.

If you define a method that returns a reference to an array, and if that array has no elements in it, your method can return either null or a reference to an array with zero elements in it. When you're implementing this kind of method, Microsoft strongly recommends that you implement the method by having it return a zero-length array because doing so simplifies the code that a developer calling the method must write. For example, this easy-to-understand code run correctly even if there are no elements to interate over:
```C#
// This code is easier to write and understand.
Appointment[] appointments = GetAppointmentsForToday();
for (Int32 a = 0; a < appointments.Length; a++) {
   ...
}
```
instead of
```C#
// This code is harder to write and understand.
Appointment[] appointments = GetAppointmentsForToday();
if (appointments != null) {
   for (Int32 a = 0, a < appointments.Length; a++) {
      ...
   }
}
```
By the way, you should do the same for fields. If your type has a field that’s a reference to an array, you should consider having the field refer to an
array even if the array has no elements in it.

## Creating Non-Zero Lower Bound Arrays

It's possible to create and work with arrays that have non-zero lower bounds. You can dynamically create your own arrays by calling Array's static `CreateInstance` method. Serveral overloads of this method exist, allowing you to specify the type of the elements in the array, the number of dimensions in the array, the lower bounds of each dimension, and the number of elements in each dimension. `CreateInstance` allocates memory for the array, saves the parameter information in the overhead portion of the array's memory block, and returns a reference to the array. If the array has two or more dimensions, you can cast the reference returned from CreateInstance to an `ElementType[,...,]` variable (where ElementType is some type name), making it easier for you to access the elements in the array. If the array has just one dimension, in C#, you have to use Array's GetValue and SetValue methods to access the element of the array.

Here's some code that demonstrates how to dynamically create a two-dimensional array of System.Decimal values. The first dimension represents calendar years from 2005 to 2009 inclusive, and the second dimension represents quarters from 1 to 4 inclusive. The code iterates over all the elements in the dynamic array. I could have hard-coded the array’s bounds into the code, which would have given better performance, but I decided to use System.Array's GetLowerBound and
GetUpperBound methods to demonstrate their use:
```C#
static void Main(string[] args) {
    // I want a two-dimensional array [2005..2009][1..4].
    Int32[] lowerBounds = { 2005, 1 };
    Int32[] lengths = { 5, 4 };
    Decimal[,] quarterlyRevenue = (Decimal[,])Array.CreateInstance(typeof(Decimal), lengths, lowerBounds);

    Console.WriteLine("{0,4} {1,9} {2,9} {3,9} {4,9}", "Year", "Q1", "Q2", "Q3", "Q4");

    Int32 firstYear = quarterlyRevenue.GetLowerBound(0);  // 0 means the first dimension
    Int32 lastYear = quarterlyRevenue.GetUpperBound(0);
    Int32 firstQuarter = quarterlyRevenue.GetLowerBound(1);
    Int32 lastQuarter = quarterlyRevenue.GetUpperBound(1);

    for (Int32 year = firstYear; year <= lastYear; year++) {
        Console.Write(year + " ");
        for (Int32 quarter = firstQuarter; quarter <= lastQuarter; quarter++) {
            Console.Write("{0,9:C} ", quarterlyRevenue[year, quarter]);
        }
        Console.WriteLine();
    }
}
```
If you compile and run this code, you get the following output:
```
Year        Q1        Q2        Q3        Q4
2005     $0.00     $0.00     $0.00     $0.00
2006     $0.00     $0.00     $0.00     $0.00
2007     $0.00     $0.00     $0.00     $0.00
2008     $0.00     $0.00     $0.00     $0.00
2009     $0.00     $0.00     $0.00     $0.00
```

By the way, if you use CreateInstance to create one-dimension array, you cannot cast it to `ElementType[]`:
```C#
Int32[] lowerBounds = { 2021 };
Int32[] lengths = { 4 };

//code below does compile but throw an InvalidCastException: "Unable to cast object of type `System.Int32[*]` to `System.Int32[]`
// In the next section, you will see that [*] means the array is not zero-based, that's why the casting fail at runtime
// If you change the lowbound from 2021 to 0, the code run successfully
Int32[] quarterlyRevenue = (Int32[]) Array.CreateInstance(typeof(Int32), lengths, lowerBounds);  //throws exception at runtime
```
so you have to do:
```C#
Array quarterlyRevenue = Array.CreateInstance(typeof(Int32), lengths, lowerBounds);
var item = (Int32)quarterlyRevenue.GetValue(2021);
quarterlyRevenue.SetValue(item,2021);   
```

## Array Internals

Internally, the CLR actually supports two different kinds of arrays:

<ul>
  <li>Single-dimensional arrays with a lower bound of 0. These arrays are sometimes called SZ (for single-dimensional, zero-based) arrays or vectors.</li>
  <li>Single-dimensional and multi-dimensional arrays with an unknown lower bound.</li>
</ul> 

You can actually see the different kinds of arrays by executing the following code:
```C#
static void Main(string[] args) {
    Array a;

    // Create a 1-dim, 0-based array, with no elements in it
    a = new String[0];
    Console.WriteLine(a.GetType());    // "System.String[]"

    a = Array.CreateInstance(typeof(String), new Int32[] { 0 }, new Int32[] { 0 });
    Console.WriteLine(a.GetType());    // "System.String[]"

    // Create a 1-dim, 1-based array, with no elements in it
    a = Array.CreateInstance(typeof(String), new Int32[] { 0 }, new Int32[] { 1 });
    Console.WriteLine(a.GetType());    // "System.String[*]" <-- INTERESTING!

    a = new String[0];
    Console.WriteLine(a.GetType());    // "System.String[,]"

    // Create a 2-dim, 0-based array, with no elements in it
    a = Array.CreateInstance(typeof(String), new Int32[] { 0, 0 }, new Int32[] { 0, 0 });
    Console.WriteLine(a.GetType());    // "System.String[,]"

    a = Array.CreateInstance(typeof(String), new Int32[] { 0, 0 }, new Int32[] { 1, 1 });
    Console.WriteLine(a.GetType());   // "System.String[,]"
}
```
Next to each Console.WriteLine is a comment that indicates the output. For the singledimensional arrays, the zero-based arrays display a type name of System.String[], whereas the 1-based array displays a type name of `System.String[*]`. The * indicates that the CLR knows that this array is not zero-based. **Note that C# does not allow you to declare a variable of type `String[*]`**, and therefore it is not possible to use C# syntax to access a single-dimensional,
non-zero–based array. Although you can call Array's `GetValue` and `SetValue` methods to access the elements of the array, this access will be slow due to the overhead of the method call.

For multi-dimensional arrays, the zero-based and 1-based arrays all display the same type name: `System.String[,]`. The CLR treats all multi-dimensional arrays as though they are not zero-based at run time. This would make you think that the type name should display as `System.String[*,*]`; however, the CLR doesn't use the *s for multi-dimensional arrays because they would always be present, and the asterisks would just confuse most developers.

Accessing the elements of a single-dimensional, zero-based array is slightly faster than accessing the elements of a non-zero–based, single-dimensional array or a multi-dimensional array. There are several reasons for this. First, there are specific IL instructions to manipulate single-dimensional, zero-based arrays, and these special IL instructions cause the JIT compiler to emit optimized code. For example, the JIT compiler will emit code that assumes that the array is zero-based, and this means that an offset doesn't have to be subtracted from the specified index when accessing an element. Second, in common situations, the JIT compiler is able to hoist the index range–checking code out of the loop, causing it to execute just once. For example, look at the following commonly written code:
```C#
public static void Main() {
   Int32[] a = new Int32[5];
   for(Int32 index = 0; index < a.Length; index++) {
      // Do something with a[index]
   }   
}
```
The first thing to notice about this code is the call to the array's Length property in the for loop's test expression. Because Length is a property, querying the length actually represents a method call. However, the JIT compiler know that Length is a property on the Array class, and the JIT compiler will actually generate code that calls the property just once and stores the result in a temporary variable that will be checked with each iteration of the loop. The result is that the JITed code is fast. In fact, some developers have understimated the abilities of the JIT cimpiler and have tried to write "clever code" in an attempt to help the JIT cimpiler. However, any clever attemps that you come up with all almost certainly impact performance negatively and make your code harder to read. You are better off leaving the call to the array's Length property in the preceding code instead of attempting to cache it in a local variable yourself.

The second thing to notice about the preceding code is that the JIT compiler knows that the for loop is accessing array elements 0 through Length - 1. SO the JIT compiler produces code that, at run time, tests that all array access will be within the array's valid range. Specifically, the JIT compiler produces code to check if `0 >= a.GetLowerBOund(0) && (Length -1) <= a.GetUpperBound(0)` (hoist the index range–checking code out of the loop). This check occurs just before the loop. If the check is good, the JIT compiler will not generate code inside the loop to verify that each array access is within the valid range. This allows array access within the loop to be very fast.

Unfortunately, as I alluded to earlier in this chapter, accessing elements of a non-zero–based single-dimensional array or of a multi-dimensional array is much slower than a single-dimensional, zerobased array. For these array types, the JIT compiler doesn't hoist index checking outside of loops, so each array access validates the specified indexes. In addition, the JIT compiler adds code to subtract the array's lower bounds from the specified index, which also slows the code down, even if you're using a multi-dimensional array that happens to be zero-based. So if performance is a concern to you, you might want to consider using an array of arrays (a jagged array) instead of a rectangular array. This is because a jagged array is an array of arrays, so `int[][]` is actually an array of `int[]`, each of which can be of different lengths and occupy their own block in memory. A multidimensional array (`int[,]`) is a single block of memory (essentially a matrix), CLR is heavily optimized for single-dimension array access, so using a jagged array will likely be faster than a multidimensional array of the same size.

C# and the CLR also allow you to access an array by using unsafe (non-verifiable) code, which is, in effect, a technique that allows you to turn off the index bounds checking when accessing an array. Note that this unsafe array manipulation technique is usable with arrays whose elements are SByte,
Byte, Int16, UInt16, Int32, UInt32, Int64, UInt64, Char, Single, Double, Decimal, Boolean, an enumerated type, or a value type structure whose fields are any of the aforementioned types. This is a very powerful feature that should be used with extreme caution because it allows you to perform direct memory accesses. If these memory accesses are outside the bounds of the array, an exception will not be thrown; instead, you will be corrupting memory, violating type safety, and possibly opening a security hole

The following C# code demonstrates three techniques (safe, jagged, and unsafe), for accessing a two-dimensional array:
```C#
static void Main(string[] args) {
    // Declare a two-dimensional array
    Int32[,] a2Dim = new Int32[c_numElements, c_numElements];

    // Declare a two-dimensional array as a jagged array (a vector of vectors)
    Int32[][] aJagged = new Int32[c_numElements][];
    for (Int32 x = 0; x < c_numElements; x++)
        aJagged[x] = new Int32[c_numElements];

    // 1: Access all elements of the array using the usual, safe technique
    Safe2DimArrayAccess(a2Dim);

    // 2: Access all elements of the array using the jagged array technique
    SafeJaggedArrayAccess(aJagged);
}

private static Int32 Safe2DimArrayAccess(Int32[,] a) {
    Int32 sum = 0;
    for (Int32 x = 0; x < c_numElements; x++) {
        for (Int32 y = 0; y < c_numElements; y++) {
            sum += a[x, y];
        }
    }
    return sum;
}

private static Int32 SafeJaggedArrayAccess(Int32[][] a) {
    Int32 sum = 0;
    for (Int32 x = 0; x < c_numElements; x++) {
        for (Int32 y = 0; y < c_numElements; y++) {
            sum += a[x][y];
        }
    }
    return sum;
}

private static unsafe Int32 Unsafe2DimArrayAccess(Int32[,] a) {
    Int32 sum = 0;
    fixed (Int32* pi = a) {  // fixed keyword is related to Ch21-Garbage Collection
        for (Int32 x = 0; x < c_numElements; x++) {
            Int32 baseOfDim = x * c_numElements;
            for (Int32 y = 0; y < c_numElements; y++) {
                sum += pi[baseOfDim + y];
            }
        }
    }
    return sum;
}
```
The Unsafe2DimArrayAccess method is marked with the unsafe modifier, which is required to use C#'s fixed statement. To compile this code, you’ll have to specify the /unsafe switch when invoking the C# compiler or select the Allow Unsafe Code check box on the Build tab of the Project Properties pane in Microsoft Visual Studio.

## Unsafe Array Access and Fixed-Size Array
To work on in the future as allocating array on stack using `stackalloc` is not common.

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