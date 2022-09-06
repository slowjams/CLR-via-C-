Chapter 21-The Managed Heap and Garbage Collection -Rework
==============================


## Essential Background from CSAPP

A dynamic memory allocator maintains an area of a process's virtual memory known as the ***heap*** as picture below shows:

![alt text](./zImages/21-1.png "Title")

Details vary from system to system, but without loss of generality, we will assume that the heap is an area of demand-zero memory that begins immediately after the uninitialized data area and grows upward (toward higher addresses). For each process, the kernel maintains a variable brk that points to the top of the heap.

An allocator maintains the heap as a collection of various-size blocks. Each block is a contiguous chunk of virtual memory that is either *allocated* or *free*.

The `malloc` function returns a pointer to a block of memory of at least size bytes:

![alt text](./zImages/21-2.png "Title")

#### Implicit Free Lists

Any practical allocator needs some data structure that allows it to distinguish block boundaries and to distinguish between allocated and free blocks. Most
allocators embed this information in the blocks themselves. One simple approach is shown below:

![alt text](./zImages/21-3.png "Title")

In this case, a block consists of a one-word header, the payload, and possibly some additional padding. The header encodes the block size (including the header
and any padding) as well as whether the block is allocated or free. If we impose a double-word alignment constraint, then the block size is always a multiple of 8 and the 3 low-order bits of the block size are always zero. Thus, we need to store only the 29 high-order bits of the block size, freeing the remaining 3 bits to encode other information. 

![alt text](./zImages/21-4.png "Title")

#### Placing Allocated Blocks

When an application requests a block of k bytes, the allocator searches the free list for a free block that is large enough to hold the requested block. The manner in which the allocator performs this search is determined by the placement policy. Some common policies are **first fit**, **next fit**, and **best fit**. 

First fit searches the free list from the beginning and chooses the first free block that fits. Next fit is similar to first fit, but instead of starting each search at the beginning of the list, it starts each search where the previous search left off. Best fit examines every free block and chooses the free block with the smallest size that fits.

#### Splitting Free Blocks

Once the allocator has located a free block that fits, it must make another policy decision about how much of the free block to allocate. The allocator will usually opt to split the free block into two parts. The first part becomes the allocated block, and the remainder becomes a new free block. Figure 9.37 shows how the allocator might split the eight-word free block in Figure 9.36 to satisfy an application’s request for three words of heap memory.

![alt text](./zImages/21-5.png "Title")

#### Coalescing Free Blocks

When the allocator frees an allocated block, there might be other free blocks that are adjacent to the newly freed block. Such adjacent free blocks can cause
a phenomenon known as false fragmentation, where there is a lot of available free memory chopped up into small, unusable free blocks. For example, Figure 9.38
shows the result of freeing the block that was allocated in Figure 9.37. 

![alt text](./zImages/21-6.png "Title")

The result is two adjacent free blocks with payloads of three words each. As a result, a subsequent request for a payload of four words would fail, even though the aggregate size of the two free blocks is large enough to satisfy the request.

To combat false fragmentation, any practical allocator must merge adjacent free blocks in a process known as ***coalescing***. This raises an important policy
decision about when to perform coalescing. The allocator can opt for:
<ul>
  <li>immediate coalescing by merging any adjacent blocks each time a block is freed.</li>
  <li>deferred coalescing by waiting to coalesce free blocks at some later time.</li>
</ul> 

Immediate coalescing is straightforward and can be performed in constant time, but with some request patterns it can introduce a form of thrashing where a
block is repeatedly coalesced and then split soon thereafter. For example, in Figure 9.38, a repeated pattern of allocating and freeing a three-word block would introduce a lot of unnecessary splitting and coalescing. In our discussion of allocators, we will assume immediate coalescing, but you should be aware that fast allocators often opt for some form of deferred coalescing.

#### Coalescing with Boundary Tags

Let us refer to the block we want to free as the current block. Then coalescing the next free block (in memory) is straightforward and efficient. The header of the current block points to the header of the next block, which can be checked to determine if the next block is free. If so, its size is simply added to the size of the current header and the blocks are coalesced in constant time.

But how would we coalesce the previous block? Given an implicit free list of blocks with headers, the only option would be to search the entire list, remembering the location of the previous block, until we reached the current block. With an implicit free list, this means that each call to free would require time linear in the size of the heap. Even with more sophisticated free list organizations, the search time would not be constant.

Knuth developed a clever and general technique, known as boundary tags, that allows for constant-time coalescing of the previous block. The idea, which is
shown in Figure 9.39, is to add a footer (the boundary tag) at the end of each block, where the footer is a replica of the header:

![alt text](./zImages/21-7.png "Title")

If each block includes such a footer, then the allocator can determine the starting location and status of the previous block by inspecting its footer, which is always one word away from the start of the current block.

Figure 9.40 shows how we would coalesce each of the four cases:

![alt text](./zImages/21-8.png "Title")

#### Explicit Free Lists

The implicit free list provides us with a simple way to introduce some basic allocator concepts. However, because block allocation time is linear in the total
number of heap blocks, **the implicit free list is not appropriate for a general purpose allocator** (although it might be fine for a special-purpose allocator where the number of heap blocks is known beforehand to be small).

A better approach is to organize the free blocks into some form of explicit data structure. Since by definition the body of a free block is not needed, the pointers that implement the data structure can be stored within the bodies of the free blocks. For example, the heap can be organized as a doubly
linked free list by including a pred (predecessor) and succ (successor) pointer in each free block, as shown in Figure 9.48:

![alt text](./zImages/21-9.png "Title")

Using a doubly linked list instead of an implicit free list reduces the first-fit allocation time from linear in the total number of blocks to linear in the number of free blocks. However, the time to free a block can be either linear or constant, depending on the policy we choose for ordering the blocks in the free list.

One approach is to maintain the list in last-in first-out (LIFO) order by inserting newly freed blocks at the beginning of the list. With a LIFO ordering
and a first-fit placement policy, the allocator inspects the most recently used blocks first. In this case, freeing a block can be performed in constant time.
If boundary tags are used, then coalescing can also be performed in constant time.

Another approach is to maintain the list in address order, where the address of each block in the list is less than the address of its successor. In this case, freeing a block requires a linear-time search to locate the appropriate predecessor. The trade-off is that address-ordered first fit enjoys better memory utilization than LIFO-ordered first fit, approaching the utilization of best fit.

A disadvantage of explicit lists in general is that free blocks must be large enough to contain all of the necessary pointers, as well as the header and possibly a footer. This results in a larger minimum block size and increases the potential for internal fragmentation.

#### Segregated Free Lists (segregated fits)

With explicit free lists, when you want to find a specific size of free block, but you have to start with the first free block to the next etc, and it might takes a lot of time to find the free block that matches the size you require. A popular approach for reducing the allocation time, known generally as **segregated storage**, is to maintain multiple free lists, where each list holds blocks that are roughly the same size. The general idea is to partition the set of all possible block sizes into equivalence classes called size classes. There are many ways to define the size classes. For example, we might partition the block sizes by powers of 2:
```
{1}, {2}, {3, 4}, {5–8}, ... , {1,025–2,048}, {2,049–4,096}, {4,097–∞}
```
![alt text](./zImages/21-10.png "Title")

To allocate a block, we determine the size class of the request and do a firstfit search of the appropriate free list for a block that fits. If we find one, then we (optionally) split it and insert the fragment in the appropriate free list. If we cannot find a block that fits, then we search the free list for the next larger size class. To free a block, we coalesce and place the result on the appropriate free list.

The segregated fits approach is a popular choice with production-quality allocators such as the GNU malloc package provided in the C standard library
because it is both fast and memory efficient.

#### Garbage Collector Basics

A garbage collector views memory as a directed reachability graph of the form shown in Figure 9.49:

![alt text](./zImages/21-11.png "Title")

The role of a garbage collector is to maintain some representation of the reachability graph and periodically reclaim the unreachable nodes by freeing them and returning them to the free list.

Garbage collectors for languages like ML and Java, which exert tight control over how applications create and use pointers, can maintain an exact representation of the reachability graph and thus can reclaim all garbage. However, collectors for languages like C and C++ cannot in general maintain exact representations of the reachability graph. Such collectors are known as **conservative garbage collectors**. They are conservative in the sense that each reachable block is correctly identified as reachable, while some unreachable nodes might be incorrectly identified as reachable.

Collectors can provide their service on demand, or they can run as separate threads in parallel with the application, continuously updating the reachability
graph and reclaiming garbage. For example, consider how we might incorporate a conservative collector for C programs into an existing malloc package, as shown in Figure 9.50:

![alt text](./zImages/21-12.png "Title")

The application calls malloc in the usual manner whenever it needs heap space. If malloc is unable to find a free block that fits, then it calls the garbage collector in hopes of reclaiming some garbage to the free list. The collector identifies the garbage blocks and returns them to the heap by calling the free function. The key idea is that the collector calls free instead of the application. When the call to the collector returns, malloc tries again to find a free block that fits. If that fails, then it can ask the operating system for additional memory.

#### Mark&Sweep Garbage Collectors

A Mark&Sweep garbage collector consists of a mark phase, which marks all reachable and allocated descendants of the root nodes, followed by a sweep phase,
which frees each unmarked allocated block. Typically, one of the spare low-order bits in the block header is used to indicate whether a block is marked or not.

![alt text](./zImages/21-13.png "Title")

The mark phase calls the mark function shown in Figure 9.51(a) once for each root node. The mark function returns immediately if p does not point to
an allocated and unmarked heap block. Otherwise, it marks the block and calls itself recursively on each word in block. Each call to the mark function marks any unmarked and reachable descendants of some root node. At the end of the mark phase, any allocated block that is not marked is guaranteed to be unreachable and, hence, garbage that can be reclaimed in the sweep phase.

The sweep phase is a single call to the sweep function shown in Figure 9.51(b). The sweep function iterates over each block in the heap, freeing any unmarked
allocated blocks (i.e., garbage) that it encounters.

Figure 9.52 shows a graphical interpretation of Mark&Sweep for a small heap. Block boundaries are indicated by heavy lines. Each square corresponds to a
word of memory. Each block has a one-word header, which is either marked or unmarked:

![alt text](./zImages/21-14.png "Title")

Initially, the heap in Figure 9.52 consists of six allocated blocks, each of which is unmarked. Block 3 contains a pointer to block 1. Block 4 contains pointers to blocks 3 and 6. The root points to block 4. After the mark phase, blocks 1, 3, 4, and 6 are marked because they are reachable from the root. Blocks 2 and 5 are unmarked because they are unreachable. After the sweep phase, the two unreachable blocks are reclaimed to the free list.

#### Conservative Mark&Sweep for C Programs

Mark&Sweep is an appropriate approach for garbage collecting C programs because it works in place without moving any blocks. However, the C language poses
some interesting challenges for the implementation of the isPtr function.

First, C does not tag memory locations with any type information. Thus, there is no obvious way for isPtr to determine if its input parameter p is a pointer or not. Second, even if we were to know that p was a pointer, there would be no obvious way for isPtr to determine whether p points to some location in the payload of an allocated block. For example, an `int*` array that each element points to element of another `int` array in heap, so you don't know where does the containing block starting address when you only get the address that points to its payload.

One solution to the latter problem is to maintain the set of allocated blocks as a balanced binary tree that maintains the invariant that all blocks in the left subtree are located at smaller addresses and all blocks in the right subtree are located in larger addresses. As shown in Figure 9.53, this requires two additional fields (left and right) in the header of each allocated block. Each field points to the header of some allocated block. The isPtr(ptr p) function uses the tree to perform a binary search of the allocated blocks. At each step, it relies on the size field in the block header to determine if p falls within the extent of the block.

![alt text](./zImages/21-15.png "Title")

The balanced tree approach is correct in the sense that it is guaranteed to mark all of the nodes that are reachable from the roots. This is a necessary guarantee, as application users would certainly not appreciate having their allocated blocks prematurely returned to the free list. However, it is conservative in the sense that it may incorrectly mark blocks that are actually unreachable, and thus it may fail to free some garbage. While this does not affect the correctness of application programs, it can result in unnecessary external fragmentation.

The fundamental reason that Mark&Sweep collectors for C programs must be conservative is that the C language does not tag memory locations with type
information. Thus, scalars like ints or floats can masquerade as pointers. For example, suppose that some reachable allocated block contains an int in its
payload whose value happens to correspond to an address in the payload of some other allocated block b. There is no way for the collector to infer that the data is really an int and not a pointer. Therefore, the allocator must conservatively mark block b as reachable, when in fact it might not be.

## Allocating Resources from the Managed Heap 

The CLR requries that all objects be allocated from the ***managed heap***. When a process is initialized, the CLR allocates a region of address space for the managed heap. The CLR also maintains a pointer, which I'll call `NextObjPtr`. This pointer indicates where the next object is to be allocated within the heap. Initially, `NextObjPtr` is set to the base address of the address space region.

As region fills with non-garbage objects, the CLR allocates more regions and continues to do this until the whole process’s address space is full. So, your application’s memory is limited by the process’s virtual address space. In a 32-bit process, you can allocate close to 1.5 gigabytes (GB) and in a 64-bit process, you can allocate close to 8 terabytes.

C#'s new operator causes the CLR to perform the following steps:

<ol>
  <li>Calculate the number of bytes requried for the type's fields (and all the fields it inherits from its base types).</li>
  <li>Add the bytes required for an object's overhead. Each object has two overhead fields: a type object pointer and a sync block index. For a 32-bit application, each of these fields requires 32 bits, adding 8 bytes to each object. For a 64-bit application, each field is 64 bits, adding 16
bytes to each object.</li>
  <li>The CLR then checks that the bytes required to allocate the object are available in the region. If there is enough free space in the managed heap, the object will fit, starting at the address pointed to by NextObjPtr, and these bytes are zeroed out. The type's constructor is called (passing `NextObjPtr` for the `this` parameter), and the new operator returns a reference to the object. Just before the reference is returned, NextObjPtr is advanced past the object and now points to the address where the next object will be placed in the heap.</li>
</ol> 

Figure 21-1 shows a managed heap consisting of three objects: A, B, and C. If another object were to be allocated, it would be placed where NextObjPtr points to (immediately after object C):

![alt text](./zImages/21-16.png "Title")

For the managed heap, allocating an object simply means adding a value to a pointer-this is blazingly fast. In many applications, objects allocated around the same time tend to have strong relationships to each other and are frequently accessed around the same time. For example, it's very common to allocate a FileStream object immediately before a BinaryWrite object is created. Then the application would use the BinaryWrite object, which internally uses the FileStream object. Because the managed heap allocates these objects next to each other in memory, you get excellent performance when accessing these objects due to locality of reference. Specifically, this means that your process's working set is small, which means your application run fast with less memory. It's also likely that the objects your code is accessing can all reside in the CPU's cache. The result is that your application will access these objects with phenomenal speed because the CPU will be able to perform most of its manipulations without haveing cache misses that would force slower access to RAM.

So far, it sounds like the managed heap provides excellent performance characteristics. However, what I have just described is assuming that memory is infinite and that the CLR can always allocate new objects at the end. However, memory is not infinite and so the CLR employs a technique known as garbage collection (GC) to "delete" objects in the heap that your application no longer requires access to.

## The Garbage Collection Algorithm

When an application calls the new operator to create an object, there might not be enough address space left in the region to allocate the object. If insufficient space exists, then the CLR performs a GC

<div class="alert alert-info p-1" role="alert">
    What I've just said is an oversimplification. In reality, a GC occurs when generation 0 is full. I'll explain generations later in this chapter. Until then, it's easiest for you to think that a garbage collection occurs when the heap is full.
</div>

For managingthe lifetime of objects, some systems use a reference couning algorithm. In fact, Microsoft's own Component Object Model (COM) uses reference couning system, each object on the heap maintains an internal field indicating how many "part" of the program are currently using that object. As each "part" gets to a place i the code where it no longer requires access to an object, it decrements that object's count field. When the count field reaches 0, the object deletes itself from memory. The big problem with many reference counting systems is that they do not handle circular reference well. For example, in a GUI application, a window will hold a reference to a child UI element. And the child UI element will hold a reference to its parent window. These references prevent the two object's counters from reaching 0, so both objects will never be deleted even if the application itself no longer has a need for the window.

Due tp this problem with reference counting garbage collector algorithms, the CLR uses a referenceing tracking algorithm instead. The reference tracking algorithm cares only about reference type variables, because only these variables can refer to an object on the heap; value type variables contain the value type instance directly. Reference type varialbes can be used in many contexts: static and instance fields within a class or a method's arguments or local variable. We refer to all reference type variable as ***roots***.

When the CLR starts a GC, the CLR first suspends all threads in the process. This prevents threads from accessing objects and changing their state while the CLR examines them. Then, the CLR performs what is called *marking phase* of the GC. First, it walks through all the objects in the heap setting a bit (contained in the sync block index field) to 0. This indicates that all objects should be deleted. Then, the CLR looks at all active roots to see which objects they refer to. This is what makes the CLR's GC a reference tracking GC. If a root contains null, the CLR ignores the root and moves on to examine the next root.

Any root referring to an object on the heap causes the CLR to mark that object. Marking an object means that the CLR sets the bit in the object’s sync block index to 1. When an object is marked, the CLR examines the roots inside that object and marks the objects they refer to. If the CLR is about to mark an already-marked object, then it does not examine the object’s fields again. This prevents an infinite loop from occurring in the case where you have a circular reference.

Figure 21-2 shows a heap containing several objects:

![alt text](./zImages/21-17.png "Title")

In this example, the application roots refer directly to objects A, C, D, and F. All of these objects are marked. When marking object D, the garbage collector notices that this object contains a field that refers to object H, causing object H to be marked as well. The marking phase continues until all the application roots have been examined.

Once complete, the heap contains some marked and some unmarked objects. The marked objects must survive the collection because there is at least one root that refers to the object; we say that the object is reachable because application code can reach (or access) the object by way of the variable that still refers to it. Unmarked objects are unreachable because there is no root existing in the application that would allow for the object to ever be accessed again.

Now that the CLR knows which objects must survive and which objects can be deleted, it begins the GC's compacting phase. This serves many benefits. First, all the surviving objects will be next to each other in memory; this restores locality of reference reducing your application’s working set size, thereby improving the performance of accessing these objects in the future. Second, the free space is all contiguous as well, so this region of address space can be freed, allowing other things to use it. Finally, compaction means that there are no address space fragmentation issues with the managed heap as is known to happen with native heaps. (Objects in the large object heap (discussed later in this chapter) do not get compacted, and therefore address space fragmentation is possible with the large object heap)

When compacting memory, the CLR is moving objects around in memory. This is a problem because any root that referred to a surviving object now refers to where that object was in memory; not where the object has been relocated to. When the application’s threads eventually get resumed, they would access the old memory locations and corrupt memory. Clearly, this can't be allowed and so, as part of the compacting phase, the CLR subtracts from each root the number of bytes that the object it referred to was shifted down in memory. This ensures that every root refers to the same object it did before; it's just that the object is at a different location in memory.

After the heap memory is compacted, the managed heap's NextObjPtr pointer is set to point to a location just after the last surviving object. This is where the next allocated object will be placed in memory. Figure 21-3 shows the managed heap after the compaction phase: 

![alt text](./zImages/21-18.png "Title")

After the compaction phase is complete, the CLR resumes all the application's threads and they continue to access the objects as if the GC never happened at all.

If the CLR is unable to reclaim any memory after a GC and if there is no address space left in the processes to allocate a new GC segment, then there is just no more memory available for this process. In this case, the new operator that attempted to allocate more memory ends up throwing an `OutOfMemoryException`. Your application can catch this and recover from it but most applications do not attempt to do so; instead, the exception becomes an unhandled exception, Windows terminates the process, and then Windows reclaims all the memory that the process was using.

<div class="alert alert-info p-1" role="alert">
     A static field keeps whatever object it refers to forever or until the AppDomain that the types are loaded into is unloaded. A common way to leak memory is to have a static field refer to a collection object and then to keep adding items to the collection object. The static field keeps the collection object alive and the collection object keeps all its items alive. For this reason, it is best to avoid static fields whenever possible.
</div>

## Garbage Collections and Debugging

As soon as a root goes out of scope, the object it refers to is unreachable and subject to having its memory reclaimed by a GC; objects aren't guaranteed to live throughout a method’s lifetime. This can have an interesting impact on your application. For example, examine the following code:

```C#
// run the program in release build in VS and press ctrl + F5
class Program {
   static void Main(string[] args) {
      // Create a Timer object that knows to call our TimerCallback method once every 2000 milliseconds.
      Timer t = new Timer(TimerCallback, null, 0, 1000);
      // Wait for the user to hit <Enter>. 
      Console.ReadLine();
   }

   static void TimerCallback(Object o) {
      // Display the date/time when this method got called.
      Console.WriteLine("In TimerCallback: " + DateTime.Now);
      // Force a garbage collection to occur for this demo.
      GC.Collect();
   }
}
```
Compile this code from the command prompt without using any special compiler switches. When you run the resulting executable file, you'll see that the TimerCallback method is called just once!

<div class="alert alert-info p-1" role="alert">
    If you do see TimerCallback method is called every 2000 milliseconds, that's because you run it in the debug mode, which is the same as using C# compiler's /debug switch which will be covered next
</div>

From examining the preceding code, you’d think that the TimerCallback method would get called once every 2,000 milliseconds. After all, a Timer object is created, and the variable t refers to this object. As long as the timer object exists, the timer should keep firing. But you'll notice in the
TimerCallback method that I force a garbage collection to occur by calling GC.Collect().

When the collection starts, it first assumes that all objects in the heap are unreachable (garbage); this includes the Timer object. Then, the collector examines the application's roots and sees that Main doesn't use the t variable after the initial assignment to it. Therefore, the application has no
variable referring to the Timer object, and the garbage collection reclaims the memory for it; this stops the timer and explains why the TimerCallback method is called just once.

This post describes how JIT compiler supports this feature in details [Understanding garbage collection in .NET](https://stackoverflow.com/a/17131389/9623401):

*The JIT compiler performs two important duties when it compiles the IL for a method into machine code. The first one is very visible in the debugger, you can see the machine code with the Debug + Windows + Disassembly window. The second duty is however completely invisible. It also generates a table that describes how the local variables inside the method body are used. That table has an entry for each method argument and local variable with two addresses. The address where the variable will first store an object reference. And the address of the machine code instruction where that variable is no longer used. Also whether that variable is stored on the stack frame or a cpu register.`*

*The "no longer used" address in the table is very important. It makes the garbage collector very efficient. It can collect an object reference, even if it is used inside a method and that method hasn't finished executing yet.*

When you compile your assembly by using the C# compiler's /debug switch, the compiler applies a System.Diagnostics.DebuggableAttribute with its DebuggingModes' DisableOptimizations flag set into the resulting assembly. At run time, when compiling a method, the JIT compiler sees this flag set, and artificially extends the lifetime of all roots to the end of the method. For my example, the JIT compiler tricks itself into believing that the t variable in Main must live until the end of the method. So, if a garbage collection were to occur, the garbage collector now thinks that t is still a root and that the Timer object that t refers to will continue to be reachable. The Timer object will survive the collection, and the TimerCallback method will get called repeatedly until Console. ReadLine returns and Main exits.

<div class="alert alert-info p-1" role="alert">
    Note that in debug mode, the JIT Optimization(that doesn't extend object's life time) is disabled, but this optimization will always enabled in release build, so if you run the code above in release build, TimerCallback method is still called just once.
</div>

So in the release build, you might say we can add assign a null to the variable so GC will think it is still reachable:

```C#
class Program {
   static void Main(string[] args) {
      Timer t = new Timer(TimerCallback, null, 0, 1000);
      Console.ReadLine();
      // Refer to t after ReadLine (this gets optimized away) 
      t = null;
   }
   ...
}
```
However, if you compile this (without the /debug+ switch) and run the resulting executable file, you'll see that the TimerCallback method is still called just once.
The problem here is that the JIT compiler is an optimizing compiler, and setting a local variable or parameter variable to null is the
same as not referencing the variable at all. In other words, the JIT compiler optimizes the t = null; line out of the code completely, and therefore, the program still does not work as we desire. The correct way to modify the Main method is as follows:

```C#
class Program {
   static void Main(string[] args) {
      Timer t = new Timer(TimerCallback, null, 0, 1000);
      Console.ReadLine();

      GC.KeepAlive(t);
   }
   ...
}
```
Now TimerCallback method is called multiple times, and the program is fixed. `GC.KeepAlive()` is implemented as:
```C#
[MethodImpl(MethodImplOptions.NoInlining)]
public static void KeepAlive(object obj)
{
}
```
You can see the method body is empty. The purpose of `GC.KeepAlive(object obj)` is just to keep reference so obj won't be collected by GC prematurely.

## Generations: Improving Performance

The CLR's GC is a generational garbage collector (also known as an ephemeral garbage collector, although I don’t use the latter term in this book). A generational GC makes the following assumptions about your code:

<ul>
  <li>The newer an object is, the shorter its lifetime will be.</li>
  <li>The older an object is, the longer its lifetime will be.</li>
  <li>Collecting a portion of the heap is faster than collecting the whole heap.</li>
</ul> 

Numerous studies have demonstrated the validity of these assumptions for a very large set of existing applications, and these assumptions have influenced how the garbage collector is implemented. In this section, I'll describe how generations work.

When initialized, the managed heap contains no objects. Objects added to the heap are said to be in generation 0. Stated simply, objects in generation 0 are newly constructed objects that the garbage collector has never examined. Figure 21-4 shows a newly started application with five objects allocated (A through E). After a while, objects C and E become unreachable:

![alt text](./zImages/21-19.png "Title")

When the CLR initializes, it selects a budget size (in kilobytes) for generation 0. So if allocating a new object causes generation 0 to surpass its budget, a garbage collection must start. Let's say that objects A through E fill all of generation 0. When object F is allocated, a garbage collection must start. The garbage collector will determine that objects C and E are garbage and will compact object D, causing it to be adjacent to object B. The objects that survive the garbage collection (objects A, B, and D) are said to be in generation 1. Objects in generation 1 have been examined by the garbage collector once. The heap now looks like Figure 21-5:

![alt text](./zImages/21-20.png "Title")

After a garbage collection, generation 0 contains no objects. As always, new objects will be allocated in generation 0. Figure 21-6 shows the application running and allocating objects F through K. In addition, while the application was running, objects B, H, and J became unreachable and should have their memory reclaimed at some point.

![alt text](./zImages/21-21.png "Title")

Now let's say that attempting to allocate object L would put generation 0 over its budget. Because generation 0 has reached its budget, a garbage collection must start. When starting a garbage collection, the garbage collector must decide which generations to examine. Earlier, I said that when the CLR initializes, it selects a budget for generation 0. Well, it also selects a budget for generation 1.

When starting a garbage collection, the garbage collector also sees how much memory is occupied by generation 1. In this case, generation 1 occupies much less than its budget, so the garbage collector examines only the objects in generation 0. Look again at the assumptions that the generational garbage collector makes. The first assumption is that newly created objects have a short life time. So generation 0 is likely to have a lot of garbage in it, and collecting generation 0 will therefore reclaim a lot of memory. The garbage collector will just ignore the objects in generation 1, which will speed up the garbage collection process.

Obviously, ignoring the objects in generation 1 improves the performance of the garbage collector. However, the garbage collector improves performance more because it doesn't traverse every object in the managed heap. If a root or an object refers to an object in an old generation, the garbage collector can ignore any of the older objects' inner references, decreasing the amount of time required to build the graph of reachable objects. Of course, it's possible that an object's field refers to a new object, then if GC doesn't check this object and therefore doesn't check this object's member fields, then the new objects will be marked as unreachable, reclaimed by GC. To ensure that the updated fields of these old objects are examined, the garbage collector uses a mechanism internal to the JIT compiler that sets a bit when an object’s reference field changes. This support lets the garbage collector know which old objects (if any) have been written to because the last collection. Only old objects that have fields change need to be examined to see whether they refer to any new object in generation 0.

Microsoft's performance tests show that it takes less than 0.001 second to perform a garbage collection of generation 0. Microsoft's goal is to have garbage collections take no more time than an ordinary page fault.

A generational garbage collector also assumes that objects that have lived a long time will continue to live. So it's likely that the objects in generation 1 will continue to be reachable from the application. Therefore, if the garbage collector were to examine the objects in generation 1, it probably wouldn't find a lot of garbage. As a result, it wouldn't be able to reclaim much memory. So it is likely that collecting generation 1 is a waste of time. If any garbage happens to be in generation 1, it just stays there. The heap now looks like Figure 21-7:

![alt text](./zImages/21-22.png "Title")

As you can see, all of the generation 0 objects that survived the collection are now part of generation 1. Because the garbage collector didn't examine generation 1, object B didn't have its memory reclaimed even though it was unreachable (it is just we know it's unreachable but GC doesn't know since it doesn't try to mark generation 1 objects, the picture might be a little bit confusing because it makes you think that GC has alread marked it but it is not, it is just marked by author to show users) at the time of the last garbage collection. Again, after a collection, generation 0 contains no objects and is where new objects will be placed. 

In fact, let's say that the application continues running and allocates objects L through O. And while running, the application stops using objects G, L, and M, making them all unreachable (again, B and G are marked by author as reachable in advance, not GC marked them). The heap now looks like Figure 21-8:

![alt text](./zImages/21-23.png "Title")

Let's say that allocating object P causes generation 0 to exceed its budget, causing a garbage collection to occur. Because the memory occupied by all of the objects in generation 1 is less than its budget, the garbage collector again decides to collect only generation 0, ignoring the unreachable objects in generation 1 (objects B and G). After the collection, the heap looks like Figure 21-9:

![alt text](./zImages/21-24.png "Title")

In Figure 21-9, you see that generation 1 keeps growing slowly. In fact, let's say that generation 1 has now grown to the point in which all of the objects in it occupy its full budget. At this point, the application continues running (because a garbage collection just finished) and starts allocating objects P through S, which fill generation 0 up to its budget. The heap now looks like Figure 21-10:

![alt text](./zImages/21-25.png "Title")

When the application attempts to allocate object T, generation 0 is full, and a garbage collection must start. This time, however, the garbage collector sees that the objects in generation 1 are occupying so much memory that generation 1's budget has been reached. Over the several generation 0 collections, it's likely that a number of objects in generation 1 have become unreachable (as in our example). So this time, the garbage collector decides to examine all of the objects in generation 1 and generation 0. After both generations have been garbage collected, the heap now looks like Figure 21-11:

![alt text](./zImages/21-26.png "Title")

As before, any objects that were in generation 0 that survived the garbage collection are now in generation 1; any objects that were in generation 1 that survived the collection are now in generation 2. **As always, generation 0 is empty immediately after a garbage collection** and is where new objects will be allocated. Objects in generation 2 are objects that the garbage collector has examined two or more times. There might have been several collections, but the objects in generation 1 are examined only when generation 1 reaches its budget, which usually requires several garbage collections of generation 0.

The managed heap supports only three generations: generation 0, generation 1, and generation 2; there is no generation 3.3 **When the CLR initializes, it selects budgets for all three generations**, the budget for generation 0 is about 256 KB, and the budget for generation 1 is about 2 MB. However, the CLR’s garbage collector is a self-tuning collector. This means that the garbage collector learns about your application’s behavior whenever it performs a garbage collection. For example, if your application constructs a lot of objects and uses them for a very short period of time, it's possible that garbage collecting generation 0 will reclaim a lot of memory. In fact, it's possible that the memory for all objects in generation 0 can be reclaimed.

If the garbage collector sees that there are very few surviving objects after collecting generation 0, it might decide to reduce the budget of generation 0.  This reduction in the allotted space will mean that garbage collections occur more frequently but will require less work for the garbage collector, so your process's working set will be small. In fact, if all objects in generation 0 are garbage, a garbage collection doesn't have to compact any memory; it can simply set NextObjPtr back to the beginning of generation 0, and then the garbage collection is performed. Wow, this is a fast way to reclaim memory!

On the other hand, if the garbage collector collects generation 0 and sees that there are a lot of surviving objects, not a lot of memory was reclaimed in the garbage collection. In this case, the garbage collector will grow generation 0's budget. Now, fewer collections will occur, but when they do, a lot more memory should be reclaimed. By the way, if insufficient memory has been reclaimed after a collection, the garbage collector will perform a full collection before throwing an OutOfMemoryException.

Throughout this discussion, I've been talking about how the garbage collector dynamically modifies generation 0’s budget after every collection. But the garbage collector also modifies the budgets of generation 1 and generation 2 by using similar heuristics. When these generations are garbage collected, the garbage collector again sees how much memory is reclaimed and how many objects survived. Based on the garbage collector’s findings, it might grow or shrink the thresholds of these generations as well to improve the overall performance of the application. The end result is that the garbage collector fine-tunes itself automatically based on the memory load required by your application—this is very cool!

## Garbage Collection Triggers

As you know, the CLR triggers a GC when it detects that generation 0 has filled its budget. This is the most common trigger of a GC; however, there are additional GC triggers as listed here:

<ul>
  <li><b>Code explicitly calls GC.Collect() method</b> Code can explicitly request that the CLR perform a collection. Although Microsoft strongly discourages such requests, at times it might make sense for an application to force a collection. I discuss this more in the "Forcing Garbage Collections" section later in this chapter.</li>
  <li><b>Windows is reporting low memory conditions</b> The CLR internally uses the Win32 CreateMemoryResourceNotification and QueryMemoryResourceNotification functions to monitor system memory overall. If Windows reports low memory, the CLR will force a garbage collection in an effort to free up dead objects to reduce the size of a process's working set.</li>
  <li><b>The CLR is unloading an AppDomain</b> When an AppDomain unloads, the CLR considers nothing in the AppDomain to be a root, and a garbage collection consisting of all generations is performed. I'll discuss AppDomains in Chapter 22, "CLR Hosting and AppDomains."</li>
  <li><b>The CLR is shutting down</b> The CLR shuts down when a process terminates normally (as opposed to an external shutdown via Task Manager, for example . During this shutdown, the CLR considers nothing in the process to be a root; it allows objects a chance to clean up but the CLR does not attempt to compact or free memory because the whole process is terminating, and Windows will reclaim all of the processes' memory.</li>
</ul> 

## Large Objects

There is one more performance improvement you might want to be aware of. The CLR considers each single object to be either a small object or a large object. So far, in this chapter, I've been focusing on small objects. Today, a large object is 85KB or more in size.4 The CLR treats large objects slightly differently than how it treats small objects:

<ul>
  <li>Large objects are not allocated within the same address space as samll objects; they are allocated elsewhere within the process's address space</li>
  <li>GC doesn't compact large objects because of the time it would require to move them in memory. For this reason, address space fragmentation can occur between large objects within the process leading to an OutOfMemoryException being thrown. In a future version of the CLR, large objects may participate in compaction.</li>
  <li>Large objects are immediately considered to be part of generation 2; they are never in generation 0 or 1. So, you should create large objects only for resources that you need to keep alive for a long time.  =Allocating short-lived large objects will cause generation 2 to be collected more frequently, hurting performance. Usually large objects are large strings (like XML or JSON) or byte arrays that you use for I/O operations, such as reading bytes from a file or network into a buffer so you can process it.</li>
</ul> 

For the most part, large objects are transparent to you; you can simply ignore that they exist and that they get special treatment until you run into some unexplained situation in your program (like why you're getting address space fragmentation).

## Garbage Collection Modes

When the CLR starts, it selects a GC mode, and this mode cannot change during the lifetime of the process. There are two basic GC modes:

<ul>
  <li><b>Workstation</b> This mode fine-tunes the garbage collector for client-side application. It is optimized to provide for low-latency GCs in order to minimize the time an application's threads are susptended so as not to frustrate the end user. In this mode, the GC assumes that other applications are running on the machine and doesn't hog CPU resource.</li>
  <li><b>Server</b> This mode fine-tunes the garbage collector for server-side applications. It is optimized for throughput and resource utilization. In this mode, the GC assumes no other applications are running on the machine, and it assumes that all the CPUs on the machine are available to assist with completing the GC. This GC mode causes the managed heap to be spilt into serveral sections, one per CPU. When a garbage collection is initiated, the garbage collector dedicates one special thread per CPU; each thread collects its own section in parallel with other threads. Parallel collections work well for server applications in which the worker threads tend to exhibit uniform behavior. This feature requires the application to be running on a computer with multiple CPUs so that the threads can truly be working simulataneously to attain a performance improvement.</li>
</ul> 

By default, applications run with the Workstation GC mode. A server application (such as ASP.NET or Microsoft SQL Server) that hosts the CLR can request the CLR to load the Server GC. However, if the server application is running on a uniprocessor machine, then the CLR will always use Workstation GC mode. A stand-alone application can tell the CLR to use the Server GC mode by creating a configuration file (as discussed in Chapter 2 and Chapter 3) that contains a
gcServer element for the application. Here's an example of a configuration file:
```
<configuration>
   <runtime>
      <gcServer enabled="true"/>
   </runtime>
</configuration>
```

When an application is running, it can ask the CLR if it is running in the Server GC mode by querying the GCSettings class's IsServerGC read-only Boolean property:

```C#
public static class Program {
   public static void Main() {
      Console.WriteLine("Application is running with server GC=" + GCSettings.IsServerGC());
   }
}
```

In addition to the two modes. the GC can run in two sub-modes: concurrent (the default) or non-concurrent. In concurrent mode, the GC has an additional background thread that marks objects concurrently while the application runs. When a thread allocates an object that pushes generation 0 over its budget, the GC first suspends all threads and then determines which generations to collect. If the garbage collector needs to collect generation 0 or 1, it proceeds as normal. However, if generation 2 needs collecting, the size of generation 0 will be increased beyond its budget to allocate the new object, and then the application's thread are resumed.

While the application's threads are running, the garbage collector has a normal priority background thread that finds unreachable objects. Once found, the garbage collector suspends all threads again and decides whether to compact memory. If the garbage collector decides to compact memory, memory is compacted, root references are fixed up, and the application;s threads are resumed. This garbage collection takes less time than usual because the set of unreachable objects has already been built. However, the garbage collector might decide not to compact memory; in fact, the garbage collector favors this approach. If you have a lot of free memory, the garbage collector won't compact the heap; this improves performance but grows your application’s working set. When
using the concurrent garbage collector, you’ll typically find that your application is consuming more memory than it would with the non-concurrent garbage collector.

You can tell the CLR not to use the concurrent collector by creating a configuration file for the application that contains a gcConcurrent element. Here's an example of a configuration file:
```
<configuration>
   <runtime>
      <<gcConcurrent enabled="false"/>
   </runtime>
</configuration>
```

## Forcing Garbage Collections

The System.GC type allows your application some direct control over the garbage collector. For starters, you can query the maximum generation supported by the managed heap by reading the GC.MaxGeneration property; this property always returns 2.

You can also force the garbage collector to perform a collection by calling GC class's Collect method, optionally passing in a generation to collect up to, a GCCollectionMode, and a Boolean indicating whether you want to perform a blocking (non-current) or background (concurrent) collection. Here is the signature of the most complex overload of the Collect method:

```C#
void Collect(Int32 generation, GCCollectionMode mode, Boolean blocking);
```
Normally you will just call `GC.Collect()` in most of scenario. Under most circumstances, you should avoid calling any of the Collect methods.

## Working with Types Requiring Special Cleanup

Fortunately for us, most types need only memory to operate. However, some types require more than just memory to be useful; some types require the use of a native resource in addition to memory.

The `System.IO.FileStream` type, for example, needs to open a file (a native resource) and store the file's handle. Then the type's Read and Write methods use this handle to manipulate the file. Similarly, the `System.Threading.Mutex` type opens a Windows mutex kernel object (a native resource) and stores its handle, using it when the Mutex's methods are called. 

If a type wrapping a native resource gets GC'd, the GC will reclaim the memory used by the object in the managed heap; but the native resource, which the GC doesn't know anything about, will be leaked. This is clearly not desirable, so the CLR offers a mechanism called ***finalization***. Finalization allows an object to execute some code after the object has been determined to be garbage but before the object's memory is reclaimed from the managed heap. All types that wrap a narive resource-such as a file, network connection, socket, or mutex-support finalization. When the CLR determines that one of these objects is no longer reachable, the object gets to finalize itself, releasing the native resource it wraps, and then, later, the GC will reclaim the object from the managed heap.

`System.Object`, the base class of everything, defines a protected and virtual method called Finalize. When the garbage collector determines that an object is garbage, it calls the object’s Finalize method (if it is overridden). Microsoft’s C# team felt that Finalize methods were a special kind of method requiring special syntax in the programming language (similar to how C# requires special syntax to define a constructor). So, in C#, you must define a Finalize method by placing a tilde symbol (~) in front of the class name, as shown in the following code sample:

```C#
internal sealed class SomeType {
   // This is the Finalize method
   ~SomeType() {
     // The code here is inside the Finalize method
   }  
}
```
If you were to compile this code and examine the resulting assembly with ILDasm.exe, you’d see that the C# compiler did, in fact, emit a protected override method named Finalize into the module's metadata. If you examined the Finalize method's IL code, you'd also see that the code inside the method's body is emitted into a try block, and that a call to base.Finalize is emitted into a finally block.

Finalize methods are called at the completion of a garbage collection on objects that the GC has determined to be garbage. This means that the memory for these objects cannot be reclaimed right away because the Finalize method might execute code that accesses a field. Because a finalizable object must survive the collection, it gets promoted to another generation, forcing the object to live much longer than it should. This is not ideal in terms of memory consumption and is why you should avoid finalization when possible. To make matters worse, when finalizable objects get promoted, any object referred to by its fields also get promoted because they must continue to live too. So, try to avoid defining finalizable objects with reference type fields.

Furthermore, be aware of the fact that you have no control over when the Finalize method will execute. Finalize methods run when a garbage collection occurs, which may happen when your application requests more memory. Also, the CLR doesn't make any guarantees as to the order in which Finalize methods are called. So, **you should avoid writing a Finalize method that accesses other objects whose type defines a Finalize method**; ([Stackoverflow Questions](https://stackoverflow.com/questions/67093620/why-it-is-ok-to-a-type-to-access-a-reference-member-in-the-types-finalize-but-n?noredirect=1#comment118601135_67093620))However, it is perfectly OK to access value type instances or reference type objects that do not define a Finalize method. You also need to be careful when calling static methods because these methods can internally access objects that have been finalized, causing the behavior of the static method to become unpredictable.

<div class="alert alert-info p-1" role="alert">
    I think the reason why you should avoid writing a Finalize method that accesses other objects whose type defines a Finalize method is, you cannot control 
    which one's finalizer will be called first, which will result in consequences indicated by the code below, it has nothing to do with the fact whether or not its reference instance member exists in the heap. (Still need to be confirmed)
</div>

```C#
class ClassA  {
  public ClassB Item;
  public ClassA(ClassB item) { this.Item = item };
  ~ClassA {
     item.Write("ClassA Finalize called");
  }
}

// ClassB is a mock of FileStream  
public class ClassB: Stream {   // 
    private byte[] _buffer;
    private SafeFileHandle _handle;
    ...

    public ClassB(...) {
       ...
        _handle = Win32Native.CreateFile(...);
    }

    public void Write(String s) {
       //use Win32Native.WriteFile(...) API to write s to the file wrapped by SafeFileHandle
    }
    ...
    ~ClassB() {
       if (_handle != null) {
          Dispose(false);
       }
    }

    protected override void Dispose(bool disposing) {
       // Flush data from the buffer to disk
       // then call _handle.Dispose(), which close the file
    }
}
```
when `~ClassA` tries to write sth to the file, the file might have been already closed by `~ClassB`, which can cause serious consequence(e.g throw an exception). it has nothing to do with the fact whether or not its ClassB instance member exists in the heap, ClassB instance member must exist in the heap when `~ClassA` executes regardless of whether ClassB has Finalize or not. Note that FileStream can access its SafeFileHandle member in its finalizer because `SafeFileHandle` inherit `CriticalFinalizerObject`, which allows CLR has control to call `~FileStream` before calling `SafeFileHandle`. In later section "An Interesting Dependency Issue", the author provides StreamWrite example that addresses the same problem, then you will see ClassA should not have finalizer.

The CLR uses a special, high-priority dedicated thread to call Finalize methods to avoid some deadlock scenarios that could occur otherwise. If a Finalize method blocks (for example, enters an infinite loop or waits for an object that is never signaled), this special thread can’t call any more Finalize methods. This is a very bad situation because the application will never be able to reclaim the memory occupied by the finalizable objects—the application will leak memory as long as it runs. If a Finalize method throws an unhandled exception, then the process terminates; there is no way to catch this exception.

## Finalization Internals

When an application creates a new object, the new operator allocates the memory from the heap. If the object's type defines a Finalize method, a pointer to the object is placed on the finalization list just before the type’s instance constructor is called. The finalization list is an internal data structure controlled by the garbage collector. Each entry in the list points to an object that should have its Finalize method called before the object’s memory can be reclaimed. Figure 21-13 shows a heap containing several objects:

![alt text](./zImages/21-27.png "Title")

Some of these objects are reachable from application roots, and some are not. When objects C, E, F, I, and J were created, the system detected that these objects' types defined a Finalize method and so added references to these objects to the finalization list.

<div class="alert alert-info p-1" role="alert">
    Even though System.Object defines a Finalize method, the CLR knows to ignore it; that is, when constructing an instance of a type, if the type’s Finalize method is the one inherited from System.Object, the object isn't considered finalizable. One of the derived types must override Object's Finalize method.
</div>

<div class="alert alert-info p-1" role="alert">
    Note that when a new instance (its type class has Finalize) is created, the reference is then added to the Finalization list, so it is there before GC occurs, the reference is not considered as a root of the object in heap, otherwise GC will never be able to mark it as garbage, that's probably why inn the picture, the author used dotted line from Finalization list to object while using solid line from Freachable queue to object, becuase references in Freachable queue are considered roots to the objects in the heap, which prevents GC collect them prematurely
</div>

When a garbage collection occurs, objects B, E, G, H, I, and J are determined to be garbage. The garbage collector scans the finalization list looking for references to these objects. When a reference is found, the reference is removed from the finalization list and appended to the ***freachable queue***. The
freachable queue (pronounced "F-reachable") is another of the garbage collector’s internal data structures. Each reference in the freachable queue identifies an object that is ready to have its Finalize method called. After the collection, the managed heap looks like Figure 21-14:

![alt text](./zImages/21-28.png "Title")

In this figure, you see that the memory occupied by objects B, G, and H has been reclaimed because these objects didn't have a Finalize method. However, the memory occupied by objects E, I, and J couldn’t be reclaimed because their Finalize methods haven’t been called yet.

A special high-priority CLR thread is dedicated to calling Finalize methods. A dedicated thread is used to avoid potential thread synchronization situations that could arise if one of the application’s normal-priority threads were used instead. When the freachable queue is empty (the usual case), this
thread sleeps. But when entries appear, this thread wakes, removes each entry from the queue, and then calls each object’s Finalize method.

The interaction between the finalization list and the freachable queue is fascinating. First, I'll tell you how the freachable queue got its name. Well, the "f" is obvious and stands for finalization; every entry in the freachable queue is a reference to an object in the managed heap that should have its Finalize method called. But the reachable part of the name means that the objects are reachable. To put it another way, the freachable queue is considered a root, just as static fields are roots. So **a reference in the freachable queue keeps the object it refers to reachable and is not garbage**.

In short, when an object isn’t reachable, the garbage collector considers the object to be garbage. Then when the garbage collector moves an object's reference from the finalization list to the freachable queue, the object is no longer considered garbage and its memory can't be reclaimed. When an
object is garbage and then not garbage, we say that the object has been ***resurrected***.

The next time the garbage collector is invoked on the older generation, it will see that the finalized objects are truly garbage because the application’s roots don't point to it and the freachable queue no longer points to it either. The memory for the object is simply reclaimed. The important point to
get from all of this is that two garbage collections are required to reclaim memory used by objects that require finalization. In reality, **more than two collections will be necessary because the objects get promoted to another generation, and not every GC will examine old generation, so you should avoid defining a type that has Finalize method as the objects will live in the heap too long**. Figure 21-15 shows what the managed heap looks like after the
second garbage collection:

![alt text](./zImages/21-29.png "Title")

## Implement a Dispose method

Classes that allow the consumer to control the lifetime of native resources it wraps implement the
IDisposable interface, which looks like this:
```C#
public interface IDisposable {
   void Dispose();
}
```
<div class="alert alert-info p-1" role="alert">
    If a class defines a field in which the field's type implements the dispose pattern, the class itself should also implement the dispose pattern. The Dispose method should dispose of the object referred to by the field. This allows someone using the class to call Dispose on it, which in turn releases the resources used by the object itself, so it is like a Dispose calling chain.
</div>

#### Dispose() and Dispose(bool)

The IDisposable interface requires the implementation of a single parameterless method, Dispose. Also, any non-sealed class should have an additional Dispose(bool) overload method to be implemented:

<ul>
  <li>A public non-virtual IDisposable.Dispose implementation that has no parameters.</li>
  <li>A protected virtual Dispose method whose signature is:</li>
</ul> 

```C#
protected virtual void Dispose(bool disposing)
{
}
```
<div class="alert alert-info p-1" role="alert">
   The disposing parameter should be false when called from a finalizer, and true when called from the IDisposable.Dispose method. In other words, it is true when deterministically called and false when non-deterministically called. 
</div>

#### The Dispose() method

Because the public, non-virtual, parameterless Dispose method is called by a consumer of the type, its purpose is to free unmanaged resources, perform general cleanup, and to indicate that the finalizer, if one is present, doesn't have to run. Freeing the actual memory associated with a managed object is always the domain of the garbage collector. Because of this, it has a standard implementation:
```C#
public void Dispose()
{
   // Dispose of unmanaged resources.
   Dispose(true);
   // Suppress finalization.
   GC.SuppressFinalize(this);
}
```
The Dispose method performs all object cleanup, so the garbage collector no longer needs to call the objects' Object.Finalize override. Therefore, the call to the SuppressFinalize method prevents the garbage collector from running the finalizer. If the type has no finalizer, the call to GC.SuppressFinalize has no effect. Note that the actual cleanup is performed by the `Dispose(bool)` method overload.

<div class="alert alert-info p-1" role="alert">
   The Object class provides no implementation for the Finalize method, and the garbage collector does not mark types derived from Object for finalization unless they override the Finalize method.
</div>

#### The Dispose(bool) method overload

In the overload, the disposing parameter is a Boolean that indicates whether the method call comes from a Dispose method (its value is true) or from a finalizer (its value is false).

The body of the method consists of two blocks of code:

<ul>
  <li>A block that frees unmanaged resources. This block executes regardless of the value of the disposing parameter.</li>
  <li>A conditional block that frees managed resources. This block executes if the value of disposing is true. The managed resources that it frees can include:<ul>
  <li><b>Managed objects that implement IDisposable.</b> The conditional block can be used to call their Dispose method (cascade dispose). If you have used a derived class of <code>SafeHandle</code> to wrap your unmanaged resource, you should call the <code>SafeHandle.Dispose()</code></li>
  <li><b>Managed objects that consume large amounts of memory or consume scarce resources.</b> Assign large managed object references to null to make them more likely to be unreachable. This releases them faster than if they were reclaimed non-deterministcally, and this is usually done outside of the conditional block</li>
  </ul> </li>
</ul> 
If the method call comes from a finalizer, only the code that frees unmanaged resources should execute. The implementer is responsible for ensuring that the false path doesn't interact with managed objects that may have been reclaimed. This is important because the order in which the garbage collector destroys managed objects during finalization is non-deterministic.

Here's the general pattern for implementing the dispose pattern for a base class that overrides Object.Finalize:
```C#
class BaseClass : IDisposable
{
    // To detect redundant calls
    private bool _disposed = false;
    //private IDisposable otherManagedObject;

    ~BaseClass() {
       Dispose(false);
    }

    // Public implementation of Dispose pattern callable by consumers.
    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }

    // Protected implementation of Dispose pattern.
    protected virtual void Dispose(bool disposing)
    {
        if (_disposed)
        {
            return;
        }

        if (disposing)
        {
            // TODO: dispose managed state (managed objects).
            // e.g call otherManagedObject.Dispose()
        }

        // TODO: free unmanaged resources (unmanaged objects) and override a finalizer below.
        // TODO: set large fields to null.

        _disposed = true;
    }
}
```
Here's the general pattern for implementing the dispose pattern for a base class that uses a safe handle:
```C#
class BaseClass : IDisposable
{
    // To detect redundant calls
    private bool _disposed = false;

    // Instantiate a SafeHandle instance.
    private SafeHandle _safeHandle = new SafeFileHandle(IntPtr.Zero, true);

    // Public implementation of Dispose pattern callable by consumers.
    public void Dispose() {
       Dispose(true);
    }

    // Protected implementation of Dispose pattern. Derived classes who have own native resource can override this method and calls base.Dispose().
    protected virtual void Dispose(bool disposing)
    {
        if (_disposed)
        {
            return;
        }

        if (disposing)
        {
           // Dispose managed state (managed objects). it uses null check probably to consider _safeHandle can be reassigned to null
            _safeHandle?.Dispose();
        }

        _disposed = true;
    }

    //The SafeHandle class provides a finalizer, so you do not have to write one yourself.
}
```

<div class="alert alert-info p-1" role="alert">
    Dispose method and Finalizer come as a pair at most of cases, if you see a class implements IDisposable, this class should also override finalizer, unless this class uses SafeHandle that users a finalizer internally.
</div>

## (Continued) Working with Types Requiring Special Cleanup

As you can see, there are a lot of caveats related to Finalize methods and they must be used with caution. Specifically, they are designed for releasing native resources. To simplify working with them, it is highly recommended that developers avoid overriding Object's Finalize method; instead, use helper classes that Microsoft now provides in the Framework Class Library (FCL). The helper classes override Finalize and add some special CLR magic I'll talk about as we go on. You will then derive your own classes from the helper classes and inherit the CLR magic.

If you are creating a managed type that wraps a native resource, you should first derive a class from a special base class called `System.Runtime.InteropServices.SafeHandle`, which looks like the following:

```C#
public abstract class SafeHandle : CriticalFinalizerObject, IDisposable {
   // This is the handle to the native resource 
   protected IntPtr handle;
   private bool _ownsHandle;

   protected SafeHandle(IntPtr invalidHandleValue, Boolean ownsHandle) {
      this.handle = invalidHandleValue;
      _ownsHandle = ownsHandle;
      
      // If ownsHandle is true, then the native resource is closed when this SafeHandle-derived object is collected 
      // ownsHandle is probably used for you pass the same native resource to different managed object, 
      //so if one(owns the handle) has already released it, the other shouldn't release it again
      if (!ownsHandle)
         GC.SuppressFinalize(this);
   }
   
   ~SafeHandle() { 
      Dispose(false); 
   }

   protected void SetHandle(IntPtr handle) {
      this.handle = handle;          
   }

   // You can explicitly release the resource by calling Dispose
   // This is the IDisposable interface’s Dispose method

   public void Dispose() { 
      Dispose(true); 
   }
   
   public void Close() {
      Dispose(true);
   }

   protected virtual void Dispose(Boolean disposing) {
       if (disposing)
          InternalDispose();
       else
          InternalFinalize();
      // cannot see the full source code as it involves P/Invoke, below is pesudocomment:
      // If resource already released, return
      // If ownsHandle is false, return
      // Set flag indicating that this resource has been released
      // Call virtual ReleaseHandle method
      // Call GC.SuppressFinalize(this) to prevent Finalize from being called only when calling from Dispose(true) which means you manually call Dispose() from the managed object
      // If ReleaseHandle returned true, return
      // If we get here, fire ReleaseHandleFailed Managed Debugging Assistant (MDA)
   }

   // A derived class overrides this method to implement the code that releases the resource
   protected abstract Boolean ReleaseHandle(); 
   
   public Boolean IsClosed {
      get {
         // Returns flag indicating whether resource was released
      }
   }

   public abstract Boolean IsInvalid {
      // A derived class overrides this property.
      // The implementation should return true if the handle's value doesn't represent a resource (this usually means that the handle is 0 or -1)
      get;
   }

   // These three methods have to do with security and reference counting;
   // I'll talk about them at the end of this section
   public void DangerousAddRef(ref Boolean success) {...}
   public IntPtr DangerousGetHandle() {...}
   public void DangerousRelease() {...}
}

public abstract class SafeHandleZeroOrMinusOneIsInvalid : SafeHandle {
   protected SafeHandleZeroOrMinusOneIsInvalid(Boolean ownsHandle): base(IntPtr.Zero, ownsHandle) {
   }

   public override Boolean IsInvalid {
      get {
         if (base.handle == IntPtr.Zero) 
            return true;
         if (base.handle == (IntPtr) (-1)) 
            return true;
         return false;
      }
   }
}
```
An concrete example:
```C#
class SomeSafeHandle : SafeHandleZeroOrMinusOneIsInvalid
{
    public SomeSafeHandle(IntPtr handle): base(true)
    {
        SetHandle(handle);
    }

    protected override bool ReleaseHandle()
    {
        SomeUnmanagedApi.ReleaseSomething(handle);
        return true;
    }
}

class MyClass : IDisposable
{
    private readonly SomeSafeHandle handle;

    public MyClass()
    {
        handle = new SomeSafeHandle(SomeUnmanagedApi.CreateSomething());
    }

    public void Dispose()
    {
        handle.Dispose();
    }
}
```
MyClass doesn't need a finalizer anymore, since it doesn't own any unmanaged resource. As a consequence, it’s not required to implement the full Dipose pattern.

The first thing to notice about the SafeHandle class is that it is derived from `CriticalFinalizerObject`, which is defined in the `System.Runtime.ConstrainedExecution` namespace. The CLR treats this class and classes derived from it in a very special manner. In particular, the CLR endows this
class with three cool features:

<ul>
  <li>The first time an object of any CriticalFinalizerObject-derived type is constructed, the CLR immediately JIT-compiles all of the Finalize method in the inheritance hierarchy. Compiling these methods upon object construction guarantees that the native resource will be released when the object is determined to be garbage. Without this eager compiling of the Finalize method, it would be possible to allocate the native resource and use it, but not to get rid of it. Under low memory cinditions, the CLR might not be able to find enough memory to compile the Finalize method, which would prevent from executing, causing the native resource to leak. Or the resource might not be freed if the Finalize method contained code that referred to a type in another assembly, and the CLR failed to locate this other assembly.</li>
  <li>The CLR calls the Finalize method of CriticalFinalizerObject-derived type after calling Finalize methods of non-CriticalFinalizerObject-derived type. This ensures that managed resource classes that have a Finalize method can access CriticalFinalizerObject-derived objects within their Finalize methods successfully. For example, the FileStream class's Finalize method can flush data from a memory buffer to an underlying disk with confidence that the disk file has not been closed yet. (the source code and full explanation will be provided shortly) </li>
  <li>The CLR calls the Finalize method of CriticalFinalizerObject-derived types if an AppDomain is rudely aborted by a host application (such as SQL Server or ASP.NET). This also is part of ensuring that the native resource is released even in a case in which a host application no longer trusts the managed code running inside of it.</li>
</ul> 

The second thing to notice about SafeHandle is that the class is abstract; it is expected that another class will be derived from SafeHandle, and this class will provide a constructor that invokes the protected constructor, the abstract method ReleaseHandle, and the abstract IsInvalid property get accessor method.

Most native resources are manipulated with handles (32-bit values on a 32-bit system and 64-bit values on a 64-bit system,So the SafeHandle class defines a protected IntPtr field called handle. In Windows, most handles are invalid if they have a value of 0 or -1. The Microsoft.Win32.SafeHandles namespace contains another helper class called `SafeHandleZeroOrMinusOneIsInvalid`.

Again, you'll notice that the SafeHandleZeroOrMinusOneIsInvalid class is abstract, and therefore, another class must be derived from this one to override the protected constructor and the abstract method ReleaseHandle. The .NET Framework provides just a few public classes derived from SafeHandleZeroOrMinusOneIsInvalid, including SafeFileHandle, SafeRegistryHandle, SafeWaitHandle, and SafeMemoryMappedViewHandle. Here is what the SafeFileHandle class looks like:
```C#
public sealed class SafeFileHandle : SafeHandleZeroOrMinusOneIsInvalid {
   public SafeFileHandle(IntPtr preexistingHandle, Boolean ownsHandle) : base(ownsHandle) {
      base.SetHandle(preexistingHandle);
   }

   protected override Boolean ReleaseHandle() {
      // Tell Windows that we want the native resource closed. 
      return Win32Native.CloseHandle(base.handle); 
   }
}
```

The SafeHandle-derived classes are extremely useful because they ensure that the native resource is freed when a GC occurs. In addition to what we've already discussed, SafeHandle offers two more capabilities. First, the CLR gives SafeHandle-derived types special treatment when used in scenarios in which you are interoperating with native code. For example, let’s examine the following code:
```C#
internal static class SomeType {
   [DllImport("Kernel32", CharSet=CharSet.Unicode, EntryPoint="CreateEvent")]
   // This prototype is not robust
   private static extern IntPtr CreateEventBad(IntPtr pSecurityAttributes, Boolean manualReset, Boolean initialState, String name);

   // This prototype is robust
   [DllImport("Kernel32", CharSet=CharSet.Unicode, EntryPoint="CreateEvent")]
   private static extern SafeWaitHandle CreateEventGood(IntPtr pSecurityAttributes, Boolean manualReset, Boolean initialState, String name);

   public static void SomeMethod() {
      IntPtr handle = CreateEventBad(IntPtr.Zero, false, false, null);
      SafeWaitHandle swh = CreateEventGood(IntPtr.Zero, false, false, null);
   }
}
```
You'll notice that the CreateEventBad method is prototyped as returning an IntPtr, which will return the handle back to managed code; however, interoperating with native code this way is not robust. You see, after CreateEventBad is called (which creates the native event resource), it is possible that a ThreadAbortException could be thrown prior to the handle being assigned to the handle variable. In the rare cases when this would happen, the managed code would leak the native resource. The only way to get the event closed is to terminate the whole process.

The SafeHandle class fixes this potential resource leak. Notice that the CreateEventGood method is prototyped as returning a SafeWaitHandle (instead of an IntPtr). When CreateEventGood is called, the CLR calls the Win32 CreateEvent function. As the CreateEvent function returns to managed code, the CLR knows that SafeWaitHandle is derived from SafeHandle, causing the CLR to automatically construct an instance of the SafeWaitHandle class on the managed heap, passing in the handle value returned from CreateEvent. The constructing of the SafeWaitHandle object and the assignment of the handle happen in native code now, which cannot be interrupted by a ThreadAbortException. Now, it is impossible for managed code to leak this native resource. Eventually, the SafeWaitHandle object will be garbage collected and its Finalize method will be called, ensuring that the resource is released.

One last feature of SafeHandle-derived classes is that they prevent someone from trying to exploit a potential security hole. The problem is that one thread could be trying to use a native resource while another thread tries to free the resource. This could manifest itself as a handle-recycling exploit. 

To understand why using raw IntPtrs are susceptible to handle recycling, you have to be pretty familiar with how thread races can occur. When a program is running, we can use multiple threads simultaneously accessing a single object that encapsulates a handle. One tries to get the managed wrapper class to close the handle, while the other simultaneously initiates an operation on the instance that attempts to use the handle, for example by passing it to a Win32 function. Because IntPtr doesn't do any sort of reference counting, if one thread says it is done with the IntPtr and closes the handle, another thread could come in and load an IntPtr onto its stack just before the call to CloseHandle. Then the thread that already has the IntPtr on its stack would be working with a dangling handle at that point, and the ensuing operation that tried to use it would see unpredictable behavior at best and a security hole at worst.

Consider an example:
```C#
// pseudo code just for demo purpose, doesn't implment IDisposable correctly
// no need to fully understand the code
class MyFile : IDisposable
{
    private IntPtr hFileHandle;
    public MyFile(string filename)
    {
        hFileHandle = OpenFile(filename, out ofStr, OF_READWRITE);
    }

    ~MyFile()

    {
        Dispose(false);
    }

    public void Dispose(bool disposing)
    {
        // if disposing true call GC.SuppressFinalize(this); etc
        if (hFileHandle != IntPtr.Zero)  
            CloseHandle(hFileHandle);
    }

    public int ReadBytes(byte[] buffer)
    {
        // store file handle on stack
        IntPtr hFile = hFileHandle;

        if (hFileHandle == IntPtr.Zero)
            throw new ObjectDisposedException();

        uint read;
        if (!ReadFile(hFile, buffer, buffer.Length, out read, IntPtr.Zero))
            throw new Exception("Error "+ Marshal.GetLastWin32Error());
        return read;
    }

    private const OF_READWRITE = 0x00000002;
    /* P/Invoke signatures for these functions omitted for brevity:
           Kernel32!OpenFile
           Kernel32!ReadFile
           Kernel32!ReadHandle
           Kernel32!CloseHandle
    */
}
```
When somebody is calling Dispose while another thread calls ReadBytes on the same instance. This could lead to:

<ol>
  <li>ReadBytes begins running, loads the value for hFileHandle onto its stack (remember: it's a value type).</li>
  <li>Dispose is scheduled for execution, either preempting ReadBytes (on a uniprocessor) or perhaps running in parallel (on a multiprocessor).</li>
  <li>Dispose executes completely, closing the handle and setting hFileHandle to IntPtr.Zero.</li>
  <li>ReadBytes still has the old value on its stack and passes it to ReadFile! Oops!</li>
</ol> 

The SafeHandle class prevents this security vulnerability by using reference counting. Internally, the SafeHandle class defines a private field that maintains a count. When a SafeHandle-derived object is set to a valid handle, the count is set to 1. Whenever a SafeHandle-derived object is passed as an argument to a native method, the CLR knows to automatically increment the counter. Likewise, when the native method returns to managed code, the CLR knows to decrement the counter. For example, you would prototype the Win32 SetEvent function as follows:
```C#
[DllImport("Kernel32", ExactSpelling=true)]
private static extern Boolean SetEvent(SafeWaitHandle swh);
// you will still need to check whether the native resource wrapped by SafeWaitHandle has been closed/release 
// by calling if SafeWaitHandle.IsClosed() then ReleaseHandle()
// if thread race occurs (i.e. when SafeWaitHandle.IsClosed() return false but then another thread close the native resource)

[DllImport("Kernel32", ExactSpelling=true)]
private static extern Boolean CloseEvent(SafeWaitHandle swh);
// another thread tries to close the native resource,
// but since SafeWaitHandle's ref count is incremented by SetEvent, it cannot release it until SetEvent returns 
```
Now when you call this method passing in a reference to a SafeWaitHandle object, the CLR will increment the counter just before the call and decrement the counter just after the call. Of course, the manipulation of the counter is performed in a thread-safe fashion. How does this improve security? Well, if another thread tries to release the native resource wrapped by the SafeHandle object, the CLR knows that it cannot actually release it because the resource is being used by a native function. When the native function returns, the counter is decremented to 0, and the resource will be released. If you are writing or calling code to manipulate a handle as an IntPtr, you can access it out of a SafeHandle object, but you should manipulate the reference counting explicitly. You accomplish this via SafeHandle's DangerousAddRef and DangerousRelease methods. You gain access to the raw handle via the DangerousGetHandle method.

I would be remiss if I didn't mention that the `System.Runtime.InteropServices` namespace also defines a `CriticalHandle` class. This class works exactly as the SafeHandle class in all ways except that it does not offer the reference-counting feature. The CriticalHandle class and the classes derived from it sacrifice security for better performance when you use it (because counters don't get manipulated). As does SafeHandle, the CriticalHandle class has two types derived from it: `CriticalHandleMinusOneIsInvalid` and `CriticalHandleZeroOrMinusOneIsInvalid`. Because Microsoft favors a more secure system over a faster system, the class library includes no types derived from either of these two classes. For your own work, I would recommend that you use CriticalHandle-derived types only if performance is an issue. If you can justify reducing security, you can switch to a CriticalHandle-derived type.

## Using a Type That Wraps a Native Resource

Now that you know how to define a SafeHandle-derived class that wraps a native resource, let's take a look at how a developer uses it. Let’s start by talking about the common `System.IO.FileStream` class:

```C#
 public class FileStream : Stream {   // Stream implements IDispose
    private byte[] _buffer;
    private SafeFileHandle _handle;
    ...

    public FileStream(...) {
       ...
        _handle = Win32Native.CreateFile(...);
    }
    ...
    ~FileStream() {
       if (_handle != null) {
          Dispose(false);
       }
    }

    protected override void Dispose(bool disposing) {
       // Flush data from the buffer to disk
       // then call _handle.Dispose();
    }
 }
```
The FileStream class offers the ability to open a file, read bytes from the file, write bytes to the file, and close the file. When a FileStream object is constructed, the Win32 CreateFile function is called, the returned handle is saved in a SafeFileHandle object, and a reference to this object is maintained via a private field in the FileStream object. The FileStream class also offers several additional properties (such as Length, Position, CanRead) and methods (such as Read, Write, Flush).

Let's say that you want to write some code that creates a temporary file, writes some bytes to the file, and then deletes the file. You might start writing the code like this:
```C#
public static void Main() {
   // Create the bytes to write to the temporary file. 
   Byte[] bytesToWrite = new Byte[] { 1, 2, 3, 4, 5 };

   // Create the temporary file.
   FileStream fs = new FileStream("Temp.dat", FileMode.Create);

   // Write the bytes to the temporary file.
   fs.Write(bytesToWrite, 0, bytesToWrite.Length);

   // Delete the temporary file.
   File.Delete("Temp.dat"); // Throws an IOException
}
```

Unfortunately, if you build and run this code, it might work, but most likely it won't. The problem is that the call to File's static Delete method requests that Windows delete a file while it is still open. So Delete throws a System.IO.IOException exception with the following string message: 
```
The process cannot access the file "Temp.dat" because it is being used by another process.
```
Be aware that in some cases, the file might actually be deleted! If another thread somehow caused a garbage collection to start after the call to Write and before the call to Delete, the FileStream's SafeFileHandle field would have its Finalize method called, which would close the file and allow Delete to work. The likelihood of this situation is extremely rare, however, and therefore the previous code will fail more than 99 percent of the time.

Fortunately, the FileStream class implements the IDisposable interface (via its base class `Stream`) and its implementation internally calls Dispose on the FileStream object’s private SafeFileHandle field. Now, we can modify our code to explicitly close the file when we want to as opposed to waiting for some GC to happen in the future. Here's the corrected source code.
```C#
public static void Main() {
   Byte[] bytesToWrite = new Byte[] { 1, 2, 3, 4, 5 };

   FileStream fs = new FileStream("Temp.dat", FileMode.Create);

   fs.Write(bytesToWrite, 0, bytesToWrite.Length);

   // Explicitly close the file when finished writing to it.
   fs.Dispose();

   File.Delete("Temp.dat"); // Throws an IOException
}
```
Keep in mind that calling Dispose is not required to guarantee native resource cleanup. Native resource cleanup will always happen eventually; calling Dispose lets you control when that cleanup happens. Also, calling Dispose does not delete the managed object from the managed heap. The only way to reclaim memory in the managed heap is for a garbage collection to kick in. This means **you can still call methods on the managed object even after you dispose of any native resources it may have been using**. 

The following code calls the Write method after the file is closed, attempting to write more bytes to the file. Obviously, the bytes can't be written, and when the code executes, the second call to the Write method throws a `System.ObjectDisposedException` exception with the following string
message: 
```
Cannot access a closed file
```
```C#
public static void Main() {
   Byte[] bytesToWrite = new Byte[] { 1, 2, 3, 4, 5 };

   FileStream fs = new FileStream("Temp.dat", FileMode.Create);

   fs.Write(bytesToWrite, 0, bytesToWrite.Length);

   // Explicitly close the file when finished writing to it.
   fs.Dispose();

   // Try to write to the file after closing it.
   fs.Write(bytesToWrite, 0, bytesToWrite.Length); // Throws ObjectDisposedException

   File.Delete("Temp.dat"); // Throws an IOException
}
```
<div class="alert alert-info p-1" role="alert">
    When defining your own type that implements the IDisposable interface, be sure to write code in all of your methods and properties to throw a <code>System.ObjectDisposedException</code> if the object has been explicitly cleaned up. A Dispose method should never thow an exception; if it's called multiple times, it should just return.
</div>

<div class="alert alert-info p-1" role="alert">
    In general, I strongly discourage explicitly calling Dispose in your code. The reason is that the CLR's garbage collector is well written, and you should let it do its job. The garbage collector knows when an object is no longer accessible from application code, and only then will it collect the object and therefore calls its finalizer. When application code calls Dispose, it is effectively saying that it knows when the application no longer has a need for the object (because the wrapped native resource has been released). For many applications, it is impossible to know for sure when an object is no longer required.
    </br></br>
    For example, if you have code that constructs a new object, and you then pass a reference to this object to another method, the other method could save a reference to the object in some internal field variable (a root). There is no way for the calling method to know that this has happened. Sure, the calling method can call Dispose, but later, some other code might try to access the object, causing an ObjectDisposedException to be thrown. I recommend that you call Dispose only at places in your code where you know you must clean up the resource (as in the case of attempting to delete an open file).
</div>

The previous code examples show how to explicitly call a type's Dispose method. If you decide to call Dispose explicitly, I highly recommend that you place the call in an exception-handling finally block. This way, the cleanup code is guaranteed to execute. So it would be better to write the previous code example as follows:
```C#
public static void Main() {
   Byte[] bytesToWrite = new Byte[] { 1, 2, 3, 4, 5 };

   FileStream fs = new FileStream("Temp.dat", FileMode.Create);

   try {
      fs.Write(bytesToWrite, 0, bytesToWrite.Length);
   }
   finally {
      // Explicitly close the file when finished writing to it.
      if (fs != null) 
         fs.Dispose();
   } 

   File.Delete("Temp.dat"); // Throws an IOException
}
```
Adding the exception-handling code is the right thing to do, and you must have the diligence to do it. Fortunately, the C# language provides a using statement, which offers a simplified syntax that produces code identical to the code just shown. Here's how the preceding code would be rewritten
using C#’s using statement:
```C#
public static void Main() {
   Byte[] bytesToWrite = new Byte[] { 1, 2, 3, 4, 5 };

   using (FileStream fs = new FileStream("Temp.dat", FileMode.Create)) {
      fs.Write(bytesToWrite, 0, bytesToWrite.Length);
   }

   File.Delete("Temp.dat"); // Throws an IOException
}
```
In the using statement, you initialize an object and save its reference in a variable. Then you access the variable via code contained inside using's braces. When you compile this code, the compilerautomatically emits the try and finally blocks. Inside the finally block, the compiler emits code to cast the object to an IDisposable and calls the Dispose method. Obviously, the compiler allows the using statement to be used only with types that implement the IDisposable interface.

## An Interesting Dependency Issue

The System.IO.FileStream type allows the user to open a file for reading and writing. To improve performance, the type's implementation makes use of a memory buffer. Only when the buffer fills does the type flush the contents of the buffer to the file. A FileStream supports the writing of bytes only. If you want to write characters and strings, you can use a System.IO.StreamWriter (FileStream + Encoding), as is demonstrated in the following code:
```C#
FileStream fs = new FileStream("DataFile.dat", FileMode.Create); 
StreamWriter sw = new StreamWriter(fs);
sw.Write("Hi there"); 

// The following call to Dispose is what you should do.
sw.Dispose();   // flush buffer data to FileStream then calls FileStream.Dispose() internally 
// NOTE: StreamWriter.Dispose closes the FileStream;
// the FileStream doesn't have to be explicitly closed.
```
Notice that the StreamWriter's constructor takes a reference to a Stream object as a parameter, allowing a reference to a FileStream object to be passed as an argument. Internally, the StreamWriter object saves the Stream’s reference. When you write to a StreamWriter object, it internally buffers the data in its own memory buffer. When the buffer is full, the StreamWriter object writes the data to the Stream. So there are two buffers involved here.

When you're finished writing data via the StreamWriter object, you should call Dispose. This causes the StreamWriter object to flush its data to the Stream object and close the Stream object.

<div class="alert alert-info p-1" role="alert">
    You don't have to explicitly call Dispose on the FileStream object because the StreamWriter calls it for you. However, if you do call Dispose explicitly, the FileStream will see that the object has already been cleaned up—the method does nothing and just returns.
</div>

What do you think would happen if there were no code to explicitly call Dispose? Well, at some point, the garbage collector would correctly detect that the objects were garbage and finalize them. But the garbage collector doesn't guarantee the order in which objects are finalized. So if the FileStream object were finalized first, it would close the file. Then when the StreamWriter object was finalized, it would attempt to write data to the closed file, throwing an exception. If, on the other hand, the StreamWriter object were finalized first, the data would be safely written to the file.

How was Microsoft to solve this problem? Making the garbage collector finalize objects in a specific order would have been impossible because objects could contain references to each other, and there would be no way for the garbage collector to correctly guess the order in which to finalize these objects. Here is Microsoft's solution:

the StreamWriter type does not support finalization, and therefore it never flushes data in its buffer to the underlying FileStream object. This means that if
you forget to explicitly call Dispose on the StreamWriter object, data is guaranteed to be lost. Microsoft expects developers to see this consistent loss of data and fix the code by inserting an explicit call to Dispose.

<div class="alert alert-info p-1" role="alert">
    The .NET Framework offers a feature called Managed Debugging Assistants (MDAs). When an MDA is enabled, the .NET Framework looks for certain common programmer errors and fires a corresponding MDA. In the debugger, it looks like an exception has been thrown. There is an MDA available to detect when a StreamWriter object is garbage collected without previously having been explicitly disposed. To enable this MDA in Microsoft Visual Studio, open your project and select the Debug.Exceptions menu item. In the Exceptions dialog box, expand the Managed Debugging Assistants node and scroll to the bottom. There you will see the StreamWriterBufferredDataLost MDA. Select the Thrown check box to have the Visual Studio debugger stop whenever a StreamWriter object's data is lost.
</div>

## Other GC Features for Use with Native Resources

Sometimes, a native resource consumes a lot of memory, but the managed object wrapping that resource occupies very little memory. The quintessential example of this is the bitmap. A bitmap can occupy serveral megabytes of native memory, but the managed object is tiny because it contains only an HBITMAP (a 4-byte or 8-byte value). From the CLR's perspective, a process could allocate hundreds of bitmaps (using little managed memory) before performing a collection. But if the process is manipulating many bitmaps, the process's memory consumption will grow at a phenomenal rate. To fix this situation, the GC class offers the following two static methods:

```C#
public static void AddMemoryPressure(Int64 bytesAllocated);
public static void RemoveMemoryPressure(Int64 bytesAllocated);
```

A class that wraps a potentially large native resource should use these methods to give the garbage collector a hint as to how much memory is really being consumed. Internally, the garbage collector monitors this pressure, and when it gets high, a garbage collection is forced.

There are some native resources that are fixed in number. For example, Windows formerly had a restriction that it could create only five device contexts. There had also been a restriction on the number of files that an application could open. Again, from the CLR's perspective, a process could allocate hundreds of objects (that use little memory) before performing a collection. But if the number of these native resources is limited, attempting to use more than are available will typically result in exceptions being thrown. 

To fix this situation, the `System.Runtime.InteropServices` namespace offers the `HandleCollector` class:
```C#
public sealed class HandleCollector {
   public HandleCollector(String name, Int32 initialThreshold);
   public HandleCollector(String name, Int32 initialThreshold, Int32 maximumThreshold);
   public void Add();
   public void Remove();

   public Int32 Count { get; }
   public Int32 InitialThreshold { get; }
   public Int32 MaximumThreshold { get; }
   public String Name { get; }
}
```
A class that wraps a native resource that has a limited quantity available should use an instance class to give garbage collector a hints as to how many instance of the resource are really being consumed. Internally, this class obejct monitors the count, and when it gets high, a garbage collection is forced.

<div class="alert alert-info p-1" role="alert">
    Interanlly, the GC.AddMemoryPressure and HandleCollector.Add methods call GC.Collect, forcing a garbage collection to start is strongly discouraged, because it usually has an adverse effect on your application's performance. However, classes that call these methods are doing so in an effort to keep limited native resources available for the application. If the native resources run out, the application will fail. For most applications, it is better to work with reduced performance than to not be working at all.
</div>

Here is some code that demonstrates the use and effect of the memory pressure methods and the HandleCollector class: 
```C#
class Program {
   static void Main(string[] args) {
      MemoryPressureDemo(0); // 0 causes infrequent GCs
      MemoryPressureDemo(10 * 1024 * 1024); // 10MB causes frequent GCs

      HandleCollectorDemo();

      Console.ReadLine();
   }

   static void MemoryPressureDemo(Int32 size) {
      Console.WriteLine();
      Console.WriteLine("MemoryPressureDemo, size={0}", size);
      // Create a bunch of objects specifying their logical size
      for (Int32 count = 0; count < 10; count++)
         new BigNativeResource(size);

      // For demo purposes, force everything to be cleaned-up
      GC.Collect();
   }

   static void HandleCollectorDemo() {
      Console.WriteLine();
      Console.WriteLine("HandleCollectorDemo");
      for (Int32 count = 0; count < 10; count++)
         new LimitedResource();

      GC.Collect();
   }

   private sealed class BigNativeResource {
      private readonly Int32 m_size;

      public BigNativeResource(Int32 size) {
         m_size = size;
         // Make the GC think the object is physically bigger
         if (m_size > 0) {
            GC.AddMemoryPressure(m_size);
         }
         Console.WriteLine("BigNativeResource create.");
      }

      ~BigNativeResource() {
         // Make the GC think the object released more memory
         if (m_size > 0) {
            GC.RemoveMemoryPressure(m_size);
            Console.WriteLine("BigNativeResource destroy.");
         }
      }
   }

   private sealed class LimitedResource {
      // Create a HandleCollector telling it that collection should
      // occur when two or more of these objects exist in the heap
      private static readonly HandleCollector s_hc = new HandleCollector("LimitedResource", 2);

      public LimitedResource() {
         // Tell the HandleCollector a LimitedResource has been added to the heap
         s_hc.Add();
         Console.WriteLine("LimitedResource create. Count={0}", s_hc.Count);
      }
      ~LimitedResource() {
         // Tell the HandleCollector a LimitedResource has been removed from the heap
         s_hc.Remove();
         Console.WriteLine("LimitedResource destroy. Count={0}", s_hc.Count);
      }
   }
}
```
If you compile and run the preceding code, your output will be similar to the following output:
```
MemoryPressureDemo, size=0
BigNativeResource create.
BigNativeResource create.
BigNativeResource create.
BigNativeResource create.
BigNativeResource create.
BigNativeResource create.
BigNativeResource create.
BigNativeResource create.
BigNativeResource create.
BigNativeResource create.
BigNativeResource create.
BigNativeResource create.
BigNativeResource create.
BigNativeResource create.
BigNativeResource create.
BigNativeResource destroy.
BigNativeResource destroy.
BigNativeResource destroy.
BigNativeResource destroy.
BigNativeResource destroy.
BigNativeResource destroy.
BigNativeResource destroy.
BigNativeResource destroy.
BigNativeResource destroy.
BigNativeResource destroy.
BigNativeResource destroy.
BigNativeResource destroy.
BigNativeResource destroy.
BigNativeResource destroy.
BigNativeResource destroy.

MemoryPressureDemo, size=10485760
BigNativeResource create.
BigNativeResource create.
BigNativeResource create.
BigNativeResource create.
BigNativeResource create.
BigNativeResource create.
BigNativeResource create.
BigNativeResource create.
BigNativeResource create.
BigNativeResource create.
BigNativeResource create.
BigNativeResource create.
BigNativeResource destroy.
BigNativeResource destroy.
BigNativeResource destroy.
BigNativeResource destroy.
BigNativeResource destroy.
BigNativeResource destroy.
BigNativeResource destroy.
BigNativeResource create.
BigNativeResource create.
BigNativeResource create.
BigNativeResource destroy.
BigNativeResource destroy.
BigNativeResource destroy.
BigNativeResource destroy.
BigNativeResource destroy.
BigNativeResource destroy.
BigNativeResource destroy.
BigNativeResource destroy.

HandleCollectorDemo
LimitedResource create. Count=1
LimitedResource create. Count=2
LimitedResource create. Count=3
LimitedResource destroy. Count=3
LimitedResource destroy. Count=2
LimitedResource destroy. Count=1
LimitedResource create. Count=1
LimitedResource create. Count=2
LimitedResource create. Count=3
LimitedResource destroy. Count=2
LimitedResource create. Count=3
LimitedResource destroy. Count=3
LimitedResource destroy. Count=2
LimitedResource destroy. Count=1
LimitedResource create. Count=1
LimitedResource create. Count=2
LimitedResource create. Count=3
LimitedResource destroy. Count=2
LimitedResource destroy. Count=1
LimitedResource destroy. Count=0
```

## Monitoring and Controlling the Lifetime of Objects Manually

The CLR provides each AppDomain with a GC handle table. This table allows an application to monitor the lifetime of an object or manually control the lifetime of an object. When an AppDomain is created, the table is empty. Each entry on the table consists of a reference to an object on the managed heap and a flag indicating how you want to monitor or control the object. An application adds and removes entries from the table via the `System.Runtime.InteropServices.GCHandle` type:
```C#
public struct GCHandle {

   private IntPtr m_handle;    // note that this IntPtr points(as index) to the entry of the handle table
                               // it is not the IntPtr of the managed object that you pass in Alloc

   // Static methods that create an entry in the table
   public static GCHandle Alloc(object value);
   public static GCHandle Alloc(object value, GCHandleType type);

   // Static methods that convert a GCHandle to an IntPtr
   public static explicit operator IntPtr(GCHandle value);
   public static IntPtr ToIntPtr(GCHandle value); 

   // Static methods that convert an IntPtr to a GCHandle
   public static explicit operator GCHandle(IntPtr value);
   public static GCHandle FromIntPtr(IntPtr value);

   // Static methods that compare two GCHandles
   public static Boolean operator ==(GCHandle a, GCHandle b);
   public static Boolean operator !=(GCHandle a, GCHandle b); 

   // Instance method to free the entry in the table (index is set to 0)
   public void Free();

   // Instance property to get/set the entry's object reference
   public object Target { get; set; }

   // Instance property that returns true if index is not 0
   public Boolean IsAllocated { get; }

   // For a pinned entry, this returns the address of the object
   public IntPtr AddOfPinnedObject();

   ...
}
```

Basically, to control or monitor an object's lifetime, you call GCHandle's static Alloc method, passing a reference to the object that you want to monitor/control, and a GCHandleType, which is a flag indicating how you want to monitor/control the object. The GCHandleType type is an enumerated type defined as follows:
```C#
public enum GCHandleType {
   Weak = 0,                    // Used for monitoring an object's existence
   WeakTrackResurrection = 1,   // Used for monitoring an object's existence
   Normal = 2,                  // Used for controlling an object's lifetime 
   Pinned = 3                   // Used for controlling an object's lifetime 
}
```
<ul>
  <li><b>Weak</b> This flag allows you to monitor the lifetime of an object. Specifically, you can detect when the garbage collector has determined this object to be unreachable from application code. Note that the object's Finalize method may or may not have executed yet and therefore, the object may still be in memory. This handle type is used to track an object, but allow it to be collected. When an object is collected, the contents of the GCHandle are zeroed. Weak references are zeroed before the finalizer runs, so even if the finalizer resurrects the object, the Weak reference is still zeroed.</li>
  <li><b>WeakTrackResurrection</b> This flag allows you to monitor the lifetime of an object. Specifically, you can detect when the garbage collector has determined that this object is unreachable from application code. Note that the object's Finalize method (if it exists) has definitely executed, and the object's memory has been reclaimed.This handle type is similar to Weak, but the handle is not zeroed if the object is resurrected during finalization. (another thing might be is, the GC content won't zered out when the object is first marked unreachable, GC content only zero out afer Finalization executes, so GC handle won't be like NotNull(reachable), Null(unreachable), NotNull(in freachable), Null(collected by next GC), it will be NotNull(reachable, unreachable, freachable), NUll(collected by next GC)</li>
  <li><b>Normal</b> This flag allows you to control the lifetime of an object. Specifically, you are telling the garbage collector that this object must remain in memory even though there may be no roots in the application that refers to this object. When a garbage collection runs, the memory for this object can be compacted (moved). The Alloc method that doesn't take a GCHandleType flag assumes that GCHandleType.Normal is specified.</li>
  <li><b>Pinned</b> This flag allows you to control the lifetime of an obejct, Specifically, you are telling the garbage collector that this object must remain in memory even though there might be no roots in the application that refer to this object. When a garbage collection runs, the memory for this object cannot be compacted. This is typically useful when you want to hand the address of the memory out to native code. The native code can write to this memory in the managed heap knowing that a GC will not move the object.</li>
</ul> 

When you call GCHandle's static Alloc method, it scans the AppDomain's GC handle table, looking for an available entry where the reference of the object you passed to Alloc is stored, and a flag is set to whatever you passed for the GCHandleType argument. Then, Alloc returns a GCHandle instance back to you. A GCHandle is a lighweight value type that contains a single instance field, an IntPtr, which refers to the index of the entry in the table. When you want to free this entry in the GC handle table, you take the GCHandle instance and call the Free method, which invaliadates the GCHandle instance by setting its IntPtr field to zero.

Here's how the garbage collector uses the GC handle table. When a garbage collection occurs:

<ol>
  <li>The garbage collector marks all of the reachable objects. Then, the garbage collector scans the GC handle table; all Normal or Pinned objects are considered roots, and these objects are marked as well (including any objects that these objects refer to via their fields</li>
  <li>The garbage collector scans the GC handle table looking for all of the Weak entries. If a Weak entry refers to an object that isn't marked, the reference identifies an unreachable object (garbage), and the entry has its reference value changed to null.</li>
  <li>The garbage collector scans the finalization list. If a reference in the list refers to an unmarked object, the reference identifies an unreachable object, and the reference is moved from the finalization list to the freachable queue. At this point, the object is marked because the object is now considered reachable.</li>
  <li>The garbage collector scans the GC handle table looking for all of the WeakTrackResurrection entries ... (I think author has an errtra here)</li>
  <li>The garbage collector compacts the memory, squeezing out the holes left by the unreachable objects. Pinned objects are not compacted (moved); the garbage collector will move other objects around them.</li>
</ol> 

Now that you have an understanding of the mechanism, let's take a look at when you'd use them. The easiest flags to understand are the Normal and Pinned flags, so let's start with these two. Both of these flags are typically used when interoperating with native code.

The Normal flag is used when you need to hand a pointer to a managed object to native code because, at some point in the future, the native code is going to call back into managed code, passing it the pointer. You can't actually pass a pointer to a managed object out to a native code, because if a garbage collection occurs, the object could move in memory, invalidating the pointer. So to work around this, you would call GCHandle's Alloc method, passing in a reference to the object and the Normal flag. Then you'd cast the returned GCHandle instance to an IntPtr and pass the IntPtr into the native code. When the native code calls back into managed code, the manage code would cast the passed IntPtr back to a GCHandle and then query the Target property to get the reference (or current address) of the managed object. When the native code no longer needs the reference, you'd call GCHandle's Free method, which allows a future garbage collection to free the object.

Below is a concrete example:

```C#
public delegate bool CallBack(int handle, IntPtr param);

internal static class NativeMethods {
   [DllImport("user32.dll")]
   internal static extern bool EnumWindows(CallBack cb, IntPtr param);   
}

public class App {
   public static void Main() {
       Run();
   }

   public static void Run() {
      TextWriter tw = Console.Out;
      GCHandle gch = GCHandle.Alloc(tw);

      CallBack cewp = new CallBack(CaptureEnumWindowsProc);

      // platform invoke will prevent delegate to be garbage collected
      // before call ends

      // instead of pass instance of TextWriter (tw), pass the index of GC handle table,
      // if you pass tw without using GCHandle, tw might be already collected by GC when native code execute
      NativeMethods.EnumWindows(cewp, GCHandle.ToIntPtr(gch))                                                 
      gch.Free();
   }
   
   // native will call this method providing its native handle (created in low-level using C++) and IntPtr recevied from EnumWindows method
   private static bool CaptureEnumWindowsProc(int handle, IntPtr param) {
      GCHandle gch = GCHandle.FromIntPtr(param);   // retrieve the GC handle table entry that associate with TextWriter instance
      TextWriter tw = (TextWriter)gch.Target;      
      tw.WriteLine(handle)
      return true;
   }
}
```
Notice that in this scenario, the native code is not actually using the managed object itself;the native code wants a way just to reference the object. In some scenarios, the native code needs to actually use the managed object. In these scenarios, the managed object must be pinned. Pinning prevents the garbage collector from moving/compacting the object. A common example is when you want to pass a managed String object to a W32 function. In this case, the String object must be pinned because you can't pass the reference of a managed object to native code and then have the garbage collector move the object in memory. If the String object were moved, the native code would either be reading or writing to memory that no longer contained the String object's characters-this will surely cause the application to run unpredictably.

When you use the CLR's P/Invoke mechanism to call a method, the CLR pins the argument for you automatically and unpins them when the native method returns. So, in most cases, you never have to use the GCHandle type to explicitly pin any managed object yourself. You do have to use the GCHandle type explicitly when you need to pass the pointer to a managed object to native code; then the native function returns, but native code might still need to use the object later. Th most common example of this is when performing asynchronous I/O operations.

Let's say that you allocate a byte array that should be filled as data comes in from a socket. Then, you would call GCHandle's Alloc method, passing in a reference to the array object and the Pinned flag. Then, using the returned GCHandle instance, you call the `AddrOfPinnedObject` method. THis returns an IntPtr that is the actual address of the pinned object in the managed heap; you'd then pass this address into the native function, which will return back to managed code immediately. While the data is coming from the socket, this byte array buffer should not move in memory; preventing this buffer from moving is accomplished by using the Pinned flag. When then asynchronous I/O operations has completed, you'd call GCHandle's Free method, which will allow a future garbage collection to move the buffer. You managed code should still have a reference to the buffer so that you can access the data, and this reference will prevent a garbage collection from freeing the buffer from memory completely.

It is also worth mentioning that C# offers a `fixed` statement that effectively pins an object over a block of code. Here is some code that demonstrates its use:

```C#
unsafe public static void Go() {
   // Allocate a bunch of objects that immediately become garbage
   for (Int32 x = 0; x < 10000; x++) 
      new Object();

   IntPtr originalMemoryAddress;
   Byte[] bytes = new Byte[1000]; // Allocate this array after the garbage objects

   // Get the address in memory of the Byte[]
   fixed (Byte* pbytes = bytes) {
       originalMemoryAddress = (IntPtr) pbytes;
   }  

   // Force a collection; the garbage objects will go away & the Byte[] might be compacted
   // because we have allocated 10000 garbages
   GC.Collect();
   
   // Get the address in memory of the Byte[] now & compare it to the first address
   fixed (Byte* pbytes = bytes) {
      Console.WriteLine("The Byte[] did{0} move during the GC", (originalMemoryAddress == (IntPtr) pbytes) ? " not" : null);
   }
}
```
Using C#'s fixed statement is more efficient that allocating a pinned GC handle. What happens is that the C# compiler emits a special "pinned" flag on the pbytes local variable. During a garbage collection, the GC examines the contents of this root, and if the root is not null, it knows not to move the object referred to by the variable during the compaction phase. The C# compiler emits IL to initialize the pbytes local variable to the address of the object at the start of a fixed block, and the compiler emits an IL instruction to set the pbytes local variable back to null at the end of the fixed block so that the variable doesn't refer to any object, allowing the object to move when the next garbage collection occurs.

Now, let's talk about the next two flags, Weak and WeakTrackResurrection. These two flags can be used in scenarios when interoperating with native code, but they can also be used in scenarios that use only managed code. The Weak flag lets you know when an object has been determined to be garbage but the object’s memory is not guaranteed to be reclaimed yet. The WeakTrackResurrection flag lets you know when an object’s memory has been reclaimed. Of the two flags, the Weak flag is much more commonly used than the WeakTrackResurrection flag. In fact, I've never seen anyone use the WeakTrackResurrection flag in a real application. (probably when using WeakTrackResurrection flag and GC contents is not null, the object might be resurrected during finalization, then if you create another reference to refer this object, this object won't be collected in next GC even though its finalizer has executed).

At this point, Object-B can be garbage collected if no other roots are keeping it alive. When Object-A wants to call Object-B’s method, it would query GCHandle’s read-only Target property. If this property returns a non-null value, then Object-B is still alive. Object-A’s code would then cast the returned reference to Object-B’s type and call the method. If the Target property returns null, then Object-B has been collected (but not necessarily finalized) and Object-A would not attempt to call the method. At this point, Object-A’s code would probably also call GCHandless Free method to relinquish the GCHandle instance.

Because working with the GCHandle type can be a bit cumbersome and because it requires elevated security to keep or pin an object in memory, the System namespace includes a `WeakReference<T>` class to help you:
```C#
public sealed class WeakReference<T> : ISerializable where T : class {
   public WeakReference(T target);
   public WeakReference(T target, Boolean trackResurrection); 
   public void SetTarget(T target);
   public Boolean TryGetTarget(out T target);
}
```
This class is really just an object-oriented wrapper around a GCHandle instance: logically, its constructor calls GCHandle's Alloc, its TryGetTarget method queries GCHandle’s Target property, its SetTarget method sets GCHandle's Target property, and its Finalize method (not shown in the preceding code, because it's protected) calls GCHandle's Free method. In addition, no special permissions are required for code to use the `WeakReference<T>` class because the class supports only weak references; it doesn't support the behavior provided by GCHandle instances allocated with a GCHandleType of Normal or Pinned.  The downside of the `WeakReference<T>` class is that an instance of it must be allocated on the heap. So the `WeakReference<T>` class is a heavier-weight object than a GCHandle instance. 

<div class="alert alert-info p-1" role="alert">
    When developers start learning about weak references, they immediately start thinking that they are useful in caching scenarios, For example, they think it would be cool to construct a bunch of objects that contain a lot of data and then to create weak references  to these objects. When the program needs the data, the program checks the weak reference to see it th eobject that contains the data is still around, and if it is, the program just uses it; the program experiences high performance. However, if a garbage collection occurred, the objects that contained the data would be destroyed, and when the program has to re-create the data, the program experiences lower performance.
    </br></br>
    The problem with this technique is the following: garbage collections do not only occur when memory is full or close to full. Instead, garbage collections occur whenever generation 0 is full. So objects are being tossed out of memory much more freuently than desired, and your application's performance suffers greatly.
</div>

Developers frequently want to assocaite a piece of data with another entity. For example, you can associate data with a thread or with an AppDomain. It is also possible to associate data with an individual object by using the `System.Runtime.CompilerServices.ConditionalWeakTable <TKey,TValue>` class, which looks like this:
```C#
public sealed class ConditionalWeakTable<TKey, TValue> where TKey : class where TValue : class {
   public ConditionalWeakTable();
   public void Add(TKey key, TValue value);
   public TValue GetValue(TKey key, CreateValueCallback<TKey, TValue> createValueCallback);
   public Boolean TryGetValue(TKey key, out TValue value);
   public TValue GetOrCreateValue(TKey key);
   public Boolean Remove(TKey key);

   public delegate TValue CreateValueCallback(TKey key); // Nested delegate definition
}
```
If you want to associate some arbitrary data with one or more objects, you would first create an instance of this class. Then, call the Add method, passing in a reference to some object for the key parameter and the data you want to associate with the object in the value parameter. If you attempt to
add a reference to the same object more than once, the Add method throws an ArgumentException; to change the value associated with an object, you must remove the key and then add it back in with the new value.

What makes the ConditionalWeakTable class so special is that it guarantees that the value remains in memory as long as the object identified by the key is in memory. 

Here is some code that demonstrates the use of the ConditionalWeakTable class. It allows you to call the GCWatch extension method on any object passing in some String tag. Then it notifies you via the console window whenever that particular object gets garbage collected:
```C#
static void Main(string[] args) {
   Object o = new Object().GCWatch("My Object created at " + DateTime.Now);
   GC.Collect(); // We will not see the GC notification here
   GC.KeepAlive(o); // Make sure the object o refers to lives up to here
   o = null; // The object that o refers to can die now

   GC.Collect(); // We'll see the GC notification sometime after this line
   Console.ReadLine();
}

static class GCWatcher {
   private readonly static ConditionalWeakTable<Object, NotifyWhenGCd<String>> s_cwt = new ConditionalWeakTable<Object, NotifyWhenGCd<String>>();

   public static T GCWatch<T>(this T @object, String tag) where T : class {
      s_cwt.Add(@object, new NotifyWhenGCd<String>(tag));
      return @object;
   }

   private sealed class NotifyWhenGCd<T> {
      private readonly T m_value;

      public NotifyWhenGCd(T value) {
         m_value = value;
      }

      public override string ToString() {
         return m_value.ToString();
      }

      ~NotifyWhenGCd() {
         Console.WriteLine("GC'd: " + m_value);
      }
   }
}
```

<!-- ![alt text](./zImages/16-1.png "Title") -->

<!-- [link text](https://) -->

<!-- <code>&lt;T&gt;</code> -->

<!-- <div class="alert alert-info p-1" role="alert">
    
</div> -->

<!-- <div class="alert alert-info p-1" role="alert">
    <h3 class="mt-0">XXX</h2>
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

<!-- <sup>1</sup> -->

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