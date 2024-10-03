Is Java pass-by-copy or pass-by-reference?

Quick google on "Is java pass-by-reference?". The answer is mostly yes.
However interestingly, many also argue Java is pass-by-copy.

which one is the correct answer? Let's find out.

Start off with a simple POJO:

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

It seems Java is pass-by-reference:

```
public static void changeId_pass_by_reference(ID one) {
    ID two = new ID("TWO");
    one.id = two.id;
}

changeId_pass_by_refernce(one)

// ID one = ID{'TWO'}
System.out.println(one);
```

If the method changeId_pass_by_refernce can update the paased parameter (ID "one") it is a proof that it is not pass-by-copy.

But it looks like Java is pass-by-copy as well...?

```
public static void changeId_pass_by_copy(ID one) {
    ID two = new ID("TWO");
    one = two;
}

changeId_pass_by_copy(one)

// ID one = ID{'ONE'}
System.out.println(one);
```

Since the assignment statement 
```
one = two;
```
has no effect on the passed parameter, it is a proof that it is pass-by-copy.


Weird. The answers seem to be contradicting to each other!

In fact, we need to take a step back to think about how pointer in C++ works.
It may sound absurd but bear with me.

In C++, pointers (say int*) and references (say int&) are pass-by-reference, while object types are pass-by-copy.

In Java, direct pointer manipulation is not supported,
so method calls only accept arguments in form of primitive/object types.
But is it really passing the object naively by the look of method signature?

Turns out, Java is actually "pass by copy **as pointer in form of referenced object**".

WTH does that mean?

To put that in C++ context, it is **pointer with pass-by-copy** (which actually does not exist.)

As an imaginary example:

```
public static void changeId_copy_of_reference(ID* one_pointer_copy) {
    ID two = new ID("TWO");

    // this assignment has no effect in Java, because the pointer is passed by copy
    one_pointer_copy = &two;

    // this assignment works as expected, because the pointer, even if it is a copy, still "references" to object instance "one"
    one_pointer_copy->id = two.id;
}
```

Wait... Java does not support pointers, right?

Let's try to break down a bit further. Disclaimer ahead,
strictly speaking this does not represent how things really work under the hood,
but it may reveal some ideas of object referencing in Java.

Let's revisit the object instantiation:

```
ID one = new ID("ONE");
```

When the object "one" is instantiated,
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
public static void changeId_copy_of_reference(pointer_one_pass_by_copy) {
```

Any attempt to directly assigning the copied pointer would not cause any effect at all to the referenced object.

```
public static void changeId_copy_of_reference(pointer_one_pass_by_copy) {
    ID two = new ID("TWO");
    pointer_two = new pointer()
    pointer_two.address = "A1 B1 C1 D1"
    
    // this assignment has no effect (which is expected)
    pointer_one_pass_by_copy = pointer_two;
...
```

In case the object is needed, the pointer will be referenced.

It will look like this:

```
ID one = new ID("ONE");
//pointer_one = new pointer()
//pointer_one.address = "A0 B0 C0 D0"

public static void changeId_copy_of_reference(pointer_one_pass_by_copy) {
    ID two = new ID("TWO");
    //pointer_two = new pointer()
    //pointer_two.address = "A1 B1 C1 D1"
    
    // Internal Java pointer referencing
    //ID one_reference = refernce(pointer_one_pass_by_copy)
    one_reference.id = two.id;
}
```

This explains in very high level how passing mechanism works in Java.
We most of the time take it for granted to avoid complicated pointer manipulation in Java,
it does not mean we are saved from memory pointer consideration though.
