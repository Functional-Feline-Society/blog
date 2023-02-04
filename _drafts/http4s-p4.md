---
author: Bones, Dog and Megachu
date: 2023-01-24 08:48:00+00:00
title:  "http4s - Case study: The digital transformation of Santa's logistical nightmare - Part 4 http4s"
categories:
- scala
- scala-cats
- cats-effect
- kafka
- FS2-Kafka
- avro
- vulcan
---




Case study: The digital transformation of Santa’s logistical nightmare - Part 4

Welcome to part four of the series:
[Part 1 - The digital transformation of Santa’s logistical nightmare - Part - Introduction](https://functional-feline-society.github.io/2022/12/16/santas-logistical-nightmare-pt1/)
[Part 2 - The digital transformation of Santa’s logistical nightmare - Part 2 - Cats Effects](https://functional-feline-society.github.io/2022/12/22/io-part-2/)
[Part 3 - F[F[S]] Kafka! - Case study: The digital transformation of Santa’s logistical nightmare - Part 3 fs2-kafka](https://functional-feline-society.github.io/2023/01/24/fs2-kafka/)

In this blog post, we’d like to go through how Santa’s workshop makes use of [http4s](https://http4s.org).


## General things we want to say about http4s.

There are many options when it comes to Web servers in Scala, we are most familiar with http4s and it works well with the rest of the Typelevel stack. Which means it works well with Cats Effect.

As with most web servers, you are provided with an API to deal with making HTTP requests, the Client side. And serving those requests,the Server side. 

Http4s has options when it comes to different backends. Most people use Blaze, which is based on Netty. But the future is with Ember which is based on an FS2 wrapper around [NIO](https://docs.oracle.com/en/java/javase/15/core/java-nio.html). For the purposes of this Toy example we chose [Ember](https://mvnrepository.com/artifact/org.http4s/http4s-ember-server). 

## Setting up the Server

Whilst the http4s API is not completely intuitive (I have to check the docs every time, might be just me tho) the requirements of the API do read well once they are written. To configure our server we do the following:

### Server builder
{% highlight scala %}
server <- EmberServerBuilder
  .default[IO]
  .withHost(ipv4"0.0.0.0")
  .withPort(port"8080")
  .withHttpApp(httpApp)
  .build
{% endhighlight %}


We use the builder pattern provided by http4s to create a server. Here we specify the interface and port to bind to (`0.0.0.0` & `8080`) as well as the http application to expose.
Finally, `.build` constructs our `Resource[IO, Server]` object that encapsulates how to start and safely shutdown our web server.

### What's this httpApp?
Towards the end of the code snippet above you will notice there is an `httpApp` parameter being passed to `withHttpApp`. 
If you inspect the type for `httpApp` you will find that it’s a `Kleisli[IO, Request[IO], Response[IO]]`. 
[Kleisli](https://typelevel.org/cats/datatypes/kleisli.html) is a case class wrapping `Request[IO] => IO[Response[IO]]`. `Kleisli` can be used to leverage a number of combinators and typeclasses provided by cats-core - e.g. `SemigroupK` for combining Routes.

For practical purposes and for this instance, you can think of `Kleisli` as a way to mount the routes, deal with some errors, etc.
It is also possible to combine routes using the `<+>` combinator from `SemigroupK` between routes as seen in [the example in the http4s docs](https://http4s.org/v0.23/docs/service.html#running-your-service).

If you'd like to learn more about Kleisli check out the video [here](https://www.youtube.com/watch?v=qL6Viix3npA). 

### Routes
Our httpApp is composed of a combined set of routes that dictate how Requests should be handled, as well as a fallback in the event no routes are matched `.orNotFound` - as seen in the snippet below.

{% highlight scala %}
httpApp = Router(
  "/"         -> SantasRoutes.ledger(addressBookService, consignmentService),
  "/internal" -> StatusRoutes.route
).orNotFound
{% endhighlight %}

`Router` lets us combine routes under a specified prefix for each route. In this case we have all the routes relating to Santa's logistical problems , `/` and then we have some status routes in `/internal` meaning that we will find the status of the app in `<host:port>/internal/status`.

Lets look at an example of how to define a Route that matches a GET request.
In `SantasRoutes.scala` we can find the snippet below.

{% highlight scala %}
  def ledger(addressBookService: SantasAddressBookService, consignmentService: ConsignmentService): HttpRoutes[IO] =
    HttpRoutes.of[IO] {
      case GET -> Root / "list" / lastName / firstName =>
        Ok(
          consignmentService.getConsignment(FullName(firstName, lastName))
        )
{% endhighlight %}

There is a bit to unpack here - the pattern match is saying that we want to match `GET` requests for resources that match `/list` with an additional two trailing segments that we wish to capture as path variables.

The following lines then say that we want to respond with a 200 OK containing whatever we find when we call `getConsignment` with the full name provided in path variables.
At this point you may be wondering how our `IO[ChristmasConsignment]` is converted into an actual HTTP response.

The http4s DSL expects an EntityEncoder to be available for whatever entity you provide to apply - in our case it’s expecting a EntityEncoder[IO, List[FullName]]. This is being derived from our Circe json Encoder using the CirceEntityCodec.circeEntityEncoder method. Our Circe codecs are being derived semi-automatically in `domain.scala` using macros from circe-generic. 

## Setting up the Client

We use the http4s [Client](https://http4s.org/v0.23/docs/client.html) for testing purposes, we will talk more about our approach to testing in a future post.

Setting up a client is not too different from the builder pattern used to create a server. 

{% highlight scala %}
val clientResource: Resource[IO, Client[IO]] = EmberClientBuilder.default[IO].build
{% endhighlight %}

In the line above we create the Client resource, we specify we are using IO for effects. 
When we use the Client, it will result in the creation of a connection pool. This would enable the reuse of connections for multiple requests. It is a common (good) practice to create one client and reuse it in your app (we haven’t done this for testing, probably because we are a bit lazy, after all we are 🐱🐱🐱 cats).

{% highlight scala %}
clientResource.use { client =>
 client.expect[List[FullName]](
   baseUri.withPath(
     Path.unsafeFromString(
       s"/house/${URLEncoder.encode(address.address, "UTF-8")}")))
}
{% endhighlight %}

When calling `expect` we are saying we expect the response to be decoded without errors to a `List[FullName]. 
This version of `expect` that takes a `Uri` as a parameter will make a `Get` request. There are alternative versions of `expect` that accept a `Request[IO]` which lets you choose the HTTP method.

Under the hood, we are using Circe to decode the incoming data. As explained in the previous section, circe is nicely integrated into http4s. 


# Conclusion

We are hoping that this series of posts are helping you, understand not just how to do this but why we wrote the code in the way we did. After all, there are many ways to solve any problem, and good solutions have a context. 

At the time of writing this, Http4s is very close to `1.0` and after many years of using it, you can really feel the thought and care that has gone to it. Thanks to everyone that make that possible... so I guess all we can do is pay it with Cat Tax.

## Cat tax

And finally the reason you are really here, to see pictures of us (Bones, Dog & Megachu)!

<div>
  <div style="display: flex; flex-direction: row; flex-wrap: wrap;width: 100%;">
    <div style="display: flex; flex-direction: column; flex-basis: 100%; flex: 1; background-position: 99% 75%; background-size: cover; background-image: url(https://functional-feline-society.github.io/images/megachu-3.jpeg);">
    </div>
    <div style="display: flex; flex-direction: column; flex-basis: 100%; flex: 1;">
      <img src="https://raw.githubusercontent.com/K1nd/k1nd.github.io/gh-pages/assets/images/Bones.jpg" alt="Bones">
    </div>
    <div style="display: flex; flex-direction: column; flex-basis: 100%; flex: 1;">
      <img src="https://raw.githubusercontent.com/K1nd/k1nd.github.io/gh-pages/assets/images/Dog.jpg" alt="Dog">
    </div>
  </div>
  <div style="display: flex; flex-direction: row; flex-wrap: wrap;width: 100%;">
<div style="display: flex; flex-direction: column; flex-basis: 100%; flex: 1; text-align: center;">
      Megachu
    </div>
    <div style="display: flex; flex-direction: column; flex-basis: 100%; flex: 1; text-align: center;">
      Bones
    </div>
    <div style="display: flex; flex-direction: column; flex-basis: 100%; flex: 1; text-align: center;">
      Dog
    </div>
</div>
<div style="flex-direction: column; flex-basis: 100%; flex: 1; text-align: center; padding:5em">
<img src="https://media.tenor.com/seHiTkRjzgEAAAAd/silvestre-piol%C3%ADn.gif"  />
</div>
</div>

