---
layout: post
title:  "Vert.x with jOOQ and ULTM or story about potato"
date:   2015-10-05 20:00:00
comments: true
tags: 
  - java
  - vert.x
  - jOOQ
  - Guice
  - Liquibase
  - ULTM
  - transactions  
  - SQL
description: Post describes how such frameworks/libs like Vert.x, jOOQ, Guice and ULTM can be combined together.

---

### Vert.x / Guice / jOOQ / ULTM ###

When you are choosing Vert.x as your main framework you have to be fully asynchronous, they said.
```JDBC```? Forget! You should avoid thread blocking. There are only few of them, so you cannot tell them to stop.


Anyway, Vert.x allows running blocking code, so why not to try it? Let's block some threads and go back to developer's comfort zone where SQL queries run one after 
another. What's more we will use ```Transactions``` to keep our data consistent. Sounds like going back to ```JEE```? Maybe. Fasten your seatbelts!

But first... story about potato (*you can skip it if you don't like potatoes...*).


### Potatoes and winter ###

According to [potatopro.com](http://www.potatopro.com) Poland is one of the world's biggest potatoes producer. What is more, winter is coming... Everyone in Poland knows,
that in order to survive winter you need to have your basement full of potatoes. 

When you buy potatoes you buy them in bags, potato bags - 10 kilos each. With potato bags comes huge risk. If inside the bag there is at least one rotten potato,
then soon or later all of them will be rotten. Winter will be long and you will die...

Basically, what you have to do is to open potato bag before the purchase and check whether potatoes are fresh or not. If one is rotten - then whole bag goes to trash.
To summarize:

> You cannot have potato bag without checking all of them.

> Caulo Poelho

I hope it all sounds like SQL transaction... First you store a ```bag```. Then you store ```potatoes``` one by one. If rotten potato is found, you withdraw
transaction and neither bag nor potatoes is stored in basement.

### Back to code ###

Whole example is available on [Github](https://github.com/qrman/vertx-guice-jooq-ultm). In this post I will show only the most interesting parts.

This is project structure:

{% highlight Bash %}

-- 
  -- conf
  -- scripts
  -- potato-db
  -- potato-logic
  pom.xml
  Makefile

{% endhighlight %}

```conf``` - properties for Maven and Vert.x

```scripts``` - linux scripts for Database creation (PostgresSQL)

```potato-db``` - all DB related files and code. More bellow.

```potato-logic``` - Verticle with all "Enterprise" logic for storing potatoes into basement :)

```pom.xml``` - parent pom

```Makefile``` - useful commands


### potato-db ###

Let's start from ```potato-db```.  There's a lot of stuff!

First of all, look inside ```pom.xml```. I used [Liquibase](http://www.liquibase.org/) for managing changes in Database structure.
If you are not familiar with Liquibase, you should at least know, that all Tables structures are defined inside xml files. 
Those files are invoked during Maven build process.
For example ```POTATO_BAG``` table is defined as follow:

{% highlight xml %}

<createTable schemaName="potato_schema"  tableName="potato_bag">
  <column name="id" type="bigint" autoIncrement="true">
    <constraints primaryKey="true" nullable="false"/>
  </column>
  <column name="origin" type="varchar(256)">
    <constraints nullable="false"/>
  </column>
</createTable>

{% endhighlight %}

Then [jOOQ](http://www.jooq.org/) comes in! We are still in ```pom.xml```. jOOQ generates Classes based on DB structure. It allows us 
to create type safe SQL queries (via awesome DSL) inside application.

All important Java stuff from ```potato-db``` is defined in ```Guice``` module so let's dive into ```DBModule```. 

{% highlight java linenos%}

public class DbModule extends AbstractModule {
    @Override
    protected void configure() {
        bind(DataSource.class).toProvider(DataSourceProvider.class).in(Singleton.class);
        
        bind(DSLContext.class).toProvider(JooqProvider.class).in(Singleton.class);
        bind(DSLContext.class).annotatedWith(TxJooq.class).toProvider(TxJooqProvider.class).in(Singleton.class);

        bind(ULTM.class).toProvider(ULTMProvider.class).in(Singleton.class);
        bind(TxManager.class).toProvider(TxManagerProvider.class).in(Singleton.class);
    }
}

{% endhighlight %}

```DSLContext``` is jOOQ "every operation on database builder". We need two of them, 
one transactional (annotated with ```@TxJooq```) and one non-transactional. 

Last two bindings give us ```Transactions```.
I used [ULTM](https://github.com/witoldsz/ultm) which stands for **Ultra Lightweight Transaction Manager** for JDBC. If you are looking 
for something simple for transactions management you should definitely try it. ```TxManger``` will be used for running code inside transaction.

That's all from ```potato-db```. Quite a lot, I guess.


### potato-logic ###

Combining Vert.x and Guice were described in other [post]({{ site.url }}/posts/2015/07/10/vertx3-guice/). So let's focus on the other parts of application.

We have ```PotatoBag``` and ```Potato``` classes, which looks as follow:

{% highlight java linenos%}

public class PotatoBag {
    private String origin;
    private List<Potato> items;
}

public class Potato {
    private int quality;
}

{% endhighlight %}

Potatoes may come from many regions of the world. If quality of a Potato is less than 100, it is treated as ```"soon will be rotten"```.
We don't want such potatoes in basement.

```PotatoVerticle``` can store a ```PotatoBag``` and return ```Potato``` list for chosen origin.

{% highlight java linenos%}

//Store PotatoBag
vertx.eventBus().consumer("potato-bag", (Message<String> bagWithPotatoMessage) -> {
    PotatoBag potatoBag = Json.decodeValue(bagWithPotatoMessage.body(), PotatoBag.class);
    log.log(Level.INFO, "Potato bag received {0}", potatoBag);

    vertx.executeBlocking(future -> {
        basementStore.store(potatoBag);
        future.complete();
    }, result -> {
        if (result.succeeded()) {
            bagWithPotatoMessage.reply("Potato bag stored in basement");
        }
        if (result.failed() && result.cause() instanceof RottenPotatoException) {
            bagWithPotatoMessage.reply("Bag with rotten potato cannot be stored");
        }
    });
});

{% endhighlight %}

There is some complexity here (If you are not familiar with Vert.x ```EventBus``` API), but the most important thing is that
we run ```basement.store(potatoBag)``` inside ```vertx.executeBlocking```. Vert.x for handling such operation uses extra thread pool,
so ```EventBus``` is not blocked by ```JDBC``` which runs somewhere under the hood.

It all looks similar for fetching potatoes from basement.

{% highlight java linenos%}

vertx.eventBus().consumer("potatoes-in-basement", (Message<String> countryMessage) -> {
   String potatoOriginCountry = countryMessage.body();
   log.log(Level.INFO, "Searching in basement for potatoes from {0}", potatoOriginCountry);
   
    vertx.executeBlocking(future -> {
       List<Potato> potatoes = basementFetch.fetchByOrigin(potatoOriginCountry);
       future.complete(potatoes);
    }, result -> {
       if (result.succeeded()) {
           countryMessage.reply(Json.encode(result.result()));
       }
       if (result.failed()) {
           countryMessage.reply(Json.encode(Collections.EMPTY_LIST));
       }
   });
);

{% endhighlight %}


The most relevant part of potato-logic module is implemented inside ```BasementStore``` class.

{% highlight java linenos%}
public class BasementStore {

    private final TxManager txManager;
    private final DSLContext txJooq;
    private final PotatoQualityChecker qualityChecker;

    @Inject
    public BasementStore(TxManager txManager, @TxJooq DSLContext txJooq, PotatoQualityChecker qualityChecker) {
        this.txManager = txManager;
        this.txJooq = txJooq;
        this.qualityChecker = qualityChecker;
    }

    public void store(PotatoBag potatoBag) {
        txManager.tx(() -> {
            PotatoBagRecord potatoBagRecord = txJooq.newRecord(POTATO_BAG);
            potatoBagRecord.setOrigin(potatoBag.getOrigin());
            potatoBagRecord.store();

            potatoBag.getItems().stream()
              .forEach((Potato potato) -> {
                  qualityChecker.check(potato);
                  storePotato(potato, potatoBagRecord);
              });
        });
    }

    private void storePotato(Potato potato, PotatoBagRecord potatoBagRecord) {
        PotatoRecord potatoRecord = txJooq.newRecord(POTATO);
        potatoRecord.setBag(potatoBagRecord.getId());
        potatoRecord.setQuality(potato.getQuality());
        potatoRecord.store();
    }
}
{% endhighlight %}

Here ```TxManager``` and ```@TxJooq``` are used. Everything which is inside 

{% highlight java linenos%}
txManager.tx(() -> {

//here

});

{% endhighlight %}

is executed inside transaction. All SQL changes will be rollback if something goes wrong inside transaction. And as long as 
there is ```PotatoQualityChecker```, ```RottenPotatoException``` may be thrown when one of ```Potato``` quality is insufficient.


### Summary ###

Basically, this is it. I hope this post shows, that you don't have  to resign form SQL and transactions while playing with Vert.x.
There are cool and easy-to-use tools for that.

Last but not least. It's all tested. But how I did it, it's a topic for another Blog post. Please left thoughts in comments and
check whole project on [Github](https://github.com/qrman/vertx-guice-jooq-ultm).

Remember: **write code and eat potatoes!**
