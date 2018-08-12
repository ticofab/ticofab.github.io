---
layout: post
title: Akka Http Websocket Example
excerpt: "An example usage of Websocket in Akka Http"
modified: 2018-08-10
tags: [scala, akka, akka-http, Websocket]
comments: true
---

[Akka Http](https://doc.akka.io/docs/akka-http/current/introduction.html#philosophy) is a module of Akka that provides a
full HTTP and Websocket server and client implementation, building on the power of [Akka Streams](https://doc.akka.io/docs/akka/2.5.14/stream/stream-introduction.html#motivation).
This means backpressure and resilience transparently out of the box. Great! 

I recently needed to implement a bi-directional Websocket channel where each connected client is handled by an actor. The
examples I could find were mostly about building a chat and therefore with a broader focus. Here is my simpler example.

* Each Websocket connection gets assigned to an new actor
* This actor will reply with `"Hello " + s + "!"` when receiving a String message
* This actor will pipe down to the client any Int value it receives.

As described in the [Server Side Websocket Support](https://doc.akka.io/docs/akka-http/current/server-side/Websocket-support.html)
page of Akka Http, a Websocket connection is modelled as a Flow that ingests messages and returns messages. 
The key issue to solve was that I needed a reference to independently push messages down to the connected client. The trick
that helped me came from [this post](https://bartekkalinka.github.io/2017/02/12/Akka-streams-source-run-it-publish-it-then-run-it-again.html)
by Bartek Kalinka. The idea is to pre-materialize a `Source.actorRef` which jots down messages to a publisher Sink.  

{% highlight java %}
{% raw %}
val (down, publisher) = Source
  .actorRef[String](1000, OverflowStrategy.fail)
  .toMat(Sink.asPublisher(fanout = false))(Keep.both)
  .run()
{% endraw %}
{% endhighlight %}

At this point, each message sent to the `down` actorRef will end up in the `publisher` Sink. We can then create a Source
out of this sink, and use that source as the output for the Websocket Flow required by Akka Http. 

{% highlight java %}
{% raw %}
Source.fromPublisher(publisher).map(TextMessage(_))
{% endraw %}
{% endhighlight %}

Every message sent to the `down` actorRef above will be published as a `TextMessage` from the source we just created.
The actor that brings this all together looks like this:
  
{% highlight java %}
{% raw %}
class ClientHandlerActor extends Actor {

  implicit val as = context.system
  implicit val am = ActorMaterializer()

  val (down, publisher) = Source
    .actorRef[String](1000, OverflowStrategy.fail)
    .toMat(Sink.asPublisher(fanout = false))(Keep.both)
    .run()

  override def receive = {
    case GetWebsocketFlow =>

      val flow = Flow.fromGraph(GraphDSL.create() { implicit b =>
      
        // only works with TextMessage. Extract the body and sends it to self
        val textMsgFlow = b.add(Flow[Message]
          .mapAsync(1) {
            case tm: TextMessage => tm.toStrict(3.seconds).map(_.text)
            case bm: BinaryMessage => 
            // consume the stream
            bm.dataStream.runWith(Sink.ignore)
            Future.failed(new Exception("yuck"))
          })

        val pubSrc = b.add(Source.fromPublisher(publisher).map(TextMessage(_)))

        textMsgFlow ~> Sink.foreach[String](self ! _)
        FlowShape(textMsgFlow.in, pubSrc.out)
      })

      sender ! flow

    // sends "Hello XXX!" down the websocket
    case s: String => down ! "Hello " + s + "!"

    // passes any int down the Websocket
    case n: Int => down ! n.toString
  }
}
{% endraw %}
{% endhighlight %}

As it is born, it materializes the actorRef and the publisher Sink. When asked to return the handling flow, it creates
the flow using the `GraphDSL` syntax and sends it back. In your route, it goes like

{% highlight java %}
{% raw %}
path("connect") {

  val handler = as.actorOf(Props[ClientHandlerActor])
  val futureFlow = (handler ? GetWebsocketFlow) (3.seconds).mapTo[Flow[Message, Message, _]]

  onComplete(futureFlow) {
    case Success(flow) => handleWebsocketMessages(flow)
    case Failure(err) => complete(err.toString)
  }
}
{% endraw %}
{% endhighlight %}

Now you can build your own custom logic and behaviour around the ClientHandlerActor.
Full code is available on a [GitHub repository](https://github.com/ticofab/akka-http-Websocket-example).
Note that this won't work in a distributed setting. For that we'll need a couple adjustments, in a coming post!

