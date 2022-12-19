
If you have never had any exposure to cats-effect and IO maybe the best place to start is somewhere like [the tutorial](https://typelevel.org/cats-effect/docs/concepts) which is pretty friendly and complete and also the [IO documentation](https://typelevel.org/cats-effect/docs/2.x/datatypes/io) it might feel a bit RTFM-ey and such, but these resources have many iterations on them and they have proven good, probably better than anything we can write here really. 

Assuming you at least got a primer on cats-effects now but getting your paws wet a little, this example might be a good place to start to see how it can be useful... how can it help bring all these side-effects together.

When we started writing this example we soon had to choose how to deal with side effects, one option was to go with Finally Tagless (I never know what is the right way to refer to this) or use IO directly. We thought that going for the concrete IO would be friendlier to people coming from other libraries or languages (if you are one such person reading this, let us know if we were correct or totally wrong) 
You will see IO as a type parameter to many traits. Some of them defined by libraries (like in the case of Env) and some are defined by us, type parameters work as usual and also they are sign of how we want to deal with all side effects.   


Now that you know what we were thinking (besides some nice Chicken treats (we're cats rememeber?)) let's look at some code in the example. It is probably a good idea to  start in the Main https://github.com/Functional-Feline-Society/santas-stream/blob/68c09fa9cb9b7c296e8cf00f822adfabcecfc3ea/src/main/scala/com/northpole/santas/Main.scala#L8

There are a few intersting things in this file, for a start this is not your vanilla scala App, it is an IOApp... a Simple one. (not sure how much to write about this?) 

Following to  `loadKafkaConfigFromEnv`we see that there is some interesting looking `Env[IO]`. Env is part of cats-effect 3 and it's tehre to  provide a simple way of reading the process's environment variables, pretty handy. Soon after we see taht we are mapping over two things! with `mapN` (should we explain how this works? mmmh?) if we were to leave it all there, we would wind up with an Option[KafkaConfig] but what we really really want is to have an IO type (because we are in a for comprehension) and also it would be really nice if the app would just hard fail if we can't read the these values (we should fail hard when we don't have this) so `liftTo` which basically does what we want and if it goes wrong, we are asking for it to respond with a `MissingConfig` error 

AS the loading of the config is now completed, we return to Main to find `SantasServer.resource(kafkaConf).useForever.void` what is this ??? well, in here we are calling `SantaServer` a type we dfined and a method called `resource` with a parameter `kafkaConf`, this method returns a `Resource[IO, Server]`, where Server refers to `http4s` web server. Resources were described in  the docs mentioned above, so if you haven't read that it might an idea to head that way... (some bridge) we want to use that resource for as long as the IOApp is up , this is why we call useForever, but also we are in a for comp so we need the IO  the problem is that Nothing is not the same as Unit and for that very reason we need to use `void` at the end, so the return type of that line is IO[Unit] which works out pretty well

😅 and we are just about finished with Main... 





one nitpick: I don't think you need Resource[IO, Stream[IO, Unit]] here - creating a Stream isn't side-effecting so you can just have Stream[IO, Unit] as the return type.