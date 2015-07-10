---
layout: post
title:  "Vert.x 3 and Guice - the easiest way"
date:   2015-08-10 23:00:00
comments: true
tags: vert.x, guice, java
---

[Guice](https://github.com/google/guice) is really easy to use. This post is not about explaining how it works.

[Vert.x](http://vertx.io) is also easy (unless you have more then 10 Verticles sending 5k/s messages to each other).

Simple technologies should play well together. So it is with Guice and Vert.x.

### WimbledonVerticle - better than MyVerticle ###

Soon Roger Federer and Novak Djokovic will play for [Wimbledon Championship](http://www.wimbledon.com). Let's use them in small application, 
which will handles HTTP GET request and returns name of tournament winner based on their Karma.

Below, WimbledonVerticle code snippet shows how Dependency magic happens. While extending AbstractVerticle we already have
an instance of Vertx class. Guice allows injecting dependencies into already constructed instance. And that's all (almost).

{% highlight java linenos%}

@Log
public class WimbledonVerticle extends AbstractVerticle {

    @Inject
    private TournamentWinnerEndpoint tournamentWinnerEndpoint;

    @Inject
    @Named("port")
    private String port;

    @Override
    public void start() throws Exception {
        //Binds all dependencies to already initialized vertx instance
        Guice.createInjector(new WimbledonModule(vertx)).injectMembers(this);

        Router router = Router.router(vertx);
        router.get("/wimbledon/winner").handler(tournamentWinnerEndpoint::handle);

        vertx.createHttpServer()
          .requestHandler(router::accept)
          .listen(Integer.valueOf(port));

        log.info("WimbledonVerticle starting");
    }
}
{% endhighlight %}

Thanks to WimbledonModule we can bind all necessary dependencies. In this example only EventBus instance and properties are binded.
Properties comes from config.json file where application port number and Wimbledon players names are defined.

{% highlight java linenos%}

public class WimbledonModule extends AbstractModule {

    private final Vertx vertx;
    private final Context context;

    public WimbledonModule(Vertx vertx) {
        this.vertx = vertx;
        this.context = vertx.getOrCreateContext();
    }

    @Override
    protected void configure() {
        bind(EventBus.class).toInstance(vertx.eventBus());
        Names.bindProperties(binder(), extractToProperties(context.config()));
    }

    private Properties extractToProperties(JsonObject config) {
        Properties properties = new Properties();
        config.getMap().keySet().stream().forEach((String key) -> {
            properties.setProperty(key, (String) config.getValue(key));
        });
        return properties;
    }
}
{% endhighlight %}


Having it all together we can implement TournamentWinnerEndpoint with injected all needed dependencies.

{% highlight java linenos%}

public class TournamentWinnerEndpoint implements Handler<RoutingContext> {

    private final String player1;
    private final String player2;
    private final EventBus eventBus;

    @Inject
    public TournamentWinnerEndpoint(EventBus eventBus, @Named("player1") String player1, @Named("player2") String player2) {
        this.eventBus = eventBus;
        this.player1 = player1;
        this.player2 = player2;
    }

    @Override
    public void handle(RoutingContext request) {
        Random tournamentKarma = new Random();
        boolean firstPlayerIsAWinner = tournamentKarma.nextBoolean();

        String winner = firstPlayerIsAWinner ? player1 : player2;
        eventBus.send(winner, "Congratulation Wimbledon Winner!");
        request.response().end(winner);
    }
}

{% endhighlight %}

And basically this is it. We won twice. Clean, nice looking code and configurable Wimbledon Champion lottery which are ready for all forthcoming Wimbledons.


Final usage:

{% highlight Bash %}
curl -w " will win Wimbledon in 2015\n" -X GET http://localhost:8080/wimbledon/winner
{% endhighlight %}

As a result:
{% highlight Bash %}
Roger Federer will win Wimbledon in 2015
{% endhighlight %}
or
{% highlight Bash %}
Novak Djokovic will win Wimbledon in 2015
{% endhighlight %}

All code is available on [Github](https://github.com/qrman/vertx-guice).

Thanks for reading!
BTW: I hope I will manage to write some Tour de France inspired post. Still two weeks until the end.

