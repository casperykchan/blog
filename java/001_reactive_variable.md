# Reactive Variables with Relations

Below is a ordinary function call in Java:

```
BiFunction<Integer, Integer, Integer> add = (a, b) -> a + b;

var a = 1;
var b = 2;
var result = add.apply(a, b);

// result
3
```

One good thing about functions or lambda expressions is that it represents the logic rather than the actual value.
The "add" function adds two integers, not hard-coding the result of 1 + 2,
so it can be applied repeatedly:

```
a = 100;
b = 200;
result = add.apply(a, b);

// result
300
```

However things can become handy if things are bind together:

```
Let: c = (a, b) -> a + b

a = 1;
b = 2;  // c is 3 at this point

a = 11; // c becomes 13
```

Instead of applying the function multiple times,
changes on input variables can be reflected to the result variable immediately.

It matches more with the mathematics semantics too,
as changes on inputs (domain) are reflected to the result variable (codomain).
The functions can then be treated as pure relations.

## Observer Pattern

To a certain extent, variable binding means observer pattern to variables,
so variables can propagate state changes to their observers.

However unlike data binding of database access or property binding of GUI components,
the key idea of function binding is once the doamin values are changed,
the observers are retrospectively updated so that the codomain values can be mapped again as a result.

If reactive programming (like RxJava or Flow API in Java) comes to your mind at this point, you are absolutely right!
To demonstrate the idea though, I would stick to composition which is simpler.

(My confession: the learning curve of RxJava is just too high!)

## Variable Class

Let us start with the definition of "Variable" class:

```java
public static class Variable<T> {
    private Optional<T> value;
    private Consumer<Variable<T>> retro;

    public Variable() {
        this.value = Optional.empty();
    }

    public Variable(T value) {
        this.value = Optional.of(Objects.requireNonNull(value));
    }

    public void set(T newT) {
        value = Optional.of(newT);
        if (nonNull(retro)) {
            retro.accept(this);
        }
    }

    public Optional<T> val() {
        return value;
    }

    <NEW_T> Variable<NEW_T> andThen(Relation<T, NEW_T> then) {
        return then.bind(this);
    }

    <R, NET_T> Variable<NET_T> andThen(BiRelation<T, R, NET_T> then, Variable<R> right) {
        return then.bind(this, right);
    }
}
```

Some notable things apart from the fact that it is just a glorified object wrapper:

1. The value is an optional so that things can be evaluated lazily.
(Of course, it can be done immediately if the variables are ready at the point of binding)

2. Remember that the variable has to listen to its state changes and propagate to its observers.
It is achieved by the retro consumer of the variable itself after a new value is set.

3. The class itself has "andThen" method for relation chaining.
This may look strange at first because in Java the chaining is done inside Function/BiFunction interface.
Let us move on to see how it can be effectively used.

## Relations

Let us also quickly dive into the Relation implementation:

```java
@FunctionalInterface
public interface Relation<IN, OUT> {

    default Variable<OUT> bind(Variable<IN> input) {

        var result = input.value.isPresent() ? new Variable<>(apply(input.value.get())) : new Variable<OUT>();

        input.retro = retro -> {
            if (retro.value.isPresent()) {
                OUT newVal = apply(retro.value.get());
                result.set(newVal);
            }
        };

        return result;
    }

    OUT apply(IN input);

    default <NEW_OUT> Relation<IN, NEW_OUT> andThen(Relation<OUT, NEW_OUT> then) {
        return (in) -> then.apply(this.apply(in));
    }
}
```

This is a single relation, which means it only takes one input.

The major task of "bind" method is to propagate state changes by a retro consumer.
The result binding variable can then reference and apply the consumer for retro updates.

This is very neat, as the parent-child node structure is encapsulated inside consumer lambda expressions,
but not explicitly formed inside the Variable class (i.e. a list of tailed Variable nodes).

## BiRelations

BiRelation is very much identical to single Relation except that it takes two inputs:

```java
@FunctionalInterface
public interface BiRelation<LEFT, RIGHT, ANSWER> {

    default Variable<ANSWER> bind(Variable<LEFT> left, Variable<RIGHT> right) {

        var result = left.value.isPresent() && right.value.isPresent() ? new Variable<>(apply(left.value.get(), right.value.get())) : new Variable<ANSWER>();

        left.retro = leftRetro -> {
            if (leftRetro.value.isPresent() && right.value.isPresent()) {
                ANSWER newVal = apply(leftRetro.value.get(), right.value.get());
                result.set(newVal);
            }
        };

        right.retro = rightRetro -> {
            if (left.value.isPresent() && rightRetro.value.isPresent()) {
                ANSWER newVal = apply(left.value.get(), rightRetro.value.get());
                result.set(newVal);
            }
        };

        return result;
    }

    ANSWER apply(LEFT left, RIGHT right);
}
```

Since there are two inputs, two retro consumers are created to propagate the state changes,
depending on which of the two inputs has been changed.

You may wonder if it is possible to chain the BiRelations like what is done in single relation.
Take two simple bi-relations for instance:

```
add : a + b
multiply : a * b
```

By chaining the relations say add-then-multiply, it becomes:

```
add-then-multiply: multiply(add(a, b), c) 
```

It looks straight forward at first,
however now the resulted relation takes three variables,
because each relation binding would resolve one variable,
so only one of the variable of the latter relations (i.e. the multiply) is replaced.

Alternatively it is possible to chain by variables.
Remember the "andThen" method done inside the Variable class?
It comes into play as it chains better as long as all relations return one result variable,
we would have less trouble and be able to continuously chain using one variable each time.

Things may still be too vague but luckily things are all set! Let us see some examples.

## Putting Things Together

Let us causally imagine a situation for a restaurant.
While it is now offering a discount as a promotion event,
the cashier wants to make sure the billing system can handle other fixed fees like service changes and taxes,
so that the bill amount can be calculated correctly.

Let's see how relations can be applied:
```
BiRelation<Integer, Integer, BigDecimal> tableForTwo = (customer1, customer2) ->  new BigDecimal(customer1 + customer2);
BiRelation<BigDecimal, Double, BigDecimal> discount = (amount, rate) ->  amount.multiply(new BigDecimal(rate));
Relation<BigDecimal, BigDecimal> fees = (amount) -> amount.multiply(new BigDecimal(1.2));
Relation<BigDecimal, String> toCurrency = (amount) -> "HKD " + amount.setScale(2, RoundingMode.HALF_UP);

var alice = new Variable<>(100);
var bob = new Variable<>(100);
var discountRate = new Variable<Double>();

var bill = tableForTwo.bind(alice, bob)
        .andThen(discount, discountRate)
        .andThen(fees)
        .andThen(toCurrency);

bill.val().ifPresent(System.out::println);  // still empty
```

The relations are defined like normal Functions/BiFunctions.
Then the bill is formed by a chain of relations with the necessary binding variables.
Since the discount rate is not yet decided, the bill will be evaluated as an empty optional.

That's pretty normal, right?
Let us try to set a discount and see!

Unlike the usual way of calling functions/lambda expression,
now you only need to set the discountRate variable and keep the relations untouched:

```
discountRate.set(0.8);
bill.val().ifPresent(System.out::println);

// bill
HKD 192.00
```

Nice!

Say now Alice wants a extra pudding to complete the wonderful dinner.
Luckily the cashier only needs to update the input field once and the new bill amount will be updated accordingly:

```
alice.set(120);
bill.val().ifPresent(System.out::println);

//bill
HKD 211.20
```

Now the usage can be simplified as you can focus on the data values:
```
// What about 30/20/10/0 % off?

DoubleStream.of(0.7, 0.8, 0.9, 1.0)
        .forEach(rate -> {
            discountRate.set(rate);
            bill.val().ifPresent(System.out::println);
        });

//bill
HKD 184.80
HKD 211.20
HKD 237.60
HKD 264.00
```

## Conclusion

Reactive variable comes with great power, but it also needs great responsibility in order to use it correctly.
Since the variables are mutable, they are not thread-safe and extra cautions must be taken.
It is not a good idea to use it as class fields or the it becomes magic and can be difficult to keep track of
the state changes, making it harder to debug or support.

Still, it's a lot of fun playing with reactive variables!
