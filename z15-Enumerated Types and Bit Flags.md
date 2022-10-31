Chapter 15-Enumerated Types and Bit Flags
==============================

## Enumerated Types

An *enumerated* type is a type that defines a set of symbolic name and value pairs. For example, the Color type shown below defines a set of symbols, with each symbol identifying a single color:
```C#
internal enum Color {
    White,
    Red,
    Green,
    Blue,
    Orange
}
```
Of course, programmers can always write a program using 0 to represent white, 1 to represent red, and so on. However, programmers shouldn’t hard-code numbers into their code and should use an enumerated type instead, for at least two reasons:
<ul>
  <li><b>Enumerated types make the program much easier to write, read, and maintain.</b> With enumerated types, the symbolic name is used throughout the code, and the programmer doesn't have to mentally map the meaning of each hard-coded value (for example, white is 0 or vice versa). Also, should a symbol's numeric value change, the code can simply be recompiled without requiring any changes to the source code. </li>
  <li><b>Enumerated types are strongly typed.</b> For example, the compiler will report an error if I attempt to pass Color.Orange as a value to a method requiring a Fruit enumerated type as a parameter.</li>
</ul> 

In the Microsoft .NET Framework, enumerated types are more than just symbols that the compiler cares about. Enumerated types are treated as first-class citizens in the type system, which allows for very powerful operations that simply can't be done with enumerated types in other environments (such as in unmanaged C++, for example).

Every enumerated type is derived directly from `System.Enum`, which is derived from `System.ValueType`. So enumerated types are value types and can be represented in unboxed and boxed form. However, unlike other value types, an enumerated type can't define any methods, properties, or events. However, you can use C#’s extension methods feature to simulate adding methods to an enumerated type.

When an enumerated type is compiled, the C# compiler turns each symbol into a constant field of the type. For example, the compiler treats the Color enumeration shown earlier as if you had written code similar to the following:
```C#
public abstract class Enum : ValueType, IComparable, IFormattable, IConvertible {
   protected Enum();
   
   public static string Format(Type enumType, object value, string format);

   public static string[] GetNames(Type enumType);
   public static Type GetUnderlyingType(Type enumType);
   public static Array GetValues(Type enumType);

   public static bool IsDefined(Type enumType, object value);

   public static object Parse(Type enumType, string value);
   public static object Parse(Type enumType, string value, bool ignoreCase);
   public static bool TryParse<TEnum>(string value, out TEnum result) where TEnum : struct;
   public static bool TryParse<TEnum>(string value, bool ignoreCase, out TEnum result) where TEnum : struct;

   public static object ToObject(Type enumType, object value);
   public static object ToObject(Type enumType, int value);
   public static object ToObject(Type enumType, byte value);
   ...
   public static object ToObject(Type enumType, long value);
   ...
   public override bool Equals(object obj);
   public override string ToString();
   public string ToString(string format);
   public string ToString(string format, IFormatProvider provider);
}

internal struct Color : System.Enum {
   // Following are public constants defining Color's symbols and values
   public const Color White = (Color) 0;
   public const Color Red = (Color) 1;
   public const Color Green = (Color) 2;
   public const Color Blue = (Color) 3;
   public const Color Orange = (Color) 4; 

   // Following is a public instance field containing a Color variable's value
   // You cannot write code that references this instance field directly 
   public Int32 value__;  
}
```
<div class="alert alert-info p-1" role="alert">
   You might be wondering why you can cast a Int32 to Color, my personal guess is that when perform such a casting, C# compiler internally assign the Int32 value to the Color's instance field <code>value__</code>, that's probably why you can do the following:
</div>

```C#
Color c = (Color) 1;   // value__ is assigned with 1, c is red now
Color c = (Color) 55;  // value__ is assigned with 1, c is 55, don't make sense but it is valid
```
The C# compiler won't actually compile this code because it forbids you from defining a type derived from the special System.Enum type. However, this pseudo-type definition shows you what's happening internally. Basically, an enumerated type is just a structure with a bunch of constant fields
defined in it and one instance field. The constant fields are emitted to the assembly's metadata and can be accessed via reflection. This means that you can get all of the symbols and their values assocaited with an enumerated type at run time. It also means that you can convert a string symbol into its equivalent numeric value. These operations are made available to you by the System.Enum base type, which offers several static and instance methods that can be performed on an instance of an enumerated type, saving you the trouble of having to use reflection. I'll discuss some of these operations next.

<div class="alert alert-info p-1" role="alert">
   Symbols defined by an enumerated type are constant values. So when a compiler sees code that references an enumerated type's symbol, the compiler substitutes the symbol's numeric value at compile time, and this code no longer references the enumerated type that defined the symbol. This means that the assembly that defines the enumerated type may not be required at run time; it was required only when compiling. Some versioning issues arise because enumerated type symbols are constants instead of read-only values. I explained these issues in the "Constants" section of Chapter 7, "Constants and Fields. "
</div>

Every enumerated type has an underlying type, which can be a byte, sbyte, short, ushort, int (the most common type and what C# chooses by default), uint, long, or ulong. Of course, these C# primitive types correspond to FCL types. The following code shows how to declare an enumerated type with an underlying type of byte (System.Byte):
```C#
internal enum Color : byte {
    White,
    Red,
    Green,
    Blue,
    Orange
}
```
`System.Enum` type has a static method called `GetUnderlyingType`, and the `System.Type` type has an instance method called `GetEnumUnderlyingType`:
```C#
public static Type GetUnderlyingType(Type enumType);  // Defined in System.Enum
public Type GetEnumUnderlyingType();                  // Defined in System.Type
```
These methods return the core type used to hold an enumerated type's value:
```C#
// The following line displays "System.Byte".
Console.WriteLine(Enum.GetUnderlyingType(typeof(Color)));
```

The C# compiler treats enumerated types as primitive types. As such, you can use many of the familiar operators such as ==, !=, <, >, <=, >=, +, -, ^, &, |, ~, ++, and - to manipulate enumerasted type instance. All of these operators actually work on the `value__ ` instance field inside each enumerated type instance. So you have two ways to compare whether two enumerated instance are equal:
```C#
Color myfavColor = Color.Blue;
Color youfavColor = Color.Red;

if (myfavColor.Equals(youfavColor)) {
   ...
}

if(myfavColor == youfavColor) {
   ...
}
```
Be careful of the `Equals` method which is from Enum base class's `public override bool Equals(object obj);`, if you use Equals method, num value passed in would need to be boxed & unboxed., so == is recommended. But use Equals gives you more flexibility as you can pass anything in it and the method will return false if you do sth like `myfavColor.Equals("what")` , if you use ==, both types on the operends have to be the same type otherwise there will be a compile error.

Furthermore, the C# compiler allows you to explicitly cast instances of an enumerated type to a different enumerated type. You can also explicitly cast an enumerated type instance to a numeric type.

Given an instance of an enumerated type, it's possible to map that value to one of several string representations by calling the ToString method inherited from System.Enum:
```C#
Color c = Color.Blue;
Console.WriteLine(c);               // "Blue" (General format)
Console.WriteLine(c.ToString());    // "Blue" (General format)
Console.WriteLine(c.ToString("G")); // "Blue" (General format)
Console.WriteLine(c.ToString("D")); // "3" (Decimal format)
Console.WriteLine(c.ToString("X")); // "03" (Hex format)
```

In addition to the ToString method, the System.Enum type offers a static Format method that you can call to format an enumerated type's value:
```C#
public static String Format(Type enumType, Object value, String format);
```
Generally, I prefer to call the ToString method because it requires less code and it’s easier to call. But using Format has one advantage over ToString: Format lets you pass a numeric value for the value parameter; you don’t have to have an instance of the enumerated type. For example, the following code will display "Blue":
```C#
// The following line displays "Blue".
Console.WriteLine(Enum.Format(typeof(Color), (Byte)3, "G"));    // Becareful that 3 has to be casted to byte as the underlying type of the enum is byte
                                                                // if you don't cast it to byte, exception throws
```

<div class="alert alert-info p-1" role="alert">
    It's possible to declare an enumerated type that has multiple symbols, all with the same numeric value. When converting a numeric value to a symbol by using general formatting, Enum’s methods return one of the symbols. However, there’s no guarantee of which symbol name is returned. Also, if no symbol is defined for the numeric value you're looking up, a string containing the numeric value is returned.
</div>

It's also possible to call System.Enum's static `GetValues` method or System.Type's instance `GetEnumValues` method to obtain an array that contains elements  an enumerated type:
```C#
public static Array GetValues(Type enumType); // Defined in System.Enum
public Array GetEnumValues();                 // Defined in System.Type
```
Using this method along with the `IFormattable.ToString` method(check Chapter 14-String for details), you can display all of an enumerated type's symbolic and numeric values, like the following:
```C#
Color[] colors = (Color[]) Enum.GetValues(typeof(Color));
Console.WriteLine("Number of symbols defined: " + colors.Length);
Console.WriteLine("Value\tSymbol\n-----\t------");
foreach (Color c in colors) {
   // Display each symbol in Decimal and General format.
   Console.WriteLine("{0,5:D}\t{0:G}", c);
}
```
The previous code produces the following output:
```
Number of symbols defined: 5
Value Symbol
-----    ----- 
    0    White
    1    Red
    2    Green
    3    Blue
    4    Orange
```

I suspect that the ToString method with the general format will be used quite frequently to show symbolic names in a program's user interface elements (list boxes, combo boxes, and the like), as long as the strings don't need to be localized (**because enumerated types offer no support for localization**).

Converting a symbol to an instance of an enumerated type is easily accomplished by using one of Enum's static `Parse` and `TryParse` methods:
```C#
// Because Orange is defined as 4, 'c' is initialized to 4.
Color c = (Color) Enum.Parse(typeof(Color), "orange", true);

// Because Brown isn't defined, an ArgumentException is thrown.
c = (Color) Enum.Parse(typeof(Color), "Brown", false); 

// Creates an instance of the Color enum with a value of 1 - Red  
//the first argument can be string representation of the enumeration name or underlying value to convert
Enum.TryParse("1", false, out c);
Enum.TryParse("Red", false, out c);

// Creates an instance of the Color enum with a non-exist string 
Enum.TryParse("Purple", false, out c);      // c is "White" as "White" considers to be the default value which is the first one in the enum type
Enum.TryParse("23", false, out c);          // c is "23", not "White", the method returns true, a litte werid ...
```

Finally, using the following Enum's static `IsDefined` method and Type's `IsEnumDefined`method:
```C#
public static Boolean IsDefined(Type enumType, Object value);  // Defined in System.Enum
public Boolean IsEnumDefined(Object value);                    // Defined in System.Type
```
you can determine whether a numeric value is legal for an enumerated type:
```C#
// Displays "True" because Color defines Red as 1
Console.WriteLine(Enum.IsDefined(typeof(Color), (Byte)1));

// Displays "True" because Color defines White as 0
Console.WriteLine(Enum.IsDefined(typeof(Color), "White"));

// Displays "False" because a case-sensitive check is performed
Console.WriteLine(Enum.IsDefined(typeof(Color), "white")); 

// Displays "False" because Color doesn't have a symbol of value 10
Console.WriteLine(Enum.IsDefined(typeof(Color), (Byte)10));

// Displays "False" because Color doesn't have a symbol of value 10
Color c = (Color)10;
Console.WriteLine(Enum.IsDefined(typeof(Color), c);
```
The static `IsDefined` method is frequently used for parameter validation. Here is an example:
```C#
public void SetColor(Color c) {
   if (!Enum.IsDefined(typeof(Color), c)) {
      throw(new ArgumentOutOfRangeException("c", c, "Invalid Color value."));   
   }
}
```
The parameter validation is useful because someone could call SetColor like the following:
```C#
SetColor((Color) 547);
```
Because no symbol has a corresponding value of 547, the SetColor method will throw an ArgumentOutOfRangeException exception, indicating which parameter is invalid and why.

<div class="alert alert-info p-1" role="alert">
   The IsDefined mehtod is very convenient, but you must use it with caution. First, IsDefined always does a case-sensitive search, and there is no way to get it to perform a case-insensitive search. Second, IsDefined is pretty slow because it uses reflection internally;; if you wrote code to manually check each possible value, your application's performance would most certainly be better.
</div>

Finally, the System.Enum type offers a set of static ToObject methods that convert an instance of a Byte, SByte, Int16, UInt16, Int32, UInt32, Int64, or UInt64 to an instance of an enumerated type:
```C#
Color c = (Color)Enum.ToObject(typeof(Color), 1);

//Note that the conversion succeeds even if value is outside the bounds of enumType members. 
//To ensure that value is a valid underlying value of the enumType enumeration, pass it to the IsDefined method.
Color c = (Color)Enum.ToObject(typeof(Color), 11);
if (Enum.IsDefined(typeof(Color), c) {
   ...
}
```


## Bit Flags

If you eant an enumeration type to represent a combination of choices, define enum members for those choices such that an individual choice is a bit field. That is, the associated values of those enum members should be the powers of two. Then, you can use the bitwise logical operators | or & to combine choices or intersect combinations of choices, respectively. To indicate that an enumeration type declares bit fields, apply the `Flags` attribute to it. As the following example shows, you can also include some typical combinations in the definition of an enumeration type:
```C#
[Flags]
public enum Days
{
    None      = 0b_0000_0000,  // 0
    Monday    = 0b_0000_0001,  // 1
    Tuesday   = 0b_0000_0010,  // 2
    Wednesday = 0b_0000_0100,  // 4
    Thursday  = 0b_0000_1000,  // 8
    Friday    = 0b_0001_0000,  // 16
    Saturday  = 0b_0010_0000,  // 32
    Sunday    = 0b_0100_0000,  // 64
    Weekend   = Saturday | Sunday
}

public static void Main() {
   Days meetingDays = Days.Monday | Days.Wednesday | Days.Friday;
   Console.WriteLine(meetingDays);
   // Output: Monday, Wednesday, Friday

   Days workingFromHomeDays = Days.Thursday | Days.Friday;
   Console.WriteLine($"Join a meeting by phone on {meetingDays & workingFromHomeDays}");
   // Output: Join a meeting by phone on Friday

   bool isMeetingOnTuesday = (meetingDays & Days.Tuesday) == Days.Tuesday;
   Console.WriteLine($"Is there a meeting on Tuesday: {isMeetingOnTuesday}");
   // Output: Is there a meeting on Tuesday: False

   var a = (Days)37;
   Console.WriteLine(a);
   // Output: Monday, Wednesday, Saturday
}
```

When defining an enumerated type thath is to be used to identify bit flags, you should, of course, explicitly assign a numeric value to each symbol. Usually, each symbol will have an individual bit turned on. It is also common to see a symbol called `None` defined with a value of 0,  and you can also define symbols that represent commonly used combinations (see the above Weekend symbol). It's also highly recommended that you apply the System.FlagsAttribute custom attribute type to the enumerated type, as shown here:
```C#
[Flags] // The C# compiler allows either "Flags" or "FlagsAttribute"
internal enum Actions {
   None = 0,
   Read = 0x0001,
   Write = 0x0002,
   ReadWrite = Actions.Read | Actions.Write, 
   Delete = 0x0004,
   Query = 0x0008, 
   Sync = 0x0010 
}
```
Because Actions is an enumerated type, you can use all of the methods described in the previous section when working with bit-flag enumerated types. However, it would be nice if some of those functions behaved a little differently. For example, let’s say you had the following code:
```C#
Actions actions = Actions.Read | Actions.Delete;  // 0x0005
Console.WriteLine(actions.ToString());            // "Read, Delete"
```
When ToString is called, it attempts to translate the numeric value into its symbolic equivalent. The numeric value is 0x0005, which has no symbolic equivalent. However, the ToString method detects the existence of the [Flags] attribute on the Actions type, and ToString now treats the numeric value not as a single value but as a set of bit flags. Because the 0x0001 and 0x0004 bits are set, ToString generates the following string: "Read, Delete". If you remove the [Flags] attribute from the Actions type, ToString would return "5".

I discussed the ToString method in the previous section, and I showed that it offered three ways to format the output: "G" (General), "D" (decimal), and "X" (hex). When you're formatting an instance of an enmuerated type by using the general format, the type is first checked to see if the [Flags] attribute is applied to it. If this attribute is not applied, a symbol matching the numeric value is looked up and returned. If the [Flags] attribute is applied, ToString works like this:

<ol>
  <li>The set of numeric values defined by the enumerated type is obtained, and the numbers are sorted in <b>descending</b> order.</li>
  <li>Each numeric value is bitwise-ANDed with the value in the enum instance, and if the result equals the numeric value, the string associated with the numeric value is appended to the output string, and the bits are considered accounted for and are turned off. This step is repeated until all numeric values have been checked or until the enum instance has all of its bits turned off./li>
  <li>If, after all the numeric values have been checked, the enum instance is still not 0, the enum instance has some bits turned on that do not correspond to any defined symbols. In this case, ToString returns the original number in the enum instance as a string</li>
  <li>If the enum instance’s original value wasn't 0(and the enum instance is 0 after all its bits turned off), the string with the comma-separated set of symbols is returned.</li>
  <li>If the enum instance’s original value was 0 and if the enumerated type has a symbol defined with a corresponding value of 0, the symbol (usually defined as "None") is returned.</li>
</ol> 

Note that the symbols you define in your enumerated type don’t have to be pure powers of 2.  For example, the Actions type could define a symbol called All with a value of 0x001F (the sum of all enum value). If an instance of the Actions type has a value of 0x001F, formatting the instance will produce a string that contains "All". The other symbol strings won’t appear. (Because of the step one above, the numbers are sorted in descending order)

It's also possible to convert a string of comma-delimited symbols into a numeric value by calling Enum's static Parse and TryParse method. Here's some code demonstrating how to use this method.
```C#
// Because Query is defined as 8, 'a' is initialized to 8.
Actions a = (Actions) Enum.Parse(typeof(Actions), "Query", true);
Console.WriteLine(a.ToString()); // "Query"

// Creates an instance of the Actions enum with a value of 28.
a = (Actions) Enum.Parse(typeof(Actions), "28", false);
Console.WriteLine(a.ToString()); // "Delete, Query, Sync"

// Because Query and Read are defined, 'a' is initialized to 9.
Enum.TryParse<Actions>("Query, Read", false, out a);
Console.WriteLine(a.ToString()); // "Read, Query"
```

You should never use the Enum's static `IsDefined` method with bit flag-enumerated types. It won't work for two reasons:
<ul>
  <li>If you pass a string to IsDefined, it doesn't split the string into separate tokens to look up; it will attempt to look up the string as through it were one big symbol with commas in it. Because you can't define an enum with a symbol that has commas in it, the symbol will never be found.</li>
  <li>If you pass a numeric value to IsDefined, it checks whether the enumerated type defines a single symbol whose numeric value matches the passed-in number. Because this is unlikely for bit flags, IsDefined will usually return false.</li>
</ul> 


## Adding Methods to Enumerated Types
To work on in the furure when in need, don't think it is useful

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