Is Java pass-by-copy or pass-by-reference? Let us find out.

```
public class ID {
    public String id;

    public ID(String id) {
        this.id = id;
    }

    @Override
    public String toString() {
        return "ID{\'" + id + "\'" +"}";
    }
}

ID one = new ID("ONE");
```

Quick google on "Is java pass-by-reference?". The answer is mostly yes.

```
public static void changeId_pass_by_ref(ID one) {
    ID two = new ID("TWO");
    one.id = two.id;
}

changeId_pass_by_ref(one)

// ID one = ID{'TWO'}
System.out.println(one);
```

However many also argue Java is pass-by-copy:

```
public static void changeId_pass_by_copy(ID one) {
    ID two = new ID("TWO");
    one = two;
}

changeId_pass_by_copy(one)

// ID one = ID{'ONE'}
System.out.println(one);
```

So what is the answer after all?

In C++, pointers (int*) and references (int&) are pass-by-reference, while object types are pass-by-copy.

In Java, direct pointer manipulation is not supported,
so method calls only accept arguments in form of primitive/object types.
But is it really passing the object naively just by the look of the method signature?

Turns out, Java is actually "pass pointer by copy, in form of referenced object".

WTH does that mean?

To put in C++ context, it is pointer with pass-by-copy
```
public static void changeId_copy_of_reference(ID* one_pointer_copy) {
    ID two = new ID("TWO");

    // this assignment has no effect in Java, because the pointer is passed by copy
    one_pointer_copy = &two;

    // this assignment works as expected, because the pointer copy "points" to object instance "one"
    one_pointer_copy->id = two.id;
}
```

Wait... Java does not support pointers, right?

Let's try to break down a bit further. Disclaimer ahead,
strictly speaking this does not represent how java works under the hook,
but it may reveal how object referencing works in Java.

First let's revisit the object instantiation:

```
ID one = new ID("ONE");
```

Essentially, when the object "one" is instantiated,
there is a pointer object "pointer_one" created which "points" to the memory location of the object in the heap.

As an imaginary example:

```
pointer_one = new pointer()
pointer_one.address = "A0 B0 C0 D0" // 4 Bytes for 32bit; 8 Bytes for 64bit
Inside the jvm heap, the memory address "A0 B0 C0 D0" points to the allocated memory of the object (ID "ONE")
```

When changeId_copy_of_reference(one) is called, you may view that it is passing pointer_one.

```
changeId_copy_of_reference(pointer_one);
```

Note that the pointer_one is actually pass-by-copy:

```
public static void changeId_copy_of_reference(pointer_one_copy) {
```

Any attempt to directly assigning the copied pointer would not cause any effect at all to the referenced object.

```
public static void changeId_copy_of_reference(pointer_one_copy) {
    ID two = new ID("TWO");
    pointer_two = new pointer()
    pointer_two.address = "A1 B1 C1 D1"
    
    // this assignment has no effect (which is expected)
    pointer_one_copy = pointer_two;
...
```

In case the object is needed, it will be referenced.

For instance, if the java code is:
```
copy_of_reference_to_one.id = two.id;
```

it will look like:

```
ID one = new ID("ONE");
//pointer_one = new pointer()
//pointer_one.address = "A0 B0 C0 D0"

public static void changeId_copy_of_reference(pointer_one_copy) {
    ID two = new ID("TWO");
    //pointer_two = new pointer()
    //pointer_two.address = "A1 B1 C1 D1"
    
    // Internal Java pointer referencing
    //ID one_reference = refernce(pointer_one_copy)
    one_reference.id = two.id;
}
```

This explains in very high level how passing mechanism works in Java.
We most of the time take it for granted to avoid complicated pointer manipulation in Java,
it does not mean we are saved from memory pointer consideration though.
