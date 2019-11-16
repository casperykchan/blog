# So you know try-catch-finally. Glad you say!

```java
public class TryCatchFinallyTest {

    @Test
    public void tryCatchFinally() throws Exception {
        System.out.println("Start");
        int result = test();
        System.out.println("Result: " + result);
    }

    private int test() throws Exception {
        try(Closeable closeable = () -> {
            System.out.println("Closeable");
            throw new IOException();
        }) {
            System.out.println("Try");
            throw new IllegalStateException();
        } catch (Exception e) {
            System.out.println("Catch");
            throw new IllegalArgumentException();
        } finally {
            System.out.println("Finally");
            return 1;
        }
    }
}
```
