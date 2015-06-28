---
layout: post
title:  "Vert.x 3 - faster, better, stronger"
date:   2015-06-28 23:00:00
comments: true
tags: vert.x, java, performance, json
---

### Vert.x 3 released ###

New version of [Vert.x](http://vertx.io)  has just been released. There are many new features and upgrades compared to Vert.x 2.
You can read about most of changes in one of the [threads](https://groups.google.com/forum/#!msg/vertx/_y_VqFQOVhs/r8zce-zzds0J) from official Vert.x mailing list.

In [Espeo](http://espeo.pl) we are using Vert.x in two projects. One of them (which I'm involved in) is at very early stage. That's why we will try to upgrade 
Vert.x version as soon as possible. In order to be 100% sure I have created simple benchmark to verify if new Vert.x comes with better performance.

### Benchmark preparation ###

In most popular scenario our application waits for HTTP post request with small (4,0 kB) JSON payload. 
Then we transform it to Java objects using [Json](http://vertx.io/docs/apidocs/io/vertx/core/json/Json.html) class in order to decode/encode operations.
At the end we send back some other JSON data.

JSON data looks as below (sample.json):

{% highlight json linenos%}
{
  "guestListName": "My Birthday Guest List",
  "eventName": "Birthday Party",
  "venue": {
    "name": "Great Place For Party",
    "address": "Baker Street 221B, London"
  },
  "guests": [
    {
      "firstName": "Sherlock",
      "lastName": "Holmes",
      "profession": "Private Detective",
      "age": "44"
    },
    {
      "firstName": "John",
      "lastName": "Watson",
      "profession": "Doctor of Medicine",
      "age": "50"
    }...
  ]
}

{% endhighlight %}

Vert.x 2 Verticle looks as below:

{% highlight java linenos%}

public class HttpVerticle extends Verticle {

  @Override
  public void start() {
    HttpServer httpServer = vertx.createHttpServer();
    httpServer.requestHandler(req -> {
      req.bodyHandler(buffer -> {
        GuestList guestList = Json.decodeValue(buffer.toString(), GuestList.class);
        req.response().end(Json.encode(guestList));
      });
    }).listen(8080);
  }
}
{% endhighlight %}

Vert.x 3 Verticle looks very similar:

{% highlight java linenos%}

public class HttpVerticle extends AbstractVerticle {

  @Override
  public void start() {
    vertx.createHttpServer()
      .requestHandler(req -> {
        req.bodyHandler(buffer -> {
          GuestList guestList = Json.decodeValue(buffer.toString(), GuestList.class);
          req.response().end(Json.encode(guestList));
        });
      }).listen(8081);
  }
}
{% endhighlight %}


There are also three classes: GuestList, Venue, Guest that map JSON data structure:

{% highlight java linenos%}

public class GuestList {

  private String guestListName;
  private String eventName;
  private Venue venue;
  private List<Guest> guests;
}

public class Venue {

  private String name;
  private String address;
}

public class Guest {

  private String firstName;
  private String lastName;
  private String profession;
  private int age;
}

{% endhighlight %}

As you can see, each Verticle waits for HTTP request, decodes request body (JSON) to Java POJOs, then encodes it once again to JSON
and sends back.

As a benchmark I used [ApacheBench](http://httpd.apache.org/docs/2.4/programs/ab.html), very easy tool for measuring HTTP server performance.

Test scenario will go as follow:

+   50 000 requests
+   100 concurrent requests


{% highlight Bash %}
#Vert.x 2:
ab -l -t 10 -c 100 -T application/json -p sample.json http://localhost:8080/test
#Vert.x 3:
ab -l -t 10 -c 100 -T application/json -p sample.json http://localhost:8081/test
{% endhighlight %}


### Benchmark results - Summary ###

After few rounds of warmup I've got results.

First Vert.x 2 result:

{% highlight Bash %}

Time taken for tests:   5.426 seconds
Complete requests:      50000
Failed requests:        0
Total transferred:      146300000 bytes
Total body sent:        221800000
HTML transferred:       144250000 bytes
Requests per second:    9214.46 [#/sec] (mean)
Time per request:       10.853 [ms] (mean)
Time per request:       0.109 [ms] (mean, across all concurrent requests)

Percentage of the requests served within a certain time (ms)
  50%     10
  66%     11
  75%     11
  80%     11
  90%     12
  95%     13
  98%     14
  99%     16
 100%     24 (longest request)

{% endhighlight %}

And Vert.x 3 result:

{% highlight Bash %}

Time taken for tests:   4.765 seconds
Complete requests:      50000
Failed requests:        0
Total transferred:      146300000 bytes
Total body sent:        221800000
HTML transferred:       144250000 bytes
Requests per second:    10493.76 [#/sec] (mean)
Time per request:       9.529 [ms] (mean)
Time per request:       0.095 [ms] (mean, across all concurrent requests)

Percentage of the requests served within a certain time (ms)
  50%      9
  66%      9
  75%     10
  80%     10
  90%     10
  95%     11
  98%     11
  99%     12
 100%     14 (longest request)

{% endhighlight %}

Benchmark results show that for my test scenario Vert.x 3 can handle 13.9% more requests per second than Vert.x 2.
Performance increased from 9214.46 to 10493.76 requests per second.

It's enough to convince me to migrate to Vert.x 3 next week.

Thanks for reading!

