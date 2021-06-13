# (A bit) More on Reactive Variable with Relations

After the initial post about reactive variable with relations, I further enhance it with a use case as noted in the post : GUI applications.

I may explain with more details later on (and that's the major reason I prefer blogging as a git repository rather than wordpress), but here is the snippet using JavaFX:

```
import javafx.application.Application;
import javafx.scene.Scene;
import javafx.scene.control.Label;
import javafx.scene.layout.VBox;
import javafx.stage.Stage;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.List;
import java.util.Objects;
import java.util.Optional;
import java.util.Random;
import java.util.concurrent.ScheduledExecutorService;
import java.util.function.BiConsumer;
import java.util.function.Consumer;
import java.util.function.Function;
import java.util.stream.Stream;

import static java.util.concurrent.Executors.newSingleThreadScheduledExecutor;
import static java.util.concurrent.TimeUnit.MILLISECONDS;
import static java.util.concurrent.TimeUnit.SECONDS;
import static javafx.application.Platform.runLater;

public class MainApp extends Application {

    @Override
    public void start(Stage stage) {
        stage.setTitle("Reactive Variable");
        VBox root = new VBox();

        BiConsumer<String, Label> updateCss = (css, label) -> {
            runLater(() -> {
                label.getStyleClass().clear();
                label.getStyleClass().add(css);
            });
        };

        var stockPrice = new Label("");
        var outstandingShares = new Label("");
        var marketCap = new Label("");

        Stream.of(stockPrice, outstandingShares, marketCap).forEach(label -> updateCss.accept("label-default", label));

        root.getChildren().add(stockPrice);
        root.getChildren().add(outstandingShares);
        root.getChildren().add(marketCap);

        Scene scene = new Scene(root, 300, 100);
        scene.getStylesheets().add(MainApp.class.getResource("styles.css").toExternalForm());
        stage.setScene(scene);
        stage.show();
        
        // --- Real fun begins ---

        ScheduledExecutorService scheduler = newSingleThreadScheduledExecutor();

        var pricesLabel = new Variable<>(stockPrice);
        var sharesLabel = new Variable<>(outstandingShares);
        var marketCapLabel = new Variable<>(marketCap);

        BiRelation<String, Label, String> toShow = (val, label) -> {
            runLater(() -> label.setText(val));
            return val;
        };

        Function<Integer, BiRelation<String, Label, String>> flash = interval -> (val, label) -> {
            updateCss.accept("label-orange", label);
            scheduler.schedule(() -> updateCss.accept("label-default", label), interval, MILLISECONDS);
            return val;
        };

        var flash300 = flash.apply(300);
        var flash500 = flash.apply(500);

        BiRelation<BigDecimal, BigDecimal, BigDecimal> multiply = (i1, i2) -> i1.multiply(i2);

        Relation<BigDecimal, String> toPlainString = val -> val.toPlainString();
        Relation<BigDecimal, String> toUsd = val -> "USD " + val.toPlainString();

        var prices = new Variable<BigDecimal>();
        var shares = new Variable<BigDecimal>();

        toShow.bind(prices.andThen(toPlainString), pricesLabel).andThen(flash300, pricesLabel);
        toShow.bind(shares.andThen(toPlainString), sharesLabel).andThen(flash300, sharesLabel);

        multiply.bind(prices, shares).andThen(toUsd).andThen(toShow, marketCapLabel).andThen(flash500, marketCapLabel);

        Random rand = new Random();
        scheduler.scheduleAtFixedRate(() -> prices.set(new BigDecimal("1." + rand.nextInt(1000))), 1, 2, SECONDS);
        scheduler.scheduleAtFixedRate(() -> shares.set(new BigDecimal("1000" + rand.nextInt(1000))), 1, 5, SECONDS);
    }

    public static void main(String[] args) {
        launch(args);
    }

    public static class Variable<T> {
        private Optional<T> value;
        private List<Consumer<Variable<T>>> retro = new ArrayList<>();

        public Variable() {
            this.value = Optional.empty();
        }

        public Variable(T value) {
            this.value = Optional.of(Objects.requireNonNull(value));
        }

        public synchronized void set(T newT) {
            value = Optional.of(newT);
            retro.stream().forEach(r -> r.accept(this));
        }

        public synchronized Optional<T> val() {
            return value;
        }

        <NEW_T> Variable<NEW_T> andThen(Relation<T, NEW_T> then) {
            return then.bind(this);
        }

        <R, NET_T> Variable<NET_T> andThen(BiRelation<T, R, NET_T> then, Variable<R> right) {
            return then.bind(this, right);
        }
    }

    @FunctionalInterface
    public interface Relation<IN, OUT> {

        default Variable<OUT> bind(Variable<IN> input) {
            var result = input.value.isPresent() ? new Variable<>(apply(input.value.get())) : new Variable<OUT>();
            input.retro.add(retro -> retro.value.map(this::apply).ifPresent(result::set));
            return result;
        }

        OUT apply(IN input);

        default <NEW_OUT> Relation<IN, NEW_OUT> andThen(Relation<OUT, NEW_OUT> then) {
            return (in) -> then.apply(this.apply(in));
        }
    }

    @FunctionalInterface
    public interface BiRelation<LEFT, RIGHT, ANSWER> {

        default Variable<ANSWER> bind(Variable<LEFT> left, Variable<RIGHT> right) {

            var result = left.value.isPresent() && right.value.isPresent() ? new Variable<>(apply(left.value.get(), right.value.get())) : new Variable<ANSWER>();

            left.retro.add(leftRetro -> {
                if (leftRetro.value.isPresent() && right.value.isPresent()) {
                    ANSWER newVal = apply(leftRetro.value.get(), right.value.get());
                    result.set(newVal);
                }
            });

            right.retro.add(rightRetro -> {
                if (left.value.isPresent() && rightRetro.value.isPresent()) {
                    ANSWER newVal = apply(left.value.get(), rightRetro.value.get());
                    result.set(newVal);
                }
            });

            return result;
        }

        ANSWER apply(LEFT left, RIGHT right);
    }
}
```

Sadly, I will stop exploring this topic any further as the major drawback - reactive variable being mutable - is really a show-stopper. I hope I can elborate further in later updates.
