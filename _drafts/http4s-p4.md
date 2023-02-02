---
author: Bones, Dog and Megachu
date: 2023-01-24 08:48:00+00:00
title:  "F[F[S]] Kafka! - Case study: The digital transformation of Santa's logistical nightmare - Part 3 fs2-kafka"
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

Welcome to part three of the series:
[Part 1 - The digital transformation of Santa’s logistical nightmare - Part - Introduction](https://functional-feline-society.github.io/2022/12/16/santas-logistical-nightmare-pt1/)
[Part 2 - The digital transformation of Santa’s logistical nightmare - Part 2 - Cats Effects](https://functional-feline-society.github.io/2022/12/22/io-part-2/)
[Part 3 - F[F[S]] Kafka! - Case study: The digital transformation of Santa’s logistical nightmare - Part 3 fs2-kafka](https://functional-feline-society.github.io/2023/01/24/fs2-kafka/)

In this blog post, we’d like to go through how Santa’s workshop makes use of [http4s]().


## General things we want to say about http4s.


There are many options when it comes to Web servers in Scala, we are most familiar with http4s and also we think it works well with the rest of the Typelevel stack. Which means it works well with Cats Effect.

As with most web servers, you are provided with an API to deal with making HTTP requests, on the Client side, and serving those requests on the Server side. 
Http4s also has some options when it comes to different backends. Most people have been using Blaze, which is based on Netty for a very long time (think a few years). But the future is with Ember which is based on an FS2 wrapper around [NIO](https://docs.oracle.com/en/java/javase/15/core/java-nio.html) 

## Setting up the Server

Whilst the http4s API is not completely intuitive, I have to check the docs every time I want to set up a Server, the requirements of the API do read well once they are written. To configure our server we do the following:

/.. Insert server code
server <- EmberServerBuilder
 .default[IO]              // create instance
 .withHost(ipv4"0.0.0.0")  // setup 
 .withPort(port"8080")
 .withHttpApp(httpApp)
 .build




We first need to choose a backend, we chose Ember. Then we use the builder pattern provided by http4s to create a server. You will notice there is an `httpApp` parameter. 
If you inspect the type for `httpApp` you will find that it’s a Kleisli[IO, Request[IO], Response[IO]] - [Kleisli](https://typelevel.org/cats/datatypes/kleisli.html) is a case class wrapping Request[IO] => IO[Response[IO]]. Kleisli can be used to leverage a number of combinators and typeclasses provided by cats-core - e.g. SemigroupK for combining Routes.
If you want to know more about Kleisli go [here](INSERT A GOOD LINK), for practical purposes you can think of it as a way to mount the routes, deal with some errors, etc. 
It is also possible to combine routes(because kleisli) using the `<+>` combinator from `SemigroupK` between routes, which is really handy in more complex scenarios. There is an example of it in [the http4s docs](https://http4s.org/v0.23/docs/service.html#running-your-service)

Our httpApp is composed of a combined set of routes that dictate how Requests should be handled, as well as a fallback in the event no routes are matched `.orNotFound`.  As can be seen on lines 25:27 of SantasServer.scala.
Finally, `.build` constructs our `Resource[IO, Server]` object that encapsulates how to start and safely shutdown our web server.


Moving on to `SantasRoutes.scala` we can see an example of how to define a Route that matches a GET request. Line 11 has a lot to unpack - the pattern match here is saying that we want to match `GET` requests for resources that match `/list` with an additional two trailing segments that we wish to capture as path variables.

Lines 12 and 13 then say that we want to respond with a 200 OK containing whatever we find when we call `getConsignment` with the full name provided in path variables.
At this point you may be wondering how our `IO[ChristmasConsignment]` is converted into an actual HTTP response.

The http4s DSL expects an EntityEncoder to be available for whatever entity you provide to apply - in our case it’s expecting a EntityEncoder[IO, List[FullName]]. This is being derived from our Circe json Encoder using the CirceEntityCodec.circeEntityEncoder method. Our Circe codecs are being derived semi-automatically in `domain.scala` using macros from circe-generic. 
## Setting up the Client

We use the http4s [Client](https://http4s.org/v0.23/docs/client.html) for testing purposes, we will talk more about our approach to testing in a future post.

Setting up a client is not too different from the builder pattern used to create a server. 

val clientResource: Resource[IO, Client[IO]] = EmberClientBuilder.default[IO].build


In the line above we create the Client resource, we specify we are using IO for effects. When we use the Client, it will result in the creation of a connection pool. This would enable the reuse of connections for multiple requests. It is a common (good) practice to create one client and reuse it in your app (we haven’t done this for testing, probably because we are a bit lazy, after all we are 🐱🐱🐱 cats).


clientResource.use { client =>
 client.expect[List[FullName]](
   baseUri.withPath(
     Path.unsafeFromString(
       s"/house/${URLEncoder.encode(address.address, "UTF-8")}")))
}

When we are calling `expect` we are saying we expect the response to be decoded without errors to a `List[FullName]`  the parameter for it is a `Uri` in this case, we already have a baseUri in scope. This version of `expect` that takes a `Uri` as a parameter will make a `Get` request. There are alternative versions of `expect` that accept a `Request[IO]` which lets you choose the HTTP method.

Under the hood, we are using Circe to decode the incoming data. As explained in the previous section, circe is nicely integrated into http4s. 




## Cat tax

And finally the reason you are really here, to see pictures of us (Bones, Dog & Megachu)!

