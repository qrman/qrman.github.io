---
layout: post
title:  "Pipeline pattern - Plumber quest"
date:   2017-02-09 12:00:00
comments: true
tags: 
  - java
  - Pipeline Pattern
  - Streams
  - Optional
  - Chain of Responsibility Pattern
  - Functional Programming

description: Some thoughts about Pipeline Pattern in Java 8 world.

---

### Describing a problem ###

Back then in old good Java 7 days, one of my favourite design pattern was *Pipeline Pattern*.
Following this pattern allows me to have my code very well organized. Especially when request processing 
requires many steps. 

{% highlight java %}

 public RequestResponse processRequest(RequestInput input) {
        return new Pipeline<>(
                inputValidator,
                itemRepository,
                itemPriceCalculator,
                responseConverter
        ).process(input);
    }
{% endhighlight %}

Every time when I was forgetting this pattern my code ends like this with business logic
hidden somewhere in third layer of processing.

{% highlight bash %}


Controller
 - Service A
    - Service B
        - Service C
            - Service D
 - Service E  

{% endhighlight %}


Flat processing structure is something I always appreciate. It is easy to read, 
flexible and ready for adding new features in process.


Recently I've decided to refresh this approach while writing new service. It comes to me that since we have Java 8, there
are few more ways to implement pipeline pattern. I will present the four easiest ones:

- classical Pipeline Pattern
- Optional way
- Stream way
- Functional way

### Mario Bros Dilemma ###

Let's start with some inspiration for writing a simple program. Mario, well known plumber is looking for princess. Most of the time,
the only clue he gets is:

> Thank You Mario!

> But our princess is in another castle!

In my example we will analyze sentence like this:

> Princess Francesca is guarded by Dragon

Each plumber: 
- Mario, 
- Luigi, 
- IT Plumber,
- Ultimate Plumber

can beat one or more beasts, such as:

- Dragon,
- Kraken,
- System Administrator

Application will analyze input sentence and decide whether plumber can beat the beast and rescue princess.
Expected output will look like this:

{% highlight Bash %}
INFO: Result: Luigi will be defeated by DRAGON and princess Francesca will have to wait for another Plumber :(
{% endhighlight %}

or like this:

{% highlight Bash %}
INFO: Result: Mario will beat the DRAGON with CANDY and save princess Francesca!!!
{% endhighlight %}


### Old is Gold - Pipeline Pattern ###

Firstly, we need some glue code. Two simple classes:

{% highlight java %}

public class Pipeline<T> {

    private final List<Pipe<T>> pipes;

    public Pipeline(Pipe<T>... pipes) {
        this.pipes = Arrays.asList(pipes);
    }

    public T process(T input) {
        T processed = input;
        for (Pipe<T> pipe : pipes) {
            processed = pipe.process(processed);
        }
        return processed;
    }
}

{% endhighlight %}

and:


{% highlight java %}


public interface Pipe<T> {

    T process(T input);
}

{% endhighlight %}

Then we can combine first pipeline flow:

{% highlight java %}

public class PipelineFlow {

    private final Pipe<PlumberQuest> princessNameFinder;
    private final Pipe<PlumberQuest> beastWeaknessFinder;
    private final Pipe<PlumberQuest> fightResult;

    public PipelineFlow() {
        this.princessNameFinder = new PrincessNameFinderPipe();
        this.beastWeaknessFinder = new BeastWeaknessFinderPipe();
        this.fightResult = new FightResultPipe();
    }

    public PlumberQuest process(PlumberQuest quest) {
        return new Pipeline<>(
                princessNameFinder,
                beastWeaknessFinder,
                fightResult
        ).process(quest);
    }
}

{% endhighlight %}

The above code is really simple. Plumber quest consists of three steps. Firstly, he should find princess name,
then check beast weakness and finally verify whether plumber can beat the beast or not. As you can see
there are no Java 8 features in this code. All next three samples are available only in Java 8 world.

### Optional Way ###

Optional gives much more than `isPresent()` and `get()`. [Optional](https://docs.oracle.com/javase/8/docs/api/java/util/Optional.html) API contains some useful methods
which are helpful in simple data transformation such as `map()` and `filter()`. Optional wasn't designed for solving pipeline pattern
but they are very handy for simple pipes. We just need to wrap input data using Optional with `Optional.of()` method and then call `map` in every step in the pipe. Second flow looks as following:

{% highlight java %}

public class OptionalFlow {

    private final PrincessNameFinder princessNameFinder;
    private final BeastWeaknessFinder beastWeaknessFinder;
    private final FightResult fightResult;

    public OptionalFlow() {
        this.princessNameFinder = new PrincessNameFinder();
        this.beastWeaknessFinder = new BeastWeaknessFinder();
        this.fightResult = new FightResult();
    }

    public PlumberQuest process(PlumberQuest quest) {
        return Optional.of(quest)
                .map(princessNameFinder::find)
                .map(beastWeaknessFinder::find)
                .map(fightResult::fightAgainstBest)
                .get();
    }
}

{% endhighlight %}


### Streams, Streams, Streams ###

Very similar effects gives wrapping all steps using stream with one element. Stream API is well known for everyone who
writes anything in Java 8.

{% highlight java %}

public class StreamFlow {

    private final PrincessNameFinder princessNameFinder;
    private final BeastWeaknessFinder beastWeaknessFinder;
    private final FightResult fightResult;

    public StreamFlow() {
        this.princessNameFinder = new PrincessNameFinder();
        this.beastWeaknessFinder = new BeastWeaknessFinder();
        this.fightResult = new FightResult();
    }

    public PlumberQuest process(PlumberQuest quest) {
        return Stream.of(quest)
                .map(princessNameFinder::find)
                .map(beastWeaknessFinder::find)
                .map(fightResult::fightAgainstBest)
                .findFirst()
                .get();

    }
}

{% endhighlight %}

Stream looks nice, but last step - getting result data - is quite lame. We have to call
`findFirst()` which returns Optional and `get()` at the end. The other way is to call `collector()` and
take first element of result list.


### Functions for purists ###

Last approach uses `Function` API. The truth is that functions were used under the hood in Optional and Stream examples.
Right now we will use function explicitly. Here we go:

{% highlight java %}

public class FunctionalFlow {

    private final Function<PlumberQuest, PlumberQuest> princessFunction;
    private final Function<PlumberQuest, PlumberQuest> beastFunction;
    private final Function<PlumberQuest, PlumberQuest> fightResultFunction;

    public FunctionalFlow() {
        this.princessFunction = quest -> new PrincessNameFinder().find(quest);
        this.beastFunction = quest -> new BeastWeaknessFinder().find(quest);
        this.fightResultFunction = quest -> new FightResult().fightAgainstBest(quest);
    }

    public PlumberQuest process(PlumberQuest quest) {
        return princessFunction
                .andThen(beastFunction)
                .andThen(fightResultFunction)
                .apply(quest);
    }
}

{% endhighlight %}

As you can see there are more than one way to implement pipes in Java 8. You can pick those you like the most.

### Epilogue ###

A few weeks ago I started learning RxJava. I realized that RxJava is the modern Pipeline Pattern. It is much more easy to dive into
world of Observables when you followed Pipeline Pattern in the past. What is more, it is really easy to rewrite code into Rx.
Last but not least: I found out that it is quite a good approach (when you are not RxNinja) firstly to write Java 8 pipes and then to refactor code with usage of Rx streams.
