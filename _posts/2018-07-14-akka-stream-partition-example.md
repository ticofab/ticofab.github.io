---
layout: post
title: Akka Streams Partition Example
excerpt: "An example usage of Akka Streams Partition Graph Stage"
modified: 2018-07-14
tags: [scala, akka, akka-streams, akka-stream]
comments: true
---

Working with [Akka Streams](https://doc.akka.io/docs/akka/current/stream/index.html) is nothing less than pure pleasure. I can't be thankful enough to the Akka team's folks for this.

This post is just a little example of something I needed a little time figuring out myself. Amongst all the graph operators, `Partition` is bizarrely not listed in the documentation at the moment of writing, even though I find it extremely useful. It lets you create some logic like that: 

![A filter logic](/images/2018-07-14-partitionpipeline.png)

The example that follows is a very simple pipeline:

![Even/odd pipeline](/images/2018-07-14-even-odd-flow.png)

{% highlight java %}
{% raw %}
val oddEvenFilter = GraphDSL.create() { implicit b =>
  // logic to dispatch odd and even numbers to different outlets
  b.add(Partition[Int](2, n => if (n % 2 == 0) 0 else 1))
}

// this flow uses the above filter
val filterFlow = Flow.fromGraph(GraphDSL.create() { implicit b =>
  import GraphDSL.Implicits._
  val input = b.add(oddEvenFilter)
  val sink1 = Sink.foreach[Int](n => println("even sink received " + n))
  input.out(0) ~> sink1
  FlowShape(input.in, input.out(1))
})

// connect to source and final sink
Source(1 to 10)
  .via(filterFlow)
  .map(_ * 2)
  .runForeach(n => println("final sink received " + n))
{% endraw %}
{% endhighlight %}

In this case the filter only had two outlets (even or odd numbers), but the graph stage lets you define your own custom logic with as many ports as needed. Very powerful!
