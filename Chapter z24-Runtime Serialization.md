Chapter 24-Runtime Serialization
==============================

*Serialization* is the process of converting an object or a graph of connected objects into a stream of bytes. *Deserialization* is the process of converting a stream of bytes back into its graph of connected objects. The ability to convert objects to and from a byte stream is an incredibly useful mechanism.
Here is an examples: An application's state (object graph) can easily be saved in a disk file or database and then restored the next time the application is run. ASP.NET saves and restores session state by way of serialization and deserialization

.NET Framework has fantastic support for serialization and deserialization built right into it. This means that all of the difficult issues are now handled completely and transparently by the .NET Framework. As a developer, you can work with
your objects before serialization and after deserialization and have the .NET Framework handle the
stuff in the middle.


## Serialization/Deserialization Quick Start

Let's start off by looking at some code:
```C#
static void Main(string[] args) {
   // Create a graph of objects to serialize them to the stream
   List<String> objectGraph = new List<String> { "Jeff", "Kristin", "Aidan", "Grant" };
   Stream stream = SerializeToMemory(objectGraph);

   // Reset everything for this demo
   stream.Position = 0;
   objectGraph = null;

   // Deserialize the objects and prove it worked
   // Deserialize the objects and prove it worked
   objectGraph = (List<String>)DeserializeFromMemory(stream);
   foreach (var s in objectGraph)
      Console.WriteLine(s);

   Console.ReadLine();
}

static MemoryStream SerializeToMemory(Object objectGraph) {
   // Construct a stream that is to hold the serialized objects
   MemoryStream stream = new MemoryStream();

   // Construct a serialization formatter that does all the hard work
   BinaryFormatter formatter = new BinaryFormatter();

   // Tell the formatter to serialize the objects into the stream
   formatter.Serialize(stream, objectGraph);

   // Return the stream of serialized objects back to the caller
   return stream;
}

static Object DeserializeFromMemory(Stream stream) {
   // Construct a serialization formatter that does all the hard work
   BinaryFormatter formatter = new BinaryFormatter();

   // Tell the formatter to deserialize the objects from the stream
   return formatter.Deserialize(stream);
}
```
When serializing an object, the full name of the type and the name of the type’s defining assembly are written to the stream. By default, BinaryFormatter outputs the assembly's full identity, which includes the assembly’s file name (without extension), version number, culture, and public key information. When deserializing an object, the formatter first grabs the assembly identity and ensures that the assembly is loaded into the executing AppDomain by calling System.Reflection.Assembly’s Load method

After an assembly has been loaded, the formatter looks in the assembly for a type matching that of the object being deserialized. If the assembly doesn’t contain a matching type, an exception is thrown and no more objects can be deserialized. If a matching type is found, an instance of the type is created and its fields are initialized from the values contained in the stream. If the type's fields don't exactly match the names of the fields as read from the stream, then a SerializationException exception is thrown and no more objects can be deserialized.

## Making a Type Serializable

When a type is designed, the developer must make the conscious decision as to whether or not to allow instances of the type to be serializable. **By default, types** are not serializable. For example, the following code does not perform as expected :
```C#
internal struct Point { public Int32 x, y; }

private static void OptInSerialization() {
   Point pt = new Point { x = 1, y = 2 };
   using (var stream = new MemoryStream()) {
      new BinaryFormatter().Serialize(stream, pt); // throws SerializationException
   }
}
```
To solve this problem, the developer must apply the System.SerializableAttribute custom attribute to this type as follows:
```C#
[Serializable]
internal struct Point { public Int32 x, y; }
```

<div class="alert alert-info p-1" role="alert">
    When serializaing a graph of objects, some of the object's types may be serializable and some of the objects may not be serializable. For performance reasons, formatters do not verify that all of the objects in the graph are serializable before serializing the graph. So, when serializing an object graph, it is entirely possible that some objects may be serialized to the stream before the SerializationException is thrown. If this happens, the stream contains corrupt data.
</div>

The `SerializableAttribute` custom attribute may be applied to reference types (class), value types (struct), enumerated types (enum), and delegate types only. Note that enumerated and delegate types are always serializable, so there is no need to explicitly apply the attribute to these types. **In addition, the SerializableAttribute attribute is not inherited by derived types**. So, given the following two type definitions, a Person object can be serialized, but an Employee object cannot :
```C#
[Serializable]
internal class Person { ... }

internal class Employee : Person { ... }
```
To fix this, you would just apply the SerializableAttribute attribute to the Employee type as well:
```C#
[Serializable]
internal class Person { ... }

[Serializable]
internal class Employee : Person { ... }
```
Note that this problem was easy to fix. However, the reverse—defining a type derived from a base type that doesn’t have the SerializableAttribute attribute applied to it—is not easy to fix. But, this is by design; if the base type doesn’t allow instances of its type to be serialized, its fields cannot be serialized, because a base object is effectively part of the derived object. This is why System. Object has the SerializableAttribute attribute applied to it.

<div class="alert alert-info p-1" role="alert">
 In general, it is recommended that most types you define be serializable. After all, this grants a lot of flexibility to users of your types. However, you must be aware that serialization reads all of an object’s fields regardless of whether the fields are declared as public, protected, internal, or private. You might not want to make a type serializable if it contains sensitive or secure data (like passwords) or if the data would have no meaning or value if transferred.
</div>

## Controlling Serialization and Deserialization

When you apply the SerializableAttribute custom attribute to a type, all instance fields (public, private, protected, and so on) are serialized. However, a type may define some instance fields that should not be serialized. In general, there are two reasons why you would not want some of a type's instance fields to be serialized:

<ul>
  <li>The field contains information that would not be valid when deserialized. For example, an object that contains a handle to a Windows kernel object (such as a file, process, thread, mutex, event, semaphore, and so on) would have no meaning when deserialized into another process or machine because Windows’ kernel handles are process-relative values</li>
  <li>The field contains information that is easily calculated. In this case, you select which fields do not need to be serialized, thus improving your application's performance by reducing the amount of data transferred.</li>
   <li>Some data structures like Dictionary that uses hashcode internally. <code>TKey.GetHashCode()</code> might not return the same hash code when called in different AppDomains, for objects which are considered equal. string for example adds a random factor, so that "test".GetHashCode() will can return different values in different AppDomains, as a security feature.</li>
</ul> 

<div class="alert alert-info p-1" role="alert">
     Do not use C#’s automatically implemented property feature to define properties inside types marked with the <code>[Serializable]</code> attribute, because the compiler generates the names of the fields and the generated names can be different each time that you recompile your code, preventing instances of your type from being deserializable.
</div>

The following code uses the System.NonSerializedAttribute custom attribute to indicate which fields of the type should not be serialized:
```C#
[Serializable]
internal class Circle {
   private Double m_radius;
      
   [NonSerialized]
   private Double m_area;

   public Circle(Double radius) {
      m_radius = radius;
      m_area = Math.PI * m_radius * m_radius;
   }
   ...
}
```
But now we have a problem when the stream is deserialized back into a Circle object. When deserialized, the Circle object will get its m_radius field set to 10, but its m_area field will be initialized to 0—not 314.159! The following code demonstrates how to modify the Circle type to fix this problem:
```C#
[Serializable]
internal class Circle {
   private Double m_radius;
      
   [NonSerialized]
   private Double m_area;

   public Circle(Double radius) {
      m_radius = radius;
      m_area = Math.PI * m_radius * m_radius;
   }
   
   [OnDeserialized]
   private void OnDeserialized(StreamingContext context) {
      m_area = Math.PI * m_radius * m_radius;
   }
}
```
Now Whenever an instance of a type is deserialized, **the formatter checks whether the type defines a method with this attribute on it and then the formatter invokes this method**. When this method is called, all the serializable fields will be set correctly, and they may be accessed to perform any additional work that would be necessary to fully deserialize the object.

In addition to the OnDeserializedAttribute custom attribute, the System.Runtime.Serialization namespace also defines `OnSerializingAttribute` `OnSerializedAttribute`, and `OnDeserializingAttribute` custom attributes, which you can apply to your type’s methods to have even more control over serialization and deserialization:
```C#
[Serializable]
public class MyType {
   Int32 x, y; [NonSerialized] Int32 sum;
   
   public MyType(Int32 x, Int32 y) {
      this.x = x; this.y = y; sum = x + y;
   }

   [OnDeserializing]
   private void OnDeserializing(StreamingContext context) {
      // Example: Set default values for fields in a new version of this type
   }

   [OnDeserialized]
   private void OnDeserialized(StreamingContext context) {
      // Example: Initialize transient state from fields
      sum = x + y;
   }

   [OnSerializing]
   private void OnSerializing(StreamingContext context) {
      // Example: Modify any state before serializing
   }

   [OnSerialized]
   private void OnSerialized(StreamingContext context) {
      // Example: Restore any state after serializing
   }
}
```
The name of the method can be anything you want it to be. Also, you should declare the method as private to prevent it from being called by normal code; the formatters run with enough security that they can call private methods.

<div class="alert alert-info p-1" role="alert">
   When you are serializaing a set of obejcts, the formatter first calls all of the object's methods that are marked with the OnSerializing attribure. Next, it serializes all of the object' fields, and finally it calls all of the object's methods marked with the OnSerialized attribute. Similarly, when you deserialize a set of objects, the foramatters calls all of the object's methods that are marked with the OnDeserializing attribute, then it deserializes all of the object's fields, and then it calls all of the objects' methods marked with the OnDeserialized attribute.
   <br><br>
   Note also that during deserialization, when a formatter sees a type offering a method marked with the OnDeserialized attribute, the formatter adds this object's reference to an internal list. After all the objects have been deserialized, the formatter traverses this list in reverse order and calls each object's OnDeserialized method. When this method is called, all the serializable fields will be necessary to fully deserialize the object. Invoking these methods in reverse order is important because it allows inner objects to finish their deserialization before the outer objects that contain them finish their deserialization. 
   <br><br>
   For example, imagine a collection object (like Hashtable or Dictionary) that internally uses a hash table to maintain its sets of items. The collection object type would implement a method marked with the On Deserialized attribute. Even though the collection object would start being deserialized first (before its items), its OnDeserialized method would be called last (after any of its item's OnDeserialized methods). This allows the items to complete deserialization so that all their fields are initialized properly, allowing a good hash code value to be calculated. Then, the collection object creates its internal buckets and uses the item's hash codes to place the items into the buckets. I show an example of how the Dictionary class uses this in the upcoming "Controlling the Serialized/ Deserialized Data" section of this chapter
</div>

If you serialize an instance of a type, add a new field to the type, and then try to deserialize the object that did not contain the new field, the formatter throws a `SerializationException` with a message indicating that the data in the stream being deserialized has the wrong number of members. This is very problematic in versioning scenarios where it is common to add new fields to a type in a newer version. Fortunately, you can use the `OptionalFieldAttribute` attribute to help you.

## How Formatters Serialize Type Instances

In this section, I give a bit more insight into how a formatter serializes an object's fields. This knowledge can help you understand the more advanced serialization and deserialization techniques explained in the remainder of this chapter.

To make things easier for a formatter, the FCL offers a `FormatterServices` type:
```C#
namespace System.Runtime.Serialization {
   public static class FormatterServices {
      public static MemberInfo[] GetSerializableMembers(Type type, StreamingContext context);
      public static object[] GetObjectData(object obj, MemberInfo[] members);  
      public static Type GetTypeFromAssembly(Assembly assem, string name);       
      public static object GetUninitializedObject(Type type);
      public static object PopulateObjectMembers(object obj, MemberInfo[] members, object[] data);
      ...
   }
}
```

This type has only static methods in it, and no instances of the type may be instantiated. The following steps describe how a formatter automatically serializes an object whose type has the SerializableAttribute attribute applied to it

1. The formatter calls FormatterServices's GetSerializableMembers method:
```C#
public static MemberInfo[] GetSerializableMembers(Type type, StreamingContext context);
```
This method uses reflection to get the type's public and private instance fields (excluding any fields marked with the NonSerializedAttribute attribute). The method returns an array of MemberInfo objects, one for each serializable instance field.

2. The object being serialized and the array of MemberInfo objects are then passed to FormatterServices' static GetObjectData method:
```C#
public static Object[] GetObjectData(Object obj, MemberInfo[] members);
```
This method returns an array of Objects where each element identifies the value of a field in the object being serialized. This Object array and the MemberInfo array are parallel. That is, element 0 in the Object array is the value of the member identified by element 0 in the MemberInfo array. 

Note that BinaryFormatter doesn't write key into the stream assuming the reader knows the exact structure and writes the values onto the correct properties, which makes BinaryFormatter is not recommended for data processing. Applications should stop using BinaryFormatter as soon as possible.

3. The formatter writes the assembly's identity and the type's full name to the stream.

4. The formatter then enumerates over the elements in the two arrays, writing each member's name (author said name is also written to the stream, but I think it is not) and value to the stream.

The following steps describe how a formatter automatically deserializes an object whose type has the SerializableAttribute attribute applied to it:

1. The formatter reads the assembly’s identity and full type name from the stream. If the assembly is not currently loaded into the AppDomain, it is loaded (as described earlier). If the assembly can't be loaded, a SerializationException exception is thrown and the object cannot be deserialized. If the assembly is loaded, the formatter passes the assembly identity information and the type's full name to FormatterServices' static GetTypeFromAssembly method.
```C#
public static Type GetTypeFromAssembly(Assembly assem, String name)
```
This method returns a System.Type object indicating the type of object that is being deserialized.

2. The formatter calls FormatterServices's static GetUninitializedObject method
```C#
public static Object GetUninitializedObject(Type type);
```
This method allocates memory for a new object but does not call a constructor for the object. However, all the object's bytes are initialized to null (reference type members) or 0 (value type members).

3. The formatter now constructs and initializes a MemberInfo array as it did before by calling the FormatterServices's GetSerializableMembers method. This method returns the set of fields that were serialized and that need to be deserialized.

4. The formatter creates and initializes an Object array from the data contained in the stream.

5. The reference to the newly allocated object, the MemberInfo array, and the parallel Object array of field values is passed to FormatterServices' static PopulateObjectMembers method
```C#
public static Object PopulateObjectMembers(Object obj, MemberInfo[] members, Object[] data);
```
This mehtod enumerates over the arrays, initializing each field to its corresponding value. At this point, the object has been completely desrialized.


## Controlling the Serialized/Deserialized Data

As discussed earlier in this chapter, the best way to get control over the serialization and deserialization process is to use the OnSerializing, OnSerialized, OnDeserializing, OnDeserialized, NonSerialized, and OptionalField attributes. However, there are some very rare scenarios where these attributes do not give you all the control you need. In addition, the formatters use reflection internally and reflection is slow, which increases the time it takes to serialize and deserialize objects. To get complete control over what data is serialized/deserialized or to eliminate the use of reflection, your type can implement the `System.Runtime.Serialization.ISerializable` interface, which is defined as follows:
```C#
public interface ISerializable {
   void GetObjectData(SerializationInfo info, StreamingContext context);
}
```
This interface has just one method in it, GetObjectData. But most types that implement this interface will also implement a **special constructor** that I'll describe shortly

<div class="alert alert-info p-1" role="alert">
    The ISerializable interface and the special constructor are intended to be used by the formatters. However, other code could call GetObjectData, which might then return potentially sensitive information, or other code could construct an object that passes in corrupt data. For this reason, it is recommended that you apply the following attribute to the GetObjectData method and the special constructor.
    <br><br>
    <code>[SecurityPermissionAttribute(SecurityAction.Demand, SerializationFormatter = true)]</code>
</div>

When a formatter serializes an object grapg, it looks at each object. If its type implements the ISerializable interface, then the formatter ognores all custom attributes and instread constructs a new `System.Runtime.Serialization.SerializationInfo` object
```C#
public sealed class SerializationInfo {
   public SerializationInfo(Type type, IFormatterConverter converter);

   public Type ObjectType { get; }
   public string AssemblyName { get; set; }
   public string FullTypeName { get; set; }

   public void AddValue(string name, object value, Type type);
   public void AddValue(string name, int value);
   public void AddValue(string name, decimal value);
   public void AddValue(string name, DateTime value);
   ...
   public object GetValue(string name, Type type);
   public int GetInt32(string name);
   public double GetDouble(string name);
   public void SetType(Type type);   // overwrite FullTypeName and AssemblyName properties

}
```
This object contains the actual set of values that should be serialized for the object.

When constructing a SerializationInfo object, the formatter passes two parameters: Type and IFormatterConverter. The type paramater identifies the object that is being serializaed. Two pieces of infomation are requried to uniquely identify a type: the string name of the type and its assembly's identity (which includes the assembly name, version, culture, and public key). When a SerializationInfo object is constructed, it obtains the type's full name (by internally querying Type’s FullName property) and stores this string in a private field. You can obtain the type's full name by querying SerializationInfo’s FullTypeName
property. Likewise, the constructor obtains the type’s defining assembly (by internally querying Type's Module property followed by querying Module’s Assembly property followed by querying Assembly's FullName property) and stores this string in a private field. You can obtain the assembly's identity by querying SerializationInfo's AssemblyName property

<div class="alert alert-info p-1" role="alert">
   Although you can set a SerializationInfo's FullTypeName and AssemblyName properties, this is discouraged. If you want to change the type that is being serialized, it is recommended that you call SerializationInfo's SetType method, passing a reference to the desired Type object. Calling SetType ensures that the tyeps's full name and defining assembly are set correctly. An example of calling SetType is shown in the "Serializing a Type As a Different Type and Deserializing an Object As a Different Object" section later in this chapter
</div>

After the SerializationInfo object is constructed and initialized, the formatter calls the type's GetObjectData method, passing it the reference to the SerializationInfo object. The GetObjectData method is responsible for determining what information is necessary to serialize the object and adding this information to the SerializationInfo object. GetObjectData indicates what information to serialize by calling one of the many overloaded AddValue methods provided by the SerializationInfo type. AddValue is called once for each piece of data that you want to add.

The following code shows an approximation of how the `Dictionary<TKey, TValue>` type implements the ISerializable and IDeserializationCallback interfaces to take control over the serialization and deserialization of its objects:
```C#
[Serializable]
public class Dictionary<TKey, TValue>: ISerializable, IDeserializationCallback {
   // Private fields go here (not shown)
   private SerializationInfo m_siInfo; // Only used for deserialization

   // Special constructor (required by ISerializable) to control deserialization
   [SecurityPermissionAttribute(SecurityAction.Demand, SerializationFormatter = true)] 
   protected Dictionary(SerializationInfo info, StreamingContext context) {
      // During deserialization, save the SerializationInfo for OnDeserialization
      m_siInfo = info;
   }

   // Method to control serialization
   [SecurityCritical]
   public virtual void GetObjectData(SerializationInfo info, StreamingContext context) {
      info.AddValue("Version", m_version);
      info.AddValue("Comparer", m_comparer, typeof(IEqualityComparer<TKey>));
      info.AddValue("HashSize", (m_ buckets == null) ? 0 : m_buckets.Length);
      if (m_buckets != null) {
         KeyValuePair<TKey, TValue>[] array = new KeyValuePair<TKey, TValue>[Count];
         CopyTo(array, 0);
         info.AddValue("KeyValuePairs", array, typeof(KeyValuePair<TKey, TValue>[]));
      }
   }

   // Method called after all key/value objects have been deserialized
   public virtual void IDeserializationCallback.OnDeserialization(Object sender) {
      if (m_siInfo == null) return; // Never set, return

      Int32 num = m_siInfo.GetInt32("Version");
      Int32 num2 = m_siInfo.GetInt32("HashSize");
      m_comparer = (IEqualityComparer<TKey>)m_siInfo.GetValue("Comparer", typeof(IEqualityComparer<TKey>));
      if (num2 != 0) {
         m_buckets = new Int32[num2];
      for (Int32 i = 0; i < m_buckets.Length; i++) m_buckets[i] = -1;
        ... // code to re-construct the internal state of the Dictionary object
   }
}
```
 The serialization process is

 1. When you pass the Dictionary object to the formatter, formatter sees it implements `ISerializable`, so formatters know that Dictionary wants to have complete control on serialization/deserialization, so the formatter creates an instance of `SerializationInfo`,  this instance contains the type information of Dictionary, then the foramtter calls the Dictionary object's GetObjectData method, passing SerializationInfo instance, which makes SerializationInfo instance adds(saves) the values that are essential to reconstruct the internal state of the deserialized Dictionary object later.

 2. The formatter serializes the SerializationInfo object (not the Dictionary object because Dictionary implements `ISerializable`) into the stream, this SerializationInfo object's FullTypeName and AssemblyName are set to `Dictionary<TKey, TValue>` related.


 The derialization process is:

 3. The formatter constructs an object of `SerializationInfo` based on the stream data. So this SerializationInfo objects contains all values that Dictionary needs (originally added by `AddValue` method in `GetObjectData` method). 

 4. The formatter calls `FormatterServices.GetUninitializedObject(typeof(Dictionary<TKey,TValue>));` to create an object of Dictionary, then the formatter calls the special constructor of Dictionary and passes SerializationInfo objects.

 5. IDeserializationCallback.OnDeserialization only calls when deserialization of the entire object graph has been completed, so the Dictionary object can reconstruct its internal state using the SerializationInfo objects 

 Note that it looks like more straightforward to reconstruct the Dictionary object's internal state at step 4 inside the special constructor, after all, we already have the SerializationInfo object which is the essential thing we need, so why not get the job done in the special constructor? The reason is, the dictionary needs to re-calculate the hash code of each entry , and it can't do this until each entry has been properly deserialized. Since the order of deserialization is not guaranteed, the dictionary needs to wait until the entire graph has been reconstructed before calculating any of those hash codes, check the following code:
 ```C#
public class Program
{
    public static void Main()
    {
        var container = new Container() { Name = "Test" };
        container.Dict.Add(container, "Container");
        
        var formatter = new BinaryFormatter();
        var stream = new MemoryStream();
        formatter.Serialize(stream, container);
        
        stream.Position = 0;
        formatter.Deserialize(stream);
    }
}

[Serializable]
public class Container : ISerializable
{
    public string Name { get; set; }
    public MyDictionary Dict { get; }
    
    public Container()
    {
        Dict = new MyDictionary();
    }
    
    protected Container(SerializationInfo info, StreamingContext context)
    {
        Console.WriteLine("Container deserialized");

        Name = info.GetString("Name");
        Dict = (MyDictionary)info.GetValue("Dict", typeof(MyDictionary));
    }
    
    public virtual void GetObjectData(SerializationInfo info, StreamingContext context)
    {
        info.AddValue("Name", Name);
        info.AddValue("Dict", Dict);
    }
    
    public override bool Equals(object other) => (other as Container)?.Name == Name;
    public override int GetHashCode() => Name.GetHashCode();
}

[Serializable]
public class MyDictionary : Dictionary<object, object>
{
    public MyDictionary() { }

    protected MyDictionary(SerializationInfo info, StreamingContext context)
        : base(info, context)
    {
        Console.WriteLine("MyDictionary deserialized");
        
        // Look at the data which Dictionary wrote...
        var kvps = (KeyValuePair<object, object>[])info.GetValue("KeyValuePairs", typeof(KeyValuePair<object, object>[]));
        Console.WriteLine("Name is: " + ((Container)kvps[0].Key).Name);
    }
}
 ```

This prints:
```
MyDictionary deserialized
Name is:
Container deserialized
```

Notice how MyDictionary is deserialized first, before Container is deserialized, despite the fact that MyDictionary contains Container as one of its keys! You can also see that Container.Name is null when MyDictionary is deserialized -- that only gets assigned to later on, some time before OnDeserialization is called. Since Container.Name is null when the MyDictionary is constructed, and Container.GetHashCode depends on Container.Name, then obviously incorrect things would happen if Dictionary tried to call GetHashCode on any of its items' keys at that point.


## Serializing a Type As a Different Type and Deserializing an Object As a Different Object

The .NET Framework’s serialization infrastructure is quite rich, and in this section, we discuss how a developer can design a type that can serialize or deserialize itself into a different type or object. Let's look at the code below:
```C#
static void Main(string[] args) {
   using (var stream = new MemoryStream()) {
      BinaryFormatter formatter = new BinaryFormatter();
      MainType mainType = new MainType();
      formatter.Serialize(stream, mainType);
      stream.Position = 0;
      var deserializedSubType = formatter.Deserialize(stream);   // deserializedSubType is SubType now
   }

   Console.ReadLine();
}

[Serializable]
public class MainType : ISerializable
{
   public string name = "MainType";
   public DateTime Date = DateTime.Now;

   public MainType() {
   }

   void ISerializable.GetObjectData(SerializationInfo info, StreamingContext context) {
      // Using the SetType method is equivalent to setting both the FullTypeName and the AssemblyName.
      // MainType doesn't need the special constructor any more
      info.SetType(typeof(SubType));  // <---------------------------------the core of first method                                 
   }

   // NOTE: The special constructor is NOT necessary because it's never called (because we use SetType method)
}

[Serializable]
public class SubType
{
   public string name = "SubType";
   public DateTime Date = DateTime.Now;

}
```
Second method:
```C#
static void Main(string[] args) {
  ...  // same
}

[Serializable]
public class MainType : ISerializable
{
   public string name = "MainType";
   public DateTime Date = DateTime.Now;

   public MainType() {
   }

   void ISerializable.GetObjectData(SerializationInfo info, StreamingContext context) {
      info.SetType(typeof(MainTypeHelper));   // <------------------------the core of second method 
   }
}

[Serializable]
public sealed class MainTypeHelper : IObjectReference
{
   public object GetRealObject(StreamingContext context) {   // doesn't need SerializationInfo object because there is no specail constructor
      return new SubType();
   }
   // Doesn't have a special constructor
}

// Now it doesn't need [Serializable]
public class SubType
{
   public string name = "SubType";
   public DateTime Date = DateTime.Now;

}
```
Let's analyze the differences between first method and second method in details, which also involve how formatters deal with `SerializationInfo` object:

Firstly, let's revisit how formatters serialize/deserialize a type that implements `ISerializable`:

When formatters try to serialize the MainType instance, formatters see the MainType implements `ISerializable`, formatters construct
a `SerializationInfo` object, setting both the `FullTypeName` and the `AssemblyName` to match type of MainType . Then formatters pass this `SerializationInfo` object to MainType's `GetObjectData` method (of `ISerializable` interface), then MainType can instruct how itself to be seriailized (e.g. calling `AddValue` method). Then formatters serialize this `SerializationInfo` object that contain need-to-serialize type (MainType) and assembly information. (Note that SerializationInfo doesn't have `[Serializable]` or implement `ISerializable`, but it is still serializable, probably because SerializationInfo is unique to formatters, it is implemention details anyway). **Note that it is the SerializationInfo object that will be serilized into the stream, not the MainType object that will be serilized**, if MainType only uses `[Serializable]` and doesn't implement `ISerializable`, then the MainType object will be serilized. 

When formatters deserilize from the stream, formatters read the "SerializationInfo" data (FullTypeName, AssemblyName etc, and data added by AddValue method, formatters will construct a SerializationInfo object, because this object will be passed to the special constructor).

Once formatters know they need to deserilize MainType, then it will call `FormatterServices.GetTypeFromAssembly(Assembly assem, String name)` and then `FormatterServices.GetUninitializedObject(Type type)` and followed by calling MainType's special constructor (MainType doen't have this constructor even though it does implement ISerializable, will explain in details).

Now let's see `SetType` method and `IObjectReference` make the serialization/deserialization process different.

**When calling `SetType` method instead of `AddValue` method, MainType doesn't need the special constructor any more, even though it does implement ISerializable.** The reason it doesn't need special constructor is because now you instruct the formatter to serialize another type (SubType) in lieu of MainType, so it is up to SubType, if SubType implements ISerializable (it doesn't in this case), then SubType does need a special constructor.

Let's see how first method differs from second method:

In the first method, `info.SetType(typeof(SubType));` makes formatters serialize `SubType` instead of `MainType`, `FullTypeName` and `AssemblyName` of `SerializationInfo` is not related to MainType any more, it is related to `SubType` now. So the SerializationInfo object (contains SubType info) will be serilized into the stream.

In the second method, `info.SetType(typeof(MainTypeHelper));` makes formatters serialize `MainTypeHelper` instead of `MainType`, `FullTypeName` and `AssemblyName` of `SerializationInfo` is related to MainTypeHelper (that's why MainTypeHelper needs to have `[Serializable]`). So the SerializationInfo object (contains MainTypeHelper info) will be serilized into the stream. After constructing the MainTypeHelper object (calling FormatterServices methods), the formatter sees that this type implements `IObjectReference`, formatters won't call any specail constructor (in fact, MainTypeHelper doesn't have a special constructor), formatters call the GetRealObject method. This method returns a reference to the object that you really want a reference to now that deserialization of the object has completed.

Now you should see the biggest difference between first method and second method. Both methods will return an instance of `SubType` eventually, but how this instance is "constructed" is different. In the first method, SubType instance is constructed by formatters by calling `FormatterServices`'s related methods. While in the second method, formatters call MainTypeHelper's GetRealObject method, SubType instance is new up there and its reference returned. `FormatterServices` is not involved at all.

Let's look at another concrete example that shows how to properly serialize and deserialize a singleton type. The Singleton class represents a type that allows only one instance of itself to exist per AppDomain. The following code tests the Singleton’s serialization and deserialization code to ensure that only one instance of the Singleton type ever exists in the AppDomain
```C#
static void Main(string[] args) {
   // Create an array with multiple elements referring to the one Singleton object
   Singleton[] a1 = { Singleton.GetSingleton(), Singleton.GetSingleton() };
   Console.WriteLine("Do both elements refer to the same object? " + (a1[0] == a1[1])); // "True"

   using (var stream = new MemoryStream()) {
      BinaryFormatter formatter = new BinaryFormatter();

      // Serialize and then deserialize the array elements
      formatter.Serialize(stream, a1);
      stream.Position = 0;
      Singleton[] a2 = (Singleton[])formatter.Deserialize(stream);

      // Prove that it worked as expected:
      Console.WriteLine("Do both elements refer to the same object? " + (a2[0] == a2[1])); // "True"
      Console.WriteLine("Do all elements refer to the same object? " + (a1[0] == a2[0])); // "True"
   }

   Console.ReadLine();
}

[Serializable]
public sealed class Singleton : ISerializable {
   // This is the one instance of this type
   private static readonly Singleton s_theOneObject = new Singleton();

   // Here are the instance fields
   public String Name = "Jeff";
   public DateTime Date = DateTime.Now;

   private Singleton() {
   }

   // Method returning a reference to the singleton
   public static Singleton GetSingleton() {
      return s_theOneObject;
   }

   // Method called when serializing a Singleton
   // I recommend using an Explicit Interface Method Implemention. 
   [SecurityPermission(SecurityAction.Demand, SerializationFormatter = true)]
   void ISerializable.GetObjectData(SerializationInfo info, StreamingContext context) {
      info.SetType(typeof(SingletonSerializationHelper));
      // No other values need to be added
   }

   [Serializable]
   private sealed class SingletonSerializationHelper : IObjectReference {
      // Method called after this obejct (which has no fields) is deserialized
      public object GetRealObject(StreamingContext context) {
         return Singleton.GetSingleton();
      }
   }
   // NOTE: The special constructor is NOT necessary because it's never called
}
```

## Serialization Surrogates, Surrogate Selector Chains and Binder

To work in the future, not very practical



<!-- <code>&lt;T&gt;<code> -->

<!-- <div class="alert alert-info p-1" role="alert">
    
</div> -->

<!-- <div class="alert alert-info pt-2 pb-0" role="alert">
    <ul class="pl-1">
      <li></li>
      <li></li>
    </ul>  
</div> -->

<!-- <ul>
  <li></li>
  <li></li>
  <li></li>
  <li></li>
</ul>  -->

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