Chapter 9-Parameters
==============================

Be aware that changing a parameter’s default value is potentially dangerous if the method is called from outside the module. A call site embeds the default value into its call. If you later change the parameter’s default value and do not recompile the code containing the call site, then it will call your method passing the old default value. You might want to consider using a default value of 0/null as a sentinel to indicate default behavior; this allows you to change
your default without having to recompile all the code with call sites. Here is an example:

```C#
// Don't do this:
private static String MakePath(String filename = "Untitled") {
   return String.Format(@"C:\{0}.txt", filename);
}

// Do this instead:
private static String MakePath(String filename = null) {
   return String.Format(@"C:\{0}.txt", filename ?? "Untitled");
}
```
If you choose go Don't do this then
```C#
public static void Main() {
    MakePath();     
}

//"Untitled" will be embedded into the calling method(Main in this example)'s IL code, if you don't recompile all the code, you will run into trouble
```

## Passing Parameters by Reference to a Method

By default, the common language runtime (CLR) assumes that all method parameters are passed by value. When reference type objects are passed, the reference (or pointer) to the object is passed (by value) to the method. This means that the method can modify the object and the caller will see the change. For value type instances, a copy of the instance is passed to the method. This means that the method gets its own private copy of the value type and the instance in the caller isn’t affected.

<div class="alert alert-info p-1" role="alert">
    Note that passing parameters by reference is not "pass by reference" as we normally say when differentiate reference type or value type as parameters. Pass by reference means useing <code>ref</code> and <code>out</code>
</div>

The CLR allows you to pass parameters by reference instead of by value. In C#, you do this by using the `out` and `ref` keywords. Both keywords tell the C# compiler to emit metadata indicating that this designated parameter is passed by reference, and the compiler uses this to generate code to pass the
**address of the parameter** rather than the parameter itself

Reference and value types behave very differently with out and ref. Let’s look at using out and ref with value types first. 

If a method’s parameter is marked with `out`, the caller isn't expected to have initialized the object prior to calling the method. The called method **can't read** from the value, and the called method **must write to the value before returning**.
```C#
public sealed class Program {
    public static void Main() {
       Int32 x;              // x is uninitialized.
       GetVal(out x);        // x doesn't have to be initialized.
       Console.WriteLine(x); // Displays "10"
    }
    private static void GetVal(out Int32 v) {
       v = 10; // This method must initialize v.
    }
}
```
If a method’s parameter is marked with `ref`, the caller **must initialize** the parameter's value **prior to calling the method**. The
called method can read from the value and/or write to the value.
```C#
public sealed class Program {
    public static void Main() {
       Int32 x = 5;          // x is initialized. 
       AddVal(ref x);        // x must be initialized.
       Console.WriteLine(x); // Displays "15"
    }
    private static void AddVal(ref Int32 v) {
       v = v + 10; // This method can use the initialized value in v.
    }
}
```

To summarize, from an IL or a CLR perspective, out and ref do exactly the same thing: they both cause a pointer to the instance to be passed. The difference is that the compiler helps ensure that your code is correct.

<div class="alert alert-info p-1" role="alert">
C# requires that a call to a method must specify out or ref even though the compiler could have known whether the method being called requires out or ref
and should be able to compile the code correctly. But the designers of the C# language felt that the caller should explicitly state its intention. This way, at the call site, it’s obvious that the method being called is expected to change the value of the variable being passed.
</div>

Below is a classic example demonstrates why `ref` or `out` is needed when pass reference types parameters
```C#
class Program
{
    static void Main(string[] args) {
        List<int> listA = new List<int> { 1, 2, 3 };
        List<int> listB = new List<int> { 1, 2, 3 };

        Update(listA);
        UpdateRef(ref listB);

        Console.WriteLine("listA");
        foreach (var val in listA)
            Console.WriteLine(val);

        Console.WriteLine("listB");
        foreach (var val in listB)
            Console.WriteLine(val);
    }

    static void Update(List<int> list) {
        list = new List<int>() { 4, 5, 6 };
    }

    static void UpdateRef(ref List<int> list) {
        list = new List<int>() { 4, 5, 6 };
    }
}
```
the output is:
```
//change the format slightly

listA 1 2 3
listB 4 5 6
```
You might wonder why `listA` is not changed to contain 4, 5, 6 since List is a reference type, but here is the catch, when `listA` is passed without `ref`, the stack and heap is like:
```
  Stack                        Heap
------------            ----------------
| ptrlistA |----------->| list (1,2,3) |
------------     |      ----------------
| ptrlistB |-----|----->| list (1,2,3) |
------------     |      ----------------
| ptrlistA |_____|  
------------         
| StackAddr|
|    of    |
| PtrListB |
------------      
```

<div class="alert alert-info p-1" role="alert">
    A copy of the list references(pointers) are pushed on stack as the arguments for the function call. 
</div>

so if you operate on existing object on heap, then you get what you want
```C#
//main
List<int> listA = new List<int> { 1, 2, 3 };
Update(listA);

static void Update(List<int> list) {
   list.Add(4);
   list.Add(5);
   list.Add(6);
}

  Stack                        Heap
------------            ----------------------
| ptrlistA |----------->| list (1,2,3,4,5,6) |
------------     |      ----------------------
| ptrlistB |-----|----->| list (1,2,3)       |
------------     |      ----------------------
| ptrlistA |_____|  
------------          
| StackAddr|
|    of    |
| PtrListB |
------------          
```

But if you make it points to a new object in heap :
```C#
//main
List<int> listA = new List<int> { 1, 2, 3 };
Update(listA);
Console.WriteLine(listA);                 // listA (the first one in stack) still points to original list object on heap

static void Update(List<int> list) {
   list = new List<int>() { 4, 5, 6 };   // list is the third one in stack that points to the new list (4,5,6) on heap
                                         // listA is not affected
}

  Stack                        Heap
------------            ----------------
| ptrlistA |----------->| list (1,2,3) | 
------------            ----------------
| ptrlistB |----------->| list (1,2,3) |
------------            ----------------
|ptrNewlist|----------->| list (4,5,6) |
------------            ----------------
| StackAddr|
|    of    |
| PtrListB |
------------            
```

<div class="alert alert-info p-1" role="alert">
    <code>ref</code> is like double pointer <code>type**</code> in C
</div>

## Using Params to Passing a Variable Number of Arguments to a Method

It's sometimes convenient for the developer to define a method that can accept a variable number of arguments. For example, the System.String type offers methods allowing an arbitrary number of strings to be concatenated together and methods allowing the caller to specify a set of strings that are to be formatted together

To declare a method that accepts a variable number of arguments, you declare the method as follows:

```C#
static Int32 Add(params Int32[] values) {
    ...
}

public static void Main() {
   Console.WriteLine(Add(1, 2, 3, 4, 5));
}
```

When the C# compiler detects a call to a method, the compiler checks all of the methods with the specified name, where no parameter has the `params` applied.
If a method exists that can accept the call, the compiler generates the code necessary to call the method. However, if the compiler can't find a match, it looks for methods that have `params` to see whether the call can be satisfied. If the compiler finds a match, it emits code that **constructs an array** and populates its elements before emitting the code that calls the selected method.

Be aware that calling a method that takes a variable number of arguments incurs an additional performance hit unless you explicitly pass `null`. After all, an array object must be allocated on the heap, the array's elements must be initialized, and the array's memory must ultimately be garbage collected.

```C#
public static void Main() {
   Console.WriteLine(Add());     // passes new Int32[0] to Add
   Console.WriteLine(Add(null)); // passes null to Add: more efficient (no array allocated)
}

static Int32 Add(params Int32[] values) {
    if(values != null)           //  null check
       ...
}
```

To help reduce the performance hit associated with this, you may want to consider defining a few overloaded methods that do not use the params keyword. For some examples, look at the System.String class's Concat method, which has the following overloads:

```C#
public sealed class String : Object, ... {
   public static string Concat(string str0, string str1);
   public static string Concat(string str0, string str1, string str2);
   public static string Concat(string str0, string str1, string str2, string str3);
   public static string Concat(params string[] values);
}
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