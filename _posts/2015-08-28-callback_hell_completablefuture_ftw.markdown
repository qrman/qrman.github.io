---
layout: post
title:  "Callback Hell and CompletableFuture FTW"
date:   2015-08-28 23:00:00
comments: true
tags: java, CompletableFuture, Callback Hell
description: Callback Hell defeated with CompletableFuture

---

### Async is sexy ###

Everyone knows that asynchronous is sexy, fancy and jazzy. That could be the end of this post.

But true is, that writing async code is hard.
Understanding async code is even harder. There are many mature frameworks/libraries which uses promises, mostly in JavaScript world.
[AngularJS](https://angularjs.org/) services: $q and $http - are my favourites ones.

### Callback Hell in Vert.x world ###

[Vert.x](http://vertx.io) is also quite nice. But it's written in callback manner. Some time ago I wanted to use Redis inside 
Verticle. Nothing special. But picture this scenario:

1. Receive EventBus event
2. Put something to Redis
3. Put something other to Redis
4. Then read something from Redis
5. Finally return some value in EventBus event

It may look like hell.... Callback Hell. Just look at this example from JUnit test:

{% highlight java linenos%}
 public void will_register_player(TestContext context) {
    Async async = context.async();
    hammerThrowTournament.addTournamentPlayer(new Player(1000, "Anita Wlodarczyk"), (AsyncResult<String> player1Handler) -> {
        hammerThrowTournament.addTournamentPlayer(new Player(1001, "Zhang Wenxiu"), (AsyncResult<String> player2Handler) -> {
            hammerThrowTournament.addTournamentPlayer(new Player(1002, "Alexandra Tavernier"), (AsyncResult<String> player3Handler) -> {
                log.info("All players has been added");
                hammerThrowTournament.registerdPlayer((AsyncResult<JsonArray> resultHandler) -> {
                    log.info("Looking for results");
                    int registerdPlayers = resultHandler.result().size();
                    context.assertEquals(registerdPlayers, 3, "All Players were registered");
                    async.complete();
                });
            });
        });
    });
}
{% endhighlight %}

Such code is even hard to display in blog post [sic!]. 

What is happening here? **Poland is World best country in Hammer Throwing** (that also could be the end of the post :) ).
We want to register 3 players in Hammer Throw Tournament. After registering we are checking that everything is ok.

Code looks ugly. My ```HammerThrowTournament``` class in every method takes Handler as parameter which is passed into ```RedisClient``` and is executed
when Redis operation ends. To be sure that one operation comes after another we create classic example of Callback Hell. Is there anything we can do with such code?

### CompletableFuture FTW ###

Yes, we can!

Let's start with wrapping all ```HammerThrowTournament``` methods with ```CompletableFuture```. So instead of this:

{% highlight java linenos%}

public void addTournamentPlayer(Player p, Handler<AsyncResult<String>> handler) {
    Map<String, String> hmset = new HashMap<>();
    hmset.put("name", p.getName());
    redisClient.hmset("PLAYERS:" + p.getId(), hmset, handler);
}

{% endhighlight %}

we have this:

{% highlight java linenos%}

public CompletableFuture<Void> addTournamentPlayer(Player p) {
    CompletableFuture<Void> promise = new CompletableFuture<>();
    Map<String, String> hmset = new HashMap<>();
    hmset.put("name", p.getName());
    redisClient.hmset("PLAYERS:" + p.getId(), hmset, (AsyncResult<String> event) -> {
        promise.complete(null);
    });
    return promise;
}

{% endhighlight %}


Few lines of code more. But just take a look how my test is organized now.

{% highlight java linenos%}

@Test
public void will_register_player(TestContext context) {
    Async async = context.async();
    hammerThrowTournament.addPlayer(new Player(1000, "Anita Wlodarczyk"))
        .thenCompose(v -> hammerThrowTournament.addPlayer(new Player(1001, "Zhang Wenxiu")))
        .thenCompose(v -> hammerThrowTournament.addPlayer(new Player(1002, "Alexandra Tavernier")))
        .thenCompose(v -> hammerThrowTournament.registerdPlayer())
        .thenAccept(numberOfRegisteredPlayer -> {
            context.assertEquals(numberOfRegisteredPlayer, 3, "All Player were registerd");
            async.complete();
        });
}
{% endhighlight %}

How beautiful it is! How easy to read! Where's the magic?

Every ```addPlayer``` method returns ```CompletableFuture<Void>``` . With ```thenCompose``` 
method we can pass ```CompletableFuture``` result into another ```CompletableFuture``` and achieve so nice chaining.


I'm not an ```CompletableFuture``` expert. You can read more about it [here](http://www.nurkiewicz.com/2013/05/java-8-definitive-guide-to.html)
and if you know how to pronounce ```Anita Włodarczyk``` correctly you can watch this [presentation](https://www.youtube.com/watch?v=S7gCcgTWSPs).

Code is available here: [https://github.com/qrman/defeat-callback-hell](https://github.com/qrman/defeat-callback-hell). Read commits history to track
code transformation.

Good night and Good luck!