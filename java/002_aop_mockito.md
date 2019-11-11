# Naive AOP by Mockito Spying

A warning to begin with :
It is just for my fun and curiosity only.
Do not ever attempt the below in any of your production code!
Mocking framework may slow down applications considerably
and result in unexpected results in non-testing environment.

---

We all know about the importance of unit testing,
as we need to be responsible for testing every line of code we write.
 
Nowadays Mocking framework is often needed in order to do unit testing properly.
When it comes to mocking framework,
Mockito is the first and only choice for many Java programmers,
since it presents an intuitive and elegant way to interact with mocks.
It also provides flexibility by spying real instances,
so unit testing can be done effectively.

Mockito is great,
but I believe mostly all of us would think Mockito is mainly for the purpose of unit testing only.

Or maybe not?
Ever thought of using Mockito in non-testing environment?

Let's find out!

## Mockito under the hook

To take a nice sip of Mockito,
we can first see know how Mockito generally works.

To mock a List, for instance:

```
List<String> list = mock(List.class);
```

Mockito internally creates and keeps track of a List mock instance
which is wired and returned as a proxy.
This is a perfect implementation of the classic Proxy pattern.

You can work on the proxy with the interface (i.e. _java.util.List_ in the example.)
To inject behavior,
or to "stub" technically speaking,
you can use Mockito `when` method:

```
when(list.get(0)).thenReturn("first");

System.out.println(list.get(0));  //prints "first"
```

Spying on Mockito works very much the same,
except that you need to use `doAnswer` or `doReturn` method:

```
var listSpy = spy(new ArrayList<String>());

doReturn("first").when(listSpy).get(0);

System.out.println(listSpy.get(0));  //prints "first"
```

Now we understand Mockito's core is all about Proxy pattern,
then the question would then become :
What can we do with Mockito proxies and its stubbing mechanism?

Sort-of naive compile or code time AOP, I would say.

## AOP and Mockito

AOP is such a big topic that I would not even dare to touch a single bit of it.
If I may allow to put it into very simple words,
AOP allows you to "inject" your logic into existing code and runtime.

One typical use case of compile-time AOP is separating logging api calls from the business logic code,
so that the business code will not be polluted by logging method calls with the trade off of enabling AOP engine in the runtime.

I emphasize "Compile-time AOP" here because that you can also do it in runtime.
It essentially means Strategy pattern in runtime which is very powerful if you think of it.
Anyway I will not cover those in detail here.

Thanks to the intuitive mechanism how Mockito stubs behavior to spied instances or mocks,
we can easily inject behavior (or _concerns_ in terms of AOP) in compile time as described above.

## Hooks

Let us start with the coding by `Hook` builder class :

```
class Hook {

    private final List<Runnable> doFirsts = new ArrayList<>();
    private final List<Consumer<Object>> doLasts = new ArrayList<>();

    Hook with(Runnable doFirst) {
        doFirsts.add(doFirst);
        return this;
    }

    Hook with(Consumer<Object> doLast) {
        doLasts.add(doLast);
        return this;
    }

    Object accept(InvocationOnMock invocationOnMock) throws Throwable {
        doFirsts.forEach(doFirst -> doFirst.run());
        Object result = invocationOnMock.callRealMethod();
        doLasts.forEach(doLast -> doLast.accept(result));
        return result;
    }
}
```

I borrow the idea of `doFirst` and `doLast` hooks from basic Gradle tasks.
If you are familiar with Gradle, you should get it easily.

The `accept` method runs the doFirst and doLast hooks before and after the invocation respectively.

### Weaving Hooks to Spied Instances

Then we can use Mockito's stubbing mechanism to inject hooks for the targeted instance.
Best explained by an example.

Knock Knock. Who's there?

```
class KnockKnock {
    void knockKnock(String who) {
        System.out.println(who + ": Who's there?");
    }
}
```

An instance of KnockKnock can be created :

```
var joke = new KnockKnock();

joke.knockKnock("Bruce");
```

which prints :

```
Bruce: Who's there?
```

Nothing interesting so far. Bruce is talking to himself for now,
but he is not alone anymore when Joker joins by spying with hooks:

```
var spied = spy(joke);

doAnswer(invocation -> new Hook()
        .with(() -> System.out.println("Joker : Knock Knock. " + invocation.getArguments()[0] + "!"))
        .with(result -> System.out.println("Joker : Not your parents"))
        .accept(invocation)
).when(spied).knockKnock(any());

spied.knockKnock("Bruce");
```

which now prints :

```
Joker : Knock Knock. Bruce!
Bruce: Who's there?
Joker : Not your parents
```

(Sad sad Bruce... joke from the internet btw)

As you can see, the hooks are triggered before and after the method invocation.

## Conclusion

Using Mockito in non-testing environment may sound absurd,
due to Mockito's superb stubbing mechanism though,
it is easy to add behavior to both mocks and real instances.

Although I haven't studied Mockito's implementation thoroughly,
it is very enjoyable to realize how Mockito's stub api is achieved
(like, why `when(mock.call()).thenReturn("result)` actually works.)

It is helpful to write better unit tests as a return, without doubt.
