# Arrays 
## Declaration and creation 
```java
int[] arr = new int[5];        // preferred style
int arr2[] = new int[5];       // legal, but C-style — avoid

int[] nums = {1, 2, 3, 4, 5};  // initializer syntax
```
- Arrays are fixed-size, contiguous blocks of memory. Once created, their length cannot change.
- Arrays are **objects** in Java — they live on the heap, have a `length` field (Not method ).

### Multi-dimensional and jagged arrays 
```java
int[][] grid = new int[3][3];       // proper 2D, 3x3

int[][] jagged = new int[3][];      // legal — rows not yet sized
jagged[0] = new int[2];
jagged[1] = new int[5];
jagged[2] = new int[1];
```

- Java truly doesn't have a multidimensional array type. `int[][]` is an array of arrays. Each row can have a different length (jagged).

```java
int[][] grid = new int[3][4];
```
The outer array has length 3 → these are the rows
Each inner array has length 4 → these are the columns (each row's own array)
```
grid[0] → [ 0, 0, 0, 0 ]   // row 0, 4 columns
grid[1] → [ 0, 0, 0, 0 ]   // row 1, 4 columns
grid[2] → [ 0, 0, 0, 0 ]   // row 2, 4 columns

int[][] a = new int[3][4];   // fully valid — both dimensions fixed
int[][] b = new int[3][];    // valid — outer size fixed, rows not yet created
int[][] c = new int[][4];    // ❌ ILLEGAL — can't skip the outer size and specify inner
```

> Until you assign jagged[0], jagged[1], etc., each row is null — not an empty array, actually null, because the outer array only holds references to inner arrays, and those references haven't been pointed anywhere yet.


> Trap: accessing jagged[0][0] before assigning jagged[0] = new int[2] throws NullPointerException — you're dereferencing a null row.


### Array covariance 
- Arrays in Java are covariant — String[] is treated as a subtype of Object[]:
```java
Object[] objs = new String[3];  // legal — arrays allow this
objs[0] = "hello";              // fine
objs[0] = 42;                   // compiles! (Integer autoboxed to Object)
                                  // but throws ArrayStoreException at RUNTIME
```

### Useful Arrays utility method
The `java.util.Arrays` class is a static utility class — every method is static
```java
Arrays.toString(arr)          // 1D readable print
Arrays.deepToString(arr2d)    // for nested/multi-dimensional arrays

Arrays.sort(arr)                    // ascending, in place
Arrays.sort(arr, fromIndex, toIndex)// sort only a sub-range
Arrays.sort(arr, comparator)        // only for object arrays — primitives have no custom comparator overload
Arrays.parallelSort(arr)            // same as sort(), but multi-threaded for large arrays
/**
 * Under the hood: primitives use a dual-pivot quicksort; object arrays use a modified mergesort (TimSort) — because object sorts need to be stable (equal elements keep relative order), and quicksort isn't stable.
**/

Arrays.binarySearch(arr, x)   // requires sorted array first

Arrays.equals(a, b)         // shallow — element-by-element for 1D
Arrays.deepEquals(a, b)     // for nested arrays
Arrays.compare(a, b)        // lexicographic comparison, returns negative/zero/positive
Arrays.mismatch(a, b)       // returns index of first mismatch, or -1 if equal

Arrays.fill(arr, value)               // fill entire array
Arrays.fill(arr, fromIndex, toIndex, value)  // fill a range
Arrays.setAll(arr, i -> i * i)         // fill using a generator function (index → value)

Arrays.copyOf(arr, newLength)          // truncates or zero/null-pads
Arrays.copyOfRange(arr, from, to)      // copy a specific sub-range

Arrays.hashCode(arr)          // hash based on contents
Arrays.deepHashCode(arr2d)
```

- Arrays.toString() on a 2D array gives you garbage like [[I@1b6d3586, [I@4554617c] — the inner arrays print as their default Object.toString(), not their contents. You need deepToString() for nested arrays. 
- Arrays.asList(someArray) — I'm intentionally not covering this in depth now since it bridges into the Collections framework, which we're keeping separate. Just know it exists; we'll cover it properly in that module.

> Trap: arr1 == arr2 and arr1.equals(arr2) both check identity, not contents — arrays never override equals(). Always use Arrays.equals() to compare array contents.

```java
List<String> list = Arrays.asList("a", "b", "c");
```
> Trap 1: this returns a fixed-size list backed by the array — you can set() elements (it writes through to the array), but add()/remove() throw UnsupportedOperationException. It's not a normal resizable ArrayList. 

> Trap 2: Arrays.asList(intArray) where intArray is int[] — gives you List<int[]> of size 1 (the whole array becomes a single element), not List<Integer>. This happens because primitives can't be generic type parameters — varargs here treats the whole primitive array as one object. Works correctly only with object arrays (Integer[], String[], etc.).

```java
int[] nums = {1, 2, 3};
List<int[]> list = Arrays.asList(nums);

System.out.println(list.size());     // 1 — NOT 3!
System.out.println(list.get(0));     // prints something like [I@1b6d3586
                                       // (it's the int[] array itself, printed as an object)
```

What happened: Arrays.asList(T... a) is a varargs method. Varargs only works with object types, not primitives. So when you pass int[] nums, Java can't "spread" it into int, int, int — instead, it treats the entire int[] as a single object, and wraps it as a one-element List<int[]>.

```java
Integer[] nums = {1, 2, 3};
List<Integer> list = Arrays.asList(nums);

System.out.println(list.size());     // 3 — correct
System.out.println(list.get(0));     // 1
```


### Varargs
- Varargs (variable-length argument lists) are syntactic sugar for arrays (Similar to enhanced for loops). 
- A method can declare a parameter like `String... args`, which allows callers to pass any number of String arguments (including zero). Internally, the compiler creates an array for the varargs parameter.

```java
void printAll(String... args) {
    for (String s : args) {
        System.out.println(s);
    }
}
```

> Trap: calling with zero arguments gives you a zero-length array — not null. Also, if you have both print(Object o) and print(Object... o) overloaded, calling print(null) is ambiguous/resolves to the non-varargs version — the compiler prefers the "more specific" non-varargs match.
