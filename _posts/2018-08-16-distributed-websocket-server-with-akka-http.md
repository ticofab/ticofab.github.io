---
layout: post
title: Distributed Websocket server with Akka HTTP
excerpt: "Handle Websocket connection with remote actors."
modified: 2018-08-16
tags: [scala, akka, akka-http, Websocket]
comments: true
---

The goal of my [previous post](http://ticofab.io/akka-http-websocket-example/) was to handle each websocket connection
with an actor. This post goes on to achieve the same thing in a distributed setting.

As described in the [Server Side Websocket Support](https://doc.akka.io/docs/akka-http/current/server-side/Websocket-support.html)
page of Akka Http, a Websocket connection is modelled as a Flow that ingests messages and returns messages.  
In order to achieve a full bi-directional communication, where messages can independently be sent to the other end, the
handling actor needs a way to "enter" such flow without a request from the connected client. The trick (inspired by  
[this post](https://bartekkalinka.github.io/2017/02/12/Akka-streams-source-run-it-publish-it-then-run-it-again.html) 
by Bartek Kalinka) is to pre-materialize a `Source.actorRef` connected to a publisher Sink.  

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

So far, same as previous post. The missing step is, how to connect this source originating on a remote node? This is what we want to achieve:

![Handling Websocket clients with remote actors](/images/distributed-websocket-with-akka-http-diagram.png)

### StreamRef to the rescue

The glorious [Akka Streams](https://doc.akka.io/docs/akka/2.5.14/stream/stream-introduction.html#motivation) library has
everything we need. An experimental feature lets us materialize a local Source into a `StreamRef`. You can think of that
as the equivalent of `ActorRef` for Streams; to name one, they both accomplish location transparency. We can then pass
the reference to the server and create there the handling Flow.

In this example, as the server receives an incoming Websocket connection, it asks its supervisor to provide a materialized source
for streaming down messages:

{% highlight java %}
{% raw %}
type HandlingFlow = Flow[Message, Message, _]
val routes = path("connect") {
  info(s"client connected")
  val futureFlow = (supervisor ? ProvideConnectionHandler) (3.seconds).mapTo[HandlingFlow]
  onComplete(futureFlow) {
    case Success(flow) => handleWebSocketMessages(flow)
    case Failure(err) => complete(HttpResponse(StatusCodes.InternalServerError, entity = HttpEntity(err.getMessage)))
  }
}
{% endraw %}
{% endhighlight %}

The supervisor forwards the request to one of the registered handling nodes - here chosen randomly:

{% highlight java %}
{% raw %}
case pch@ProvideConnectionHandler =>
      // forward it to a random node, if any
      val chosenNode = nodes(Random.nextInt(nodes.size))
      (chosenNode ? pch) (3.seconds) .... // more details below
{% endraw %}
{% endhighlight %}
      
The handling actor, running on a remote node does a few things:

* Upon creation, registers with the listener node.
* Creates a source of Strings and materialises it as a remote source.
* Pushes a greeting down that source every time it receives a message.

{% highlight java %}
{% raw %}
class Handler extends Actor with LogSupport {

  implicit val as = context.system
  implicit val am = ActorMaterializer()

  // make sure that we register only once we are effectively up
  val cluster = Cluster(as)
  cluster registerOnMemberUp {
    cluster.state.leader.foreach(leaderAddress =>
      as.actorSelection(RootActorPath(leaderAddress) / "user" / "supervisor") ! RegisterNode)
  }

  // every message this actor sends to the down actorRef will end up in the publisher sink...
  val (down, publisher) = Source
    .actorRef[String](1000, OverflowStrategy.fail)
    .toMat(Sink.asPublisher(fanout = false))(Keep.both)
    .run()

  // ... and every message in the publisher sink will be published by this Source. The final effect is
  // that messages sent to the down actorRef will be published to this source.
  val streamRef = Source.fromPublisher(publisher).runWith(StreamRefs.sourceRef())

  override def receive: Receive = {

    case ProvideConnectionHandler =>
      // send the streamRef reference back
      streamRef.map(ConnectionHandler(self, _)).pipeTo(sender)

    case s: String =>
      // in this example, all strings coming in are from the websocket client. Reply with a salute.
      down ! "Hello " + s + "!"
  }

}
{% endraw %}
{% endhighlight %}

Once the SourceRef has come back, the supervisor can create the needed flow:

{% highlight java %}
{% raw %}
...
case pch@ProvideConnectionHandler =>
  // forward it to a random node, if any
  val chosenNode = nodes(Random.nextInt(nodes.size))
  (chosenNode ? pch) (3.seconds)
    .mapTo[ConnectionHandler]
    .map { case ConnectionHandler(handlingActor, sourceRef) =>

      // create a handling flow to send back
      Flow.fromGraph(GraphDSL.create() { implicit b =>

        val textMsgFlow = b.add(Flow[Message]
          .mapAsync(1) {
            case tm: TextMessage => tm.toStrict(3.seconds).map(_.text)
            case bm: BinaryMessage =>
              bm.dataStream.runWith(Sink.ignore)
              Future.failed(new Exception("yuck"))
          })

        // map strings coming from this source to TextMessage
        val pubSrc = b.add(sourceRef.map(TextMessage(_)))

        // forward each message to the handling actor
        textMsgFlow ~> Sink.foreach[String](handlingActor ! _)

        FlowShape(textMsgFlow.in, pubSrc.out)
      })
    }
    .pipeTo(sender)
{% endraw %}
{% endhighlight %}

Full code is available on a [GitHub repository](https://github.com/ticofab/akka-http-distributed-websockets). Note that
this code is not meant for production - the `StreamRef` feature itself is marked as 'may change' at the moment of writing:
thread carefully! (Pun intended.)  

If you have any suggestion, please don't hesitate to leave a comment.
