Chapter 19-Nullable Value
==============================

As you know, a variable of a value type can never be null; it always contains the value type's value itself. In fact, this is why they call these types ***value type***. Unfortunately, there are some scenarios in which this is a program. For example, when designning a database, it's possible to define a column's data type to be a 32-bit integer that would map to the `Int32` data type of the FCL. But a column in a database can indicate that the value is nullable. That is, it is OK to have no value in the row's column. Working with database data by using .NET Framework can be quite difficuly because in the CLR, there is no way to represent an `Int32` value is null.

Here is another example. In Java, the java.util.Date class is a reference type, and therefore, a variable of this type can be set to null. However, in the CLR, a `System.DateTime` is a value type, and a `DateTime` variable can never be null. If an application written in Java wants to communicate a date/time to a web service running the CLR, there is a problem if the Java application sends null because the CLR has no way tp represent this and operate on it.

To improve this situation, Microsoft added the concept of nullable value types to the CLR. To understand how they work, we first need to look at the `System.Nullable<T>` structure:
```C#
[Serializable, StructLayout(LayoutKind.Sequential)]
public struct Nullable<T> where T : struct {
   // These 2 fields represent the state 
   private Boolean hasValue = false;   // Assume null
   internal T value = default(T);      // Assume all bits zero

   public Nullable(T value) {
      this.value = value;
      this.hasValue = true;
   }  

   public Boolean HasValue { get { return hasValue; } }

   public T Value {
      get {
         if (!hasValue) {
            throw new InvalidOperationException("Nullable object must have a value.");
         }
         return value;
     }
   } 

   public T GetValueOrDefault() { return value; } 

   public T GetValueOrDefault(T defaultValue) {
      return hasValue ? value : defaultValue;
   } 

   public override Boolean Equals(Object other) {
     if (!hasValue) 
        return other == null;
     if (other == null) 
        return false;
     return value.Equals(other);   // a little bit tricky here, you can pass either T or Nullable<T>, check the Int32 source code below
   } 

   public override int GetHashCode() {
      if (!hasValue) 
         return 0;
      return value.GetHashCode();
   }

   public static implicit operator Nullable<T> (T value) {
      return  new NUllable<T>(value);
   }

   public static explicit operator T(Nullable<T> value) {
     return value.Value;
   }
}

//A concrete example to show how Equals work
public struct Int32 : ... {
   ...
   public override bool Equals(Object obj) {
      if (!(obj is Int32)) {   // You want to do null check here, if object is null then return false immediately but the expression xxx is null is computed   
                               // differently for reference types and nullable value types. 
                               // for nullable value types, it uses Nullable<T>.HasValue. For reference types, it uses (object)xxx == null.
         return false;                
      }
      return m_value == ((Int32)obj).m_value;   // Here you can see if Nullable<Int32> is passed as obj argument, Nullable<T>'s explicit casting involved
   }
}
```
Because `Nullable<T>` is itself a value type, instances of it are still fairly lightweight. That is, instances can still be on the stack, and an instance is **the same size as the original value type plus the size of a Boolean field**. Notice that Nullable's type parameter, T, is constrained to struct. This was done because reference type variables can already be null.

So now, if you want to use a nullable Int32 in your code, you can write something like this:
```C#
Nullable<Int32> x = 5;
Nullable<Int32> y = null;
Console.WriteLine("x: HasValue={0}, Value={1}", x.HasValue, x.Value);
Console.WriteLine("y: HasValue={0}, Value={1}", y.HasValue, y.GetValueOrDefault());
```
Note that `Nullable<Int32> x = 5;` works because of `Nullable<T>`'s explicit casting `public static implicit operator Nullable<T> (T value) { return  new NUllable<T>(value); }`. You might wonder why `Nullable<Int32> y = null;` works because `null` is not a valid struct type (`T` is constrained to struct). The reason it works because checking for null and assigning null to a variable of type `Nullable<T>` is explicitly covered by the compiler:
```C#
Nullable<Int32> y = null;
```
is converted to
```C#
Nullable<Int32> y = new Nullable<int>(); // or Nullable<Int32> y = default(Nullable<int>); implementation details, doesn't matter
```
and
```C#
if (y == null)
```
is converted to
```C#
if (!y.HasValue)
```

<div class="alert alert-info p-1" role="alert">
   Some other languages like Javascript will convert <code>null</code> to <code>false</code>, but in C#, it is not supported as true/false is a boolean struct, value types cannot have a null value and the if statment needs a boolean struct, it cannot reveive a nullable boolean struct, so code below won't compile:
</div>

```C#
if (null) {   // Compile error CS0037 Cannot convert null to 'bool' becuase it is a non-nullable value type
   ...
}

bool? a = true;
if (a) {      // Compile error CS0266 Cannot implicitly convert type `bool?` to `bool`. An explicit conversion exists...
   ...
}
```
You might ask why compiler doesn't add native support that "retrieve" value from `Nullable<Boolean>` automatically so that it can be used in if statment mentioned above. Check the following warning description:

<div class="alert alert-info p-1" role="alert">
  You (actually I) might wonder why nullable doesn't overload implicit casting (instead of explicit casting) from nullable to T as <br><code>public static explicit operator T(Nullable<T> value) { return value.Value  }<T></code><br>So that you can do: <br><code>Int32? a = 5; <br>Int32 b = a;</code><br>The reasion it is not supprose to do this is because implicit conversions should not throw exceptions (it is always safe to perform implicit conversions). That's one of the key reasons for deciding whether to make implicit or explicit (the other being, generally, whether loss of precision is possible). Basically, they're implicit when you don't have to worry about the conversion, and you use explicit conversion to acknowledge that it might throw an exception in situation like: <br><code>Int32? a = null; <br>Int32 b = (Int32) a;  // throws System.InvalidOperationException: 'Nullable object must have a value.' in runtime</code>
</div>

## C#'s Support for Nullable Value Types

Notice in the code that C# allows you to use fairly simple syntax to initialize the two `Nullable<Int32>` variables, x and y. In fact, the C# team wants to integrate nullable value types into the C# language, making them first-class citizens. To that end, C# offers a cleaner syntax for working with nullable value
types. C# allows the code to declare and initialize the x and y variables to be written using question mark notation:
```C#
Int32? x = 5;
Int32? y = null;
```
In C#, `Int32?` is a synonym notation for `Nullable<Int32>`. But C# takes this further. C# allows you to perform conversions and casts on nullable instances. And C# also supports applying operators to nullable instances. The following code shows examples of these:
```C#
// Implicit conversion from non-nullable Int32 to Nullable<Int32>
Int32? a = 5;

// Implicit conversion from 'null' to Nullable<Int32>
Int32? b = null; // Same as "Int32? b = new Int32?();" which sets HasValue to false 

// Explicit conversion from Nullable<Int32> to non-nullable Int32
Int32 c = (Int32) a;

// Casting between nullable primitive types
Double? d = 5; // Int32->Double? (d is 5.0 as a double)
Double? e = b; // Int32?->Double? (e is null) 
```

C# also allows you to apply operators to nullable instances. The following code shows examples of this:
```C#
Int32? a = 5;
Int32? b = null;

a++;    // a = 6
b = -b; // b = null

a = a + 3; // a = 9
b = b * 3; // b = null;
```

Note that if you compare Nullable types:
```C#
public bool F(int? a, int? b) {
   return a == b;
}
```
You (actually I) might think it uses `Object`'s `public static bool Equals(Object objA, Object objB)` (`==` calls this static method interanlly normally), which you might further think `a` and `b` will be casted to objects. But actually compiler convertes the above code to:
```C#
public bool F(Nullable<int> a, Nullable<int> b) {
   Nullable<int> num = a;   // not sure why the compiler introduces these two temp variables, not important
   Nullable<int> num2 = b;
   return (num.GetValueOrDefault() == num2.GetValueOrDefault()) & (num.HasValue == num2.HasValue);   // no boxing involved
}
```
Note that `&` bitwise operator is used here, and you (actually I) might think that it is better to use conditional logical operator `&&` to short-circuit the evaluation when the first part is false. Branching is very expensive when you can't predict it.(Check CSAPP Ch4 for Processor Architecture).That's why compilers will emit `&` instead of `&&` in many such situations to avoid branching.

Mentioning `&` happens to bring an unrelated fact: ***conditional logical operators && and || don't support `Nullable<Boolean>` operands.*** Because to put in simple, both operends on logical operators has to be `Boolean` type, not `Nullable<Boolean>`, to give a thorough explanation, operators on `Nullable<T>` are "lifted" operators,, which means if T has the operator, T? will have the "lifted" counterpart. `&&` and `||` aren't really operators in the same sense as & and | - for example, they can't be overloaded - from the ECMA spec 14.2.2 Operator overloading:
>The overloadable binary operators are: + - * / % & | ^ << >> == != > < >= <= Only the operators listed above can be overloaded. In particular, it is not possible to overload member access, method invocation, or the =, &&, ||, ??, ?:, checked, unchecked, new, typeof, as, and is operators.

Here is how C# interprets the operators:

<ul>
  <li><b>Unary operators (+, ++, -, --, ! , ~)</b> If the operand is null, the result is null</li>
  <li><b>Binary operators (+, -, *, /, %, &, |, ^, <<, >>)</b> If either operand is null, the result is null. However, an exception is made when the & and | operators are operating on <code>Boolean?</code> operands, so that the behavior of these two operators gives the same behavior as demonstrated by SQL's three-valued logic. For these two operators, if neither operand is null</li>
  <li><b>Equality operators (==, !=)</b> If both operands are null, they are equals. If one operand is null, they are not equal. If neither operand is null, compare the values to determine if they are equal.</li>
  <li><b>Relational operators (<, >, <=, >=) </b> If either operand is null, the result is false. If neither operand is null, compare the values.</li>
</ul> 

You should be aware that manipulating nullable instances does generate a lot of code. For example, see thje following method:
```C#
private static Int32? NullableCodeSize(Int32? a, Int32? b) {
   return a + b; 
}
```
When compiling this method, there is quite a bit of resulting Intermediate Language (IL) code, which also makes performing operations on nullable types slower than performing the same operation on non-nullable types. Here is the C# equivalent of the compiler-produced IL code:
```C#
private static Nullable<Int32> NullableCodeSize(Nullable<Int32> a, Nullable<Int32> b) {
   Nullable<Int32> nullable1 = a;
   if (!(nullable1.HasValue & nullable2.HasValue)) {
      return new Nullable<Int32>();
   }
   return new Nullable<Int32>(nullable1.GetValueOrDefault() + nullable2.GetValueOrDefault());
}
```
Finally, let me point out that you can define your own value types that overload the various operators previously mentioned. I discuss how to do this in the "Operator Overload Methods" section in Chapter 8-Methods. If you then use a nullable instance of your own value type, the compiler does the right thing and invokes your overloaded operator. For example, suppose that you have a `Point` value type that defines overloads for the `==` and `!=` operators as follows:
```C#
internal struct Point {
   private Int32 m_x, m_y;
   public Point(Int32 x, Int32 y) { m_x = x; m_y = y; }

   public static Boolean operator ==(Point p1, Point p2) {
      return (p1.m_x == p2.m_x) && (p1.m_y == p2.m_y);
   }

   public static Boolean operator !=(Point p1, Point p2) {
      return !(p1 == p2);
   }
}
```
At this point, you can use nullable instances of the `Point` type and the compiler will invoke your overloaded operators:
```C#
public static void Main() {
   Point? p1 = new Point(1, 1);
   Point? p2 = new Point(2, 2);
   Console.WriteLine("Are points equal? " + (p1 == p2).ToString());       // Are points equal? False
   Console.WriteLine("Are points not equal? " + (p1 != p2).ToString());   // Are points not equal? True 
}
```

## C#'s Null-Coalescing Operator

C# has an operator called the null-coalescing operator (`??`), which takes two operands. If the operand on the left is not `null`, the operand's value is returned. If the operand on the left is `null`, the value of the right operand is returned. The null-colaescing operator offers a very convenient way to set a variable's default value.

A cool feature of the null-coalescing operator is that it can be used with reference types as well as nullable value types. Here is some code that demonstates the use of the null-coalescing operator:
```C#
private static void NullCoalescingOperator() {
   Int32? b = null;

   // The following line is equivalent to:  x = (b.HasValue) ? b.Value : 123 
   Int32 x = b ?? 123;

   // The following line is equivalent to:
   // String temp = GetFilename();
   // filename = (temp != null) ? temp : "Untitled";
   String filename = GetFilename() ?? "Untitled"; 
}
```
Some people argue that the null-coalescing operator is simply syntactic sugar for the `?:` operator, and that the C# compiler team should not have added this operator to the language. However, the null-coalescing operator offers two significant syntactic improvement. The first is the `??` operator works better with expressions:
```C#
Func<String> f = () => SomeMethod() ?? "Untitled";
```
This code is much easier to read and understand than the following line, which requires variable assignments and multiple statements:
```C#
Func<String> f = () => {
   var temp = SomeMethod();
   return temp ! = null ? temp : "Untittled";
}
```
The second improvement is that `??` works better in composition scenarios. For example, the following single line:
```C#
String s = SomeMethod1() ?? SomeMethod2() ?? "Untitled";
```
is far easier to read and understand than this chunk of code:
```C#
String s;
var sm1 = SomeMethod1();
if (sm1 != null) 
   s = sm1;
else {
   var sm2 = SomeMethod2();
   if (sm2 != null) 
      s = sm2;
   else
      s = "Untitled";
}
```

## Null-conditional operators ?. and ?[]
Available in C# 6 and later, a null-conditional operator applies a member access, ?., or element access, ?[], operation to its operand only if that operand evaluates to non-null; otherwise, it returns null. That is:
<ul>
  <li>If a evaluates to null, the result of a?.x or a?[x] is null.</li>
  <li>If a evaluates to non-null, the result of a?.x or a?[x] is the same as the result of a.x or a[x], respectively.</li>
</ul> 


## The CLR Has Special Support for Nullable Value Types

The CLR has built-in support for nullable value types. This special support is provided for boxing, unboxing, calling `GetType()`, calling interface methods, and it is given to nullable types to make them fit more seamlessly into the CLR. This also makes them behave more natually and as most developers would expect. Let's take a closer look at the CLR's specail support for nullable types.

#### Boxing Nullable Value Types

Image a `Nullable<Int32>` variable that is logically set to `null`. If this variable is passed to a method prototyped as expecting an `Object`, the variable must be boxed, and a reference to the boxed `Nullable<Int32>` is passed to the method. This is not ideal because the method is now being passed a non-null value even though the `Nullable<Int32>` variable logically contained the value of null. To fix this, the CLR executes some special code when boxing a nullable variable to keep up the illusion that nullable types are first-class citizens in the environment.

Specifically, when the CLR is boxing a `Nullable<T>` instance, it checks to see if it is `null`, and if so, the CLR doesn't actually box anything, and null is returned, If the nullable isntance is not null, the CLR takes the value out of the nullalble instance and boxes it, In other words, a `Nullable<Int32>` with a value of 5 is boxed into a boxed-Int32 with a value of 5, Here is some code that demonstrates this behavior:
```C#
// Boxing Nullable<T> is null or boxed T
Int32? n = null;
Object o = n; // o is null, no boxing involves
Console.WriteLine("o is null={0}", o == null); // "True" 

n = 5;
o = n; // o refers to a boxed Int32
Console.WriteLine("o's type={0}", o.GetType()); // "System.Int32"
```

#### Unboxing Nullable Value Types

The CLR allows a boxed value type T to be unboxed into a `T` or a `Nullable<T>`. If the reference to the boxed value type is null, and you are unboxing it to a `Nullable<T>`, the CLR sets `Nullable<T>`'s value to null. Here is some code to demonstrate this behavior:
```C#
// Create a boxed Int32
Object o = 5;

// Unbox it into a Nullable<Int32> and into an Int32
Int32? a = (Int32?) o; // a = 5
Int32 b = (Int32) o; // b = 5

// Create a reference initialized to null
o = null;

// "Unbox" it into a Nullable<Int32> and into an Int32
a = (Int32?) o; // a = null
b = (Int32) o; // NullReferenceException
```

#### Calling GetType via a Nullable Value Type

When calling `GetType()` on a `Nullable<T>` object, the CLR actually lies and returns the type `T` instead of the type `Nullable<T>`. Here is some code that demonstrates this behavior:
```C#
Int32? x = 5;

// The following line displays "System.Int32"; not "System.Nullable<Int32>"
Console.WriteLine(x.GetType());
```

#### Calling Interface Methods via a Nullable Value Type

In the following code, I'm casting n, a `Nullable<Int32>`, to `IComparable<Int32>`, an interface type. However, the `Nullable<T>` type doesn't implement the `IComparable<Int32>` interface as `Int32` does. The C# compiler allows this code to compile anyway, and the CLR's verifier considers this code verifiable to allow you a  more convenient syntax:
```C#
Int32? n = 5;
Int32 result = ((IComparable) n).CompareTo(5);   // Compiles & runs OK
```
If the CLR didn't provide this special support, it would be more cumbersome for you to write code to call an interface method on a nullable value type. You’d have to cast the unboxed value type first before casting to the interface to make the call:
```C#
Int32 result = ((IComparable) (Int32) n).CompareTo(5); // Cumbersome
```

## Nullable Reference Types

Prior to C# 8.0, all reference types were nullable. Start from C# 8.0, **everything is non-nullable by default** (The `null` literal type is the only type that is nullable by default), so reference types is like structs now (structs can't be null).
So from C# 8.0, we need to explicit make a reference type nullable if we need it accept `null` value, and we can't think in pre-C# 8.0 way that reference types can be nullable.

The ability of making reference type nullable (well, reference type can be null actually, we make it non-nullable in code analysis level, not CLR level) solve a problem of this:
```C#
public void SomeMethod<T>(T? value) { ... }

// before C# 8.0, nullable only makes senses for structs type, and you don't know if the T is struct or class at compile time, what if T is class? what does class can be null when 
// it is design to be allowed for null?
```
You can see that by assuming class is non-nullable by default can unify the usage of `?`.

Depending on the VS version and target framework you use, you might need to add `<Nullable>enable</Nullable>` in your .csproj file:
```C#
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>netcoreapp3.0</TargetFramework>
    <LangVersion>8.0</LangVersion>
    <Nullable>enable</Nullable>  // <------------------
  </PropertyGroup>
</Project>
```


Let's say we have the following code without Nullable function turning on using C# 8.0 or above:
```C#
// C# 8.0
static void Main(string[] args) {
   string name = Test(0);
   name.ToString();
   Console.WriteLine(name);
}

static string Test(int i) {
   if (i <= 0) {
      return null;
   }
   return "Tom";
}
```
Now let's turn on the nullable reference type function in csproj file:
```
<Nullable>enable</Nullable>
```

When we turn this feature on, C# compiler will treat all reference types non-nullable by default. So there will be warning in 
```C#
// 1st warning
static string Test(int i) {  // <--- compiler CS8600 warning "Converting null literal or possible null value to non-nullable type (string name)"
   if (i <= 0) {
      return null;
   }
   return "Tom";
}

//more warning, don't worry about the ToString on a string object, weird but just for demo purpose
static void Main(string[] args) {
   string name = Test(0);  // <--- same CS8600 warning here
   name.ToString();        // <--- compiler CS8602 "Warning Dereference of a possibly null reference" 
   Console.WriteLine(name);
}
```
To fix, we'll do:
```C#
static string? Test(int i) {  
   if (i <= 0) {
      return null;
   }
   return "Tom";
}

static void Main(string[] args) {
   string? name = Test(0); 
   if (name == null)   // doing a null check makes compiler turns off the warning
      return;
   else
     Console.WriteLine(name.ToString()); 
          
}
```
You can use null-forgiving operator `!` if you are confident that the variable won't be null so there won't be warning:
```C#
static void Main(string[] args) {
   string? name = Test(0); 
   Console.WriteLine(name!.ToString());          
}
```

There are also a range of attributes you can use to inform compilier, for example:
```C#
static void ThrowIfNull([NotNull] string? x)
{
    if (x == null)
    {
        throw new ArgumentNullException();
    }
}
```
`NotNull` attributes tell the compiler that "if this method return then `x` won't be null, which can be considered when the caller who calls this method, `NotNull` itself is not about callee, otherwise it would be weird to say `x` is nullable (because of `?`) and also say `x` is not null (`[NotNull]` before it), it is about caller. For example:
```C#
static void Caller()
{
   string? str = GetString();
   ThrowIfNull(str);
   Console.WriteLine(str.ToUpper());   // compiler warning that .ToUpper() is a possible NRE if you do NOT use [NotNull] in the callee's parameter
}
```


Something interesting:
```C#
string a = null;     // warning
string b = null!;    // ok, forcely assign null to b
string c = default!; // ok, forcely assign null to c
```
You might wonder why you want to forcely assign `null` to a non-nullable reference type. Assigning `null!` effectively says "I know this should never be null, but guess what, I'm doing it anyway". But why do you do that? well, the only reason to do it would be because you know you're going to assign a non-null value before any other code could get a `NullReferenceException`, so you want to signal or other developers (pull request code review, peer progamming etc, or even yourself when you forgot what you wrote is for after a long time) that you haven't forgotten to assign it. So now you don't need to worry about what initial value you need to assign to a non-nullable

The case of `var` type:

When nullable reference types are enabled, the compiler always consider the type is nullable:

```C#
var str = "hello";
// equivalent to
string str? = "hello";
```

This choice was made to avoid lots of compilation warnings when enabling nullable reference types and identity potential null. Think about the following case:

```C#
var str = "hello";
_ = str.Length; 

str = null; // compiler would produce an warning if str was `string` instead of `string?`, getting one extra warning
// ...
```











<!-- <code>&lt;T&gt;</code> -->

<!-- <div class="alert alert-info p-1" role="alert">
    
</div> -->

<!-- <div class="alert alert-info pt-2 pb-0" role="alert">
    <ul class="pl-1">
      <li></li>
      <li></li>
    </ul>  
</div> -->

<!-- <ul>
  <li><b></b></li>
  <li><b></b></li>
  <li><b></b></li>
  <li><b></b></li>
</ul>  -->

<!-- ![alt text](./zImages/16-1.png "Title") -->

<!-- <span style="color:red">hurt</span> -->

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